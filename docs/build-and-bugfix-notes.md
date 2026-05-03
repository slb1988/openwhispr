# Build & Bugfix Notes

## 打包注意事项（macOS arm64）

### 正常打包流程

```bash
npm run pack
```

`pack` = `prepack`（编译 Swift 原生二进制 + 下载各平台 bin）+ `build:renderer`（Vite）+ `electron-builder --dir`（不签名）。

### 跳过 prepack（bin 已存在时）

当 GitHub API 限流（HTTP 403/401）或 token 失效时，如果 `resources/bin/` 下的二进制已存在，可直接跳过 prepack：

```bash
npm run build:renderer && CSC_IDENTITY_AUTO_DISCOVERY=false node_modules/.bin/electron-builder --dir
```

### GitHub Token

下载二进制时需要 GitHub Token 以提升 API 限流上限（60次/h → 5000次/h）：

```bash
GITHUB_TOKEN=ghp_xxx npm run pack
```

Token 失效（HTTP 401）时改用上面的跳过方式。

### 注意

- **Node 版本必须是 24**，否则 `package-lock.json` 会与 CI 不兼容。用 `nvm exec 24 npm install`。
- 打出的包在 `dist/mac-arm64/OpenWhispr.app`，测试前先 `pkill -f OpenWhispr` 关掉旧实例。

---

## Bug Fix：Language Models 标签页打开崩溃（React error #185）

**症状**：打开设置 → Language Models，页面白屏，ErrorBoundary 显示 "Minified React error #185"（Maximum update depth exceeded，无限渲染循环）。

### 根本原因

`selectResolvedLLMConfig` 每次调用都返回一个**新对象**：

```typescript
// settingsStore.ts
export const selectResolvedLLMConfig = (state, scope): ResolvedLLMConfig => {
  return { scope, mode, provider, model, ... }; // 每次 new object
};
```

在 `InferenceConfigEditor` 中直接用作 Zustand selector：

```typescript
// 错误写法 — Zustand 用引用相等比较，每次都触发重渲染
const config = useSettingsStore((s) => selectResolvedLLMConfig(s, scope));
```

Zustand 对 selector 返回值做引用相等检查。因为每次都返回新对象，**任何无关 store 更新都会触发这个组件重渲染**，导致 `config.provider`、`config.cloudBaseUrl` 等 props 对子组件来说每次都"变了"，进而触发子组件 `useEffect` → 子组件 `setState` → 父组件重渲染 → 无限循环。

### 修复方法

所有使用 `selectResolvedLLMConfig`（返回对象）的组件都必须用 `useShallow` 做浅比较，只在字段值实际变化时才重渲染：

```typescript
import { useShallow } from "zustand/react/shallow";

// 修复后（InferenceConfigEditor.tsx、DictationAgentSettings.tsx）
const config = useSettingsStore(useShallow((s) => selectResolvedLLMConfig(s, scope)));
```

**受影响的文件（需同步修复）：**
- `src/components/settings/InferenceConfigEditor.tsx`
- `src/components/settings/DictationAgentSettings.tsx`（agentConfig + cleanupConfig 两处）

**规则**：凡是 selector 返回对象（而非 string/boolean/number），必须包 `useShallow`。返回原始值的 selector（如 `selectIsCloudCleanupMode` 返回 boolean）不需要。
```

### 相关修复（同一次调试中发现）

**1. `InferenceConfigEditor` 中 `setField(...)` 内联调用不稳定**

`setField` 是一个闭包工厂，每次调用都返回新函数。在 JSX 中内联调用会在每次渲染时产生新的函数引用，传给子组件后导致子组件 props 看起来每次都变：

```tsx
// 错误 — 每次渲染 setField("remoteUrl") 都是新函数
<OpenAICompatiblePanel setBaseUrl={setField("remoteUrl")} ... />

// 修复 — 用 useCallback 保证稳定引用
const setRemoteUrl = useCallback((v: string) => setResolvedLLMConfig(scope, { remoteUrl: v }), [scope]);
<OpenAICompatiblePanel setBaseUrl={setRemoteUrl} ... />
```

需要为每个字段单独创建稳定的 `useCallback`：`setMode`、`setProvider`、`setModel`、`setRemoteUrl`、`setCloudBaseUrl`、`setCustomApiKey`、`setDisableThinking`。

**2. `OpenAICompatiblePanel` 中 `loadRemoteModels` 依赖了 `model`/`setModel`**

`loadRemoteModels` 的 `useCallback` deps 包含 `model` 和 `setModel`，但内部会调用 `setModel("")`，导致：`setModel("")` → `model` 变化 → `loadRemoteModels` 重建 → effect 重触发 → 循环。

```typescript
// 修复 — 用 ref 读取 model/setModel，不放入 deps
const modelRef = useRef(model);
const setModelRef = useRef(setModel);
useEffect(() => { modelRef.current = model; }, [model]);
useEffect(() => { setModelRef.current = setModel; }, [setModel]);

// useCallback deps 中移除 model 和 setModel
}, [baseUrl, apiKey, t]); // 不含 model/setModel
```

### 调试方式

`ErrorBoundary` 在生产包中只显示 minified 错误。增强 `componentDidCatch` 在 UI 中展示完整 stack + component stack：

```typescript
componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
  const detail = `${error.name}: ${error.message}\n\nStack:\n${error.stack}\n\nComponent Stack:\n${errorInfo.componentStack}`;
  this.setState({ errorDetail: detail });
}
```

Component Stack 中可以看到崩溃链路上的组件名，对照源码定位循环所在位置。

---

## 问题排查：macOS 辅助功能权限授权后仍提示"Failed to paste text"

**症状**：粘贴时报错 `Accessibility permissions required for automatic pasting. Text has been copied to clipboard - please paste manually with Cmd+V.`，即使已在系统设置中授权辅助功能权限。

### 根本原因

`clipboard.js` 中 `checkAccessibilityPermissions` 的实现：

```js
// clipboard.js:1634
const allowed = systemPreferences.isTrustedAccessibilityClient(false);
```

参数 `false` 表示只查询当前状态（不主动弹出授权提示）。这本身没问题，但以下场景会导致"已授权仍失败"：

1. **权限结果被缓存 5 秒（TTL）**：`ACCESSIBILITY_CHECK_TTL_MS = 5000`（`clipboard.js:13`）。授权后立即重试，缓存里还是 `false`。
2. **"僵尸权限"（开发环境最常见）**：重新构建 app 后，macOS TCC 数据库中存有旧的同名条目（指向旧路径的 Electron/OpenWhispr），导致即使新添加了权限，旧权限"挡住"了 `AXIsProcessTrusted()` 的判断。
3. **需要重启应用**：macOS 的 `AXIsProcessTrusted()` 在权限变更后需要重启进程才能生效（macOS 系统限制）。

### 解决步骤

**方法一：清理僵尸权限（开发环境首选）**

1. 打开 **系统设置 → 隐私与安全性 → 辅助功能**
2. 找到所有 `OpenWhispr`、`Electron` 相关条目，点 **`-` 全部删除**
3. 点 **`+`** 重新添加新的 OpenWhispr（`dist/mac-arm64/OpenWhispr.app` 或开发时的 Electron）
4. 确保复选框已勾选
5. **完全退出并重启** OpenWhispr

**方法二：通过终端重置 TCC 数据库**

```bash
tccutil reset Accessibility
```

然后重启 OpenWhispr，系统会重新弹出授权提示。

**方法三：排查是否为缓存问题**

授权后**等待 5 秒以上**再尝试粘贴。如果成功，说明是缓存命中了旧的 `false` 值。缓存逻辑在 `clipboard.js:1629`。

### 注意

- 开发环境（`npm run dev`）每次重新构建后 Electron 可执行路径可能变化，导致 TCC 数据库中的旧条目失效 — 必须按方法一手动清理。
- 授权的是 Electron 可执行文件，不是 app bundle，因此路径敏感。

---

## Bug Fix：Cloud Provider 返回 `<think>` 标签未被过滤

**症状**：使用 DeepSeek、Qwen 等 OpenAI 兼容云端 Provider 时，模型返回的 `<think>...</think>` 推理内容会原样出现在听写结果中。

### 修复

**非流式路径**（`ReasoningService.processText`）在 return 前加过滤：

```typescript
return result.replace(/<think>[\s\S]*?<\/think>/gi, "").trim();
```

**流式路径**（`ReasoningService.processTextStreaming`）的过滤条件从仅限 local/LAN 扩展到所有 provider：

```typescript
// 修复前：只对 local/LAN 过滤
const stripThinking = (isLocalProvider || isLanCleanup) && config.disableThinking !== false;

// 修复后：对所有 provider 过滤
const stripThinking = config.disableThinking !== false;
```
