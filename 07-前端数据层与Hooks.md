# 07 - 前端数据层与 Hooks:API 客户端、Context 与自定义 Hook 详解

> 本篇深入 `apps/frontend/lib` 与 `apps/frontend/hooks` 两个目录,系统讲解 API 客户端封装、Context 状态管理、三个核心自定义 Hook,并辅助介绍类型、常量与 i18n 模块。所有引用均给出 `文件路径:行号` 便于回查。

---

## 一、API 客户端:统一 fetch 封装

### 1.1 `lib/api/client.ts` 的设计要点

`apps/frontend/lib/api/client.ts:7-33` 定义两个运行时常量:

```ts
const DEFAULT_PUBLIC_API_URL = '/';
const INTERNAL_API_ORIGIN = 'http://127.0.0.1:8000';

export const API_URL = normalizeApiUrl(process.env.NEXT_PUBLIC_API_URL ?? DEFAULT_PUBLIC_API_URL);
export const API_BASE = resolveRuntimeApiBase(toApiBase(API_URL));
```

`resolveRuntimeApiBase`(`client.ts:25-30`)是核心 trick:当运行在浏览器端时,`API_BASE` 保持相对路径(`/api/v1`),由 `next.config.ts:21-42` 的 rewrites 转发;运行在服务端(Server Component / `print/*`)时,把相对路径改为 `http://127.0.0.1:8000/api/v1`,绕过 Next.js 代理直接打后端。这就是 print 页面不依赖浏览器 cookie 也能拉取数据的机制。

`apps/frontend/lib/api/client.ts:43-74` 的 `apiFetch` 是统一的 fetch wrapper:

```ts
export async function apiFetch(endpoint, options?, timeoutMs?): Promise<Response> {
  // 1. 规范化 URL(补前导斜杠,识别绝对 URL 与 /api 路径)
  // 2. 默认 timeout = 240_000ms(与后端 resumes.py wait_for 保持一致)
  // 3. 使用 AbortController 取消超时请求
  // 4. AbortError 转译为 'Request timed out. Please try again...'
}
```

`client.ts:79-118` 在此基础上封装了 `apiPost` / `apiPatch` / `apiPut` / `apiDelete`,这些 helper **不**自动 JSON 化返回值,调用方需自行 `await res.json()` —— 这是项目刻意保留的"低级但灵活"的 API,例如 `lib/api/resume.ts:122-135` 在 `improveResume` 中用 `response.text()` 自己做 JSON 解析,避免空响应导致 `res.json()` 抛错。

### 1.2 业务 API 模块

| 文件 | 范围 | 关键函数 |
|------|------|----------|
| `lib/api/resume.ts` | 简历与岗位 CRUD、AI 改善、PDF 下载 | `uploadJobDescriptions`、`previewImproveResume`、`confirmImproveResume`、`fetchResume`、`updateResume`、`downloadResumePdf`、`deleteResume`、`renameResume`、`retryProcessing`、`fetchJobDescription` |
| `lib/api/config.ts` | LLM / 功能 / 提示词 / 语言配置 | `fetchLlmConfig`、`updateLlmConfig`、`testLlmConnection`、`fetchSystemStatus`、`PROVIDER_INFO`、`fetchFeaturePrompts`、`updateFeaturePrompts`(422 抛 `FeaturePromptsError`,`config.ts:310-318`)、`clearAllApiKeys`、`resetDatabase` |
| `lib/api/enrichment.ts` | AI 增强 + AI 重新生成 | `analyzeResume`、`generateEnhancements`、`applyEnhancements`、`regenerateItems`、`applyRegeneratedItems` |
| `lib/api/index.ts` | barrel re-export | 注意只 re-export 公共部分,部分函数仍需直接 import 子模块 |

`lib/api/resume.ts:111-135` 的 `postImprove` 是个特殊 wrapper:它在 `response.text()` 拿到原文后再 `JSON.parse`,这样既能拿到非 2xx 的服务端详细错误,也能容错处理空响应。`lib/api/resume.ts:218-252` 的 `getResumePdfUrl` 把 `TemplateSettings` 序列化为 URL query string,后端 Playwright 读取后渲染 PDF。

### 1.3 类型安全与错误处理

每个业务模块都在文件顶部定义与后端 Pydantic 一一对应的 interface(例如 `lib/api/resume.ts:8-50` 的 `ProcessedResume`、`lib/api/config.ts:42-48` 的 `SystemStatus`)。错误路径采用一致的 `data.detail || 'Fallback (status X).'` 模式,如 `lib/api/config.ts:91-93`:

```ts
if (!res.ok) {
  const data = await res.json().catch(() => ({}));
  throw new Error(data.detail || `Failed to update LLM config (status ${res.status}).`);
}
```

`lib/api/config.ts:328-369` 的 `updateFeaturePrompts` 是更精细的例子:它在 422 + `code: 'missing_placeholders'` 时抛 `FeaturePromptsError`,让 UI 能精确定位到哪个字段缺了哪个占位符。

---

## 二、Context 提供者

### 2.1 `StatusCacheProvider`

`apps/frontend/lib/context/status-cache.tsx:39-216` 是项目里最重的 Context:

- 缓存 `/status` 响应(`SystemStatus`),首屏自动 fetch。
- 启动一个 30 min 周期的 `setInterval` 重新拉取 LLM 健康(`status-cache.tsx:189-199`),不打扰数据库统计。
- 暴露 `incrementResumes` / `decrementResumes` / `incrementJobs` / `incrementImprovements` / `setHasMasterResume`(`status-cache.tsx:97-176`)用于乐观更新 —— 例子见 `app/(default)/dashboard/page.tsx:178-186` 的 `handleUploadComplete`。
- 提供 `useIsStatusStale`(`status-cache.tsx:229-251`)辅助判断数据陈旧度。

所有更新都走 immutable spread(`status-cache.tsx:99-110`),从不直接修改 `prev.status`,符合函数式风格。

### 2.2 `LanguageProvider`

`apps/frontend/lib/context/language-context.tsx:26-113` 维护两个独立状态:

| 字段 | 来源 | 持久化 |
|------|------|--------|
| `uiLanguage` | localStorage(`resume_matcher_ui_language`) | 纯客户端 |
| `contentLanguage` | localStorage → backend `/config/language` | 同步到后端 |

`language-context.tsx:64-87` 的 `setContentLanguage` 采用"乐观更新 + 失败回滚"模式:先更新本地 state,再 PUT 后端,失败时回滚。`setUiLanguage`(`language-context.tsx:89-96`)是纯客户端 setter,只写 localStorage,无网络往返。

### 2.3 `ResumePreviewProvider`

`apps/frontend/components/common/resume_previewer_context.tsx`(由 `app/(default)/layout.tsx:1` 引入)承担 `/tailor` → `/builder` 的数据接力:`tailwind/page.tsx:154` 在 confirm 之后 `setImprovedData(confirmed)`,builder 通过 `useResumePreview()` 读取并填充表单。详见 `apps/frontend/app/(default)/tailor/page.tsx:69-162`。

---

## 三、三个核心自定义 Hook

### 3.1 `useEnrichmentWizard` —— 多步向导状态机

`apps/frontend/hooks/use-enrichment-wizard.ts` 用 `useReducer` 管理一个 9 步向导:`idle → analyzing → questions → generating → preview → applying → complete → error | no-improvements`。

#### Reducer 设计

`use-enrichment-wizard.ts:75-165` 的 `wizardReducer` 接收 13 种 action,典型模式:

```ts
case 'SET_ANSWER': {
  return {
    ...state,
    answers: { ...state.answers, [action.questionId]: action.answer },
  };
}
```

所有 case 都返回新对象,符合不可变约束。

#### 对外 API

`use-enrichment-wizard.ts:302-321` 暴露的字段:

| 类别 | 字段 |
|------|------|
| State | `state` (整棵 WizardState) |
| 动作 | `startAnalysis`、`setAnswer`、`nextQuestion`、`prevQuestion`、`goToQuestion`、`generateEnhancements`、`applyChanges`、`reset`、`retry` |
| 派生 | `currentQuestion`、`currentItem`、`isLastQuestion`、`isFirstQuestion`、`answeredCount`、`totalQuestions` |

#### 使用示例

```tsx
import { useEnrichmentWizard } from '@/hooks/use-enrichment-wizard';

function EnrichmentModal({ resumeId }: { resumeId: string }) {
  const wizard = useEnrichmentWizard(resumeId);

  useEffect(() => {
    if (wizard.state.step === 'idle') wizard.startAnalysis();
  }, [wizard.state.step, wizard]);

  if (wizard.state.step === 'questions') {
    return (
      <QuestionStep
        question={wizard.currentQuestion!}
        value={wizard.state.answers[wizard.currentQuestion!.question_id] ?? ''}
        onChange={(v) => wizard.setAnswer(wizard.currentQuestion!.question_id, v)}
        onNext={wizard.nextQuestion}
        onPrev={wizard.prevQuestion}
        isFirst={wizard.isFirstQuestion}
        isLast={wizard.isLastQuestion}
      />
    );
  }
  if (wizard.state.step === 'preview') {
    return <PreviewStep items={wizard.state.preview} onApply={wizard.applyChanges} />;
  }
  // ... 其他 step
}
```

`wizard.retry`(`use-enrichment-wizard.ts:267-284`)是个非常贴心的设计:它根据当前 state 决定回退到哪一步 —— 已有 preview 则停在 preview,只有 answers 则回到 questions,否则从头开始。

### 3.2 `useFileUpload` —— 拖拽 + 异步上传

`apps/frontend/hooks/use-file-upload.ts:76-629` 是一个完全自包含的文件上传 hook,处理:

| 能力 | 位置 |
|------|------|
| 拖拽事件(dragenter/dragover/drop) | `use-file-upload.ts:522-578` |
| 点击触发 | `openFileDialog`(`use-file-upload.ts:590-596`) |
| `getInputProps` 注入 `<input type="file">` | `use-file-upload.ts:598-612` |
| 类型/大小校验 | `validateFile`(`use-file-upload.ts:125-158`) |
| 唯一 ID | `generateUniqueId`(`use-file-upload.ts:167-169`) |
| 重复检测 | `use-file-upload.ts:367-378` |
| 图片预览 `URL.createObjectURL` | `use-file-upload.ts:160-165` |
| 上传过程(fetch + FormData) | `uploadFileInternal`(`use-file-upload.ts:171-320`) |
| 并发计数 | `inFlightUploadsRef`(`use-file-upload.ts:105`) |
| 卸载时清理 | `clearFiles`(`use-file-upload.ts:501-516`) |

返回值是 `[state, actions]` tuple,方便解构。`isUploadingGlobal` 由引用计数维护,只有当所有文件都上传完毕才变 false,避免界面闪烁。

#### 使用示例

```tsx
import { useFileUpload, formatBytes } from '@/hooks/use-file-upload';
import { getUploadUrl } from '@/lib/api';

function ResumeUploader({ onUploaded }: { onUploaded: (resumeId: string) => void }) {
  const [{ files, isDragging, errors, isUploadingGlobal }, { getInputProps, openFileDialog, removeFile }] =
    useFileUpload({
      maxSize: 10 * 1024 * 1024,           // 10MB
      accept: 'application/pdf',            // 仅 PDF
      multiple: false,
      uploadUrl: getUploadUrl(),
      onUploadSuccess: (file, response) => {
        if (typeof response.resume_id === 'string') onUploaded(response.resume_id);
      },
    });

  return (
    <div onDragEnter={...} onDragOver={...} onDrop={...}>
      <input {...getInputProps()} />
      <button onClick={openFileDialog} disabled={isUploadingGlobal}>
        {isUploadingGlobal ? 'Uploading...' : 'Select PDF'}
      </button>
      {files.map((f) => (
        <div key={f.id}>
          {f.file.name} — {formatBytes(f.file.size)}
          <button onClick={() => removeFile(f.id)}>×</button>
        </div>
      ))}
      {errors.map((e, i) => <p key={i} className="text-red-600">{e}</p>)}
    </div>
  );
}
```

注意 `use-file-upload.ts:309` 会过滤与该文件同名的通用错误,避免重复提示。

### 3.3 `useRegenerateWizard` —— 章节重写向导

`apps/frontend/hooks/use-regenerate-wizard.ts:59-194` 是 AI "regenerate" 流程的状态机,与 enrichment 类似但更轻:它直接用多个 `useState` 而非 reducer,因为状态字段数量少且彼此独立。

`use-regenerate-wizard.ts:95-125` 的 `generate`:

1. 校验至少选中一项(`use-regenerate-wizard.ts:96-99`)。
2. 构造 `RegenerateRequest`(含 `output_language`,`use-regenerate-wizard.ts:106-111`)。
3. 调用 `regenerateItemsApi`(`lib/api/enrichment.ts:151-160`)。
4. 成功后切到 `previewing` step,失败时回到 `instructing` 让用户修改 prompt。

`acceptChanges`(`use-regenerate-wizard.ts:140-166`)在 apply 成功后 `await new Promise(r => setTimeout(r, 0))` —— 一行非常精妙,给 React 机会 flush UI 后再 reset,避免 wizard 关闭时的视觉抖动。

#### 使用示例

```tsx
import { useRegenerateWizard } from '@/hooks/use-regenerate-wizard';

function RegenerateButton({ resumeId, onSuccess }: { resumeId: string; onSuccess: () => void }) {
  const w = useRegenerateWizard({
    resumeId,
    outputLanguage: 'en',
    onSuccess,
  });

  return (
    <>
      <Button onClick={w.startRegenerate}>Regenerate Selected</Button>
      {w.step === 'selecting' && (
        <ItemPicker
          items={allItems}
          selected={w.selectedItems}
          onChange={w.setSelectedItems}
        />
      )}
      {w.step === 'instructing' && (
        <InstructionDialog
          value={w.instruction}
          onChange={w.setInstruction}
          onConfirm={w.generate}
        />
      )}
      {w.step === 'previewing' && (
        <DiffView items={w.regeneratedItems} onAccept={w.acceptChanges} onReject={w.rejectAndRegenerate} />
      )}
    </>
  );
}
```

---

## 四、工具函数

### 4.1 `lib/utils.ts` —— cn + 日期归一化

`apps/frontend/lib/utils.ts:11-13` 提供 `cn()`,基于 `clsx` + `tailwind-merge`,是处理条件 className 的标准工具。

`utils.ts:28-51` 的 `formatDateRange` 把 en-dash/em-dash 归一化为 ASCII hyphen-minus,并补全 `Jun 2025 Aug 2025` → `Jun 2025 - Aug 2025`。注释 `utils.ts:17` 说明这能保证 PDF 文本提取的 ATS 兼容性。

### 4.2 `lib/utils/download.ts`

- `downloadBlobAsFile`(`download.ts:1-15`):触发浏览器下载并 revoke object URL。
- `sanitizeFilename`(`download.ts:33-62`):把任意标题转成安全的 PDF 文件名,使用 `Array.from()` 处理多字节字符避免截断 emoji/中文。
- `buildResumeFilename`(`download.ts:68-92`):构造 `{Name} - {Type} - {Company}.pdf` 模式,各字段缺失时优雅降级。

### 4.3 `lib/utils/section-helpers.ts`

围绕 `SectionMeta`(`components/dashboard/resume-component.tsx` 导出)操作:

- `DEFAULT_SECTION_META`(`section-helpers.ts:16-71`):6 个内建章节的元数据。
- `localizeDefaultSectionMeta`(`section-helpers.ts:93-110`):**只**翻译用户没改过的英文默认名,避免覆盖自定义。
- `withLocalizedDefaultSections`(`section-helpers.ts:118-125`):用 i18n 包裹整个 `ResumeData`,用于 print 路由注入。
- `getSortedSections` / `getAllSections`(`section-helpers.ts:137-148`):按 `order` 排序,前者过滤隐藏。
- `createCustomSection`(`section-helpers.ts:165-182`):自动生成 `custom_<n>` 唯一 ID。

### 4.4 `lib/utils/keyword-matcher.ts`

- 200+ 词停用词表(`keyword-matcher.ts:6-200`)。
- `extractKeywords`(`keyword-matcher.ts:209-223`):切词 + 过滤。
- `segmentTextByKeywords`(`keyword-matcher.ts:229-257`):返回 `{text, isMatch}[]`,前端高亮匹配。
- `calculateMatchStats`(`keyword-matcher.ts:262-285`):计算匹配百分比。

### 4.5 `lib/utils/html-sanitizer.ts`

`apps/frontend/lib/utils/html-sanitizer.ts:21-27` 用 `isomorphic-dompurify` 提供白名单:

```ts
const ALLOWED_TAGS = ['strong', 'em', 'u', 'a'];
const ALLOWED_ATTR = ['href', 'target', 'rel'];
```

`FORCE_BODY: true` 让 sanitize 输出 body 元素而非 DocumentFragment,以便直接用于 `dangerouslySetInnerHTML`。

---

## 五、类型定义

| 文件 | 内容 |
|------|------|
| `lib/types/template-settings.ts` | `TemplateType` / `PageSize` / `AccentColor` / `MarginSettings` / `TemplateSettings` 等;`DEFAULT_TEMPLATE_SETTINGS`、`settingsToCssVars`(`template-settings.ts:159-196`)、`TEMPLATE_OPTIONS` |
| `lib/types/lucide.d.ts` | 给 `lucide-react/dist/esm/icons/*` 加 module declaration,支持 deep path import 的类型推断 |

`lib/types/template-settings.ts:159-196` 的 `settingsToCssVars` 把所有 spacing/font 配置转成 CSS 变量,在 `print/resumes/[id]/page.tsx` 里 `<Resume style={settingsToCssVars(settings)} />` 直接消费,避免手写 `style={{ '--section-gap': ... }}`。

---

## 六、常量管理

| 文件 | 常量 |
|------|------|
| `lib/config/version.ts` | `APP_VERSION = '1.2'`、`APP_CODENAME = 'Nightvision'`、`APP_NAME` 与 `getVersionString` |
| `lib/constants/page-dimensions.ts` | `PAGE_DIMENSIONS`(A4/LETTER mm)、`mmToPx`、`getContentArea`、`calculatePreviewScale`(`page-dimensions.ts:14-73`) |

`calculatePreviewScale`(`page-dimensions.ts:59-62`)把页面 mm 换算成 96 DPI 像素,供 `components/preview/` 缩放预览。

---

## 七、国际化

### 7.1 双目录结构

| 路径 | 角色 |
|------|------|
| `i18n/config.ts` | locale 元数据(5 种语言 + 名称 + flag) |
| `lib/i18n/` | 翻译引擎:客户端 hook、server util、messages 索引、嵌套查找 |
| `messages/*.json` | 翻译文本,文件名 `pt-BR.json` 被 import 为 `pt` |

`i18n/config.ts:5` 的 `locales = ['en','es','zh','ja','pt']` 是 source of truth;**注意** `docs/agent/features/i18n.md` 表格已过期,实际项目支持葡萄牙语(`apps/frontend/CLAUDE.md:90`)。

### 7.2 Messages 索引与类型对齐

`apps/frontend/lib/i18n/messages.ts:9-17` 是项目里**最容易出 bug** 的地方:

```ts
export type Messages = typeof en;
const allMessages: Record<Locale, Messages> = { en, es, zh, ja, pt };
```

因为 5 个 locale 都被强类型化为 `Messages`(= `en.json` 的精确形状),只要给 `en.json` 加一个 key,`tsc` / `next build` 就会在 es/zh/ja/pt-BR 中寻找同样的路径并失败。这是项目**主动选择**的"结构漂移即刻 fail"的策略,见 `apps/frontend/CLAUDE.md:97-105`。

### 7.3 客户端 hook 与 server util

- `lib/i18n/translations.ts:16-37` 的 `useTranslations` 返回 `{ t, messages, locale }`,`t(key, params)` 支持 `dot.path` 与 `{placeholder}` 替换。
- `lib/i18n/translations.ts:47-55` 的 `translate` 是 server 端等价物,供 print 路由使用。
- `lib/i18n/utils.ts:1-24` 提供 `getNestedValue` 与 `applyParams`,`getNestedValue` 在 key 缺失时**返回原 key 字符串**,不抛错 —— 这是一种降级策略,UI 会显示"common.save"而不是崩。
- `lib/i18n/locale.ts:3-8` 的 `resolveLocale` 把 `searchParams.lang` 限定到合法 locale,避免越界。

### 7.4 使用示例

```tsx
'use client';
import { useTranslations } from '@/lib/i18n';

export default function SaveButton() {
  const { t, locale } = useTranslations();
  return (
    <button aria-label={t('common.saveAria')}>
      {t('common.save', { count: 3 })}
    </button>
  );
}
```

`print/resumes/[id]/page.tsx:13` 则是 server-side 用法:

```ts
import { translate } from '@/lib/i18n/server';
const localized = translate(locale, 'resume.sections.summary');
```

---

## 八、Context 消费模式速查

```tsx
import { useStatusCache } from '@/lib/context/status-cache';
import { useLanguage } from '@/lib/context/language-context';

function AnyClientComponent() {
  const { status, isLoading, refreshStatus, incrementResumes } = useStatusCache();
  const { contentLanguage, setContentLanguage, uiLanguage, setUiLanguage } = useLanguage();

  if (isLoading) return <Spinner />;
  return <div>{status?.database_stats.total_resumes} resumes</div>;
}
```

每个 hook 都会在 Provider 外抛错(`status-cache.tsx:218-224`、`language-context.tsx:115-121`),所以在测试或迁移时缺少 Provider 会立即被定位。

---

## 九、最佳实践总结

1. **后端访问 100% 走 `lib/api/*`**:组件层永远不要 `fetch('/api/...')`,否则会绕过 240s 超时和 AbortError 翻译。`apps/frontend/CLAUDE.md:65` 明确禁止。
2. **i18n 增删 key 必须同步 5 个 JSON**:`en.json` 是其他 4 个文件的形状模板,build 阶段会严格校验。
3. **Context 暴露的 setter 保持稳定**:在 `useCallback` 包装并放进 Provider 的 `value` 中,避免消费组件无效重渲染。
4. **不可变更新**:Reducer 和 Context 内部都使用 spread,例如 `status-cache.tsx:99-110`。
5. **乐观更新 + 失败回滚**:`LanguageProvider.setContentLanguage`(`language-context.tsx:64-87`)是范例 —— 立即更新 UI,再同步后端,失败时回滚。
6. **错误路径携带后端 detail**:`data.detail || 'Fallback (status X).'`(`lib/api/config.ts:91-93` 等)是统一模式,既保留后端语义又不会因 `res.json()` 失败丢失上下文。

---

## 十、阅读顺序建议

1. 先读 `lib/api/client.ts:1-75`,理解 240s 超时 + 内部 origin 切换。
2. 接着看 `lib/api/resume.ts:111-135` 的 `postImprove`,体会"先 text 再 parse"的容错思路。
3. 再看 `lib/context/status-cache.tsx`,掌握 Context + 乐观更新的写法。
4. 然后按顺序阅读三个 Hook,关注 Reducer 与 `useState` 的取舍。
5. 最后阅读 `lib/i18n/*`,理解 build-time 结构校验的设计动机。

完成本篇后,你已经掌握前端"客户端状态 + 网络层"的全部主干,可以继续阅读 [前端核心模块] 与 [前端特性与流程] 了解具体业务页面与端到端流程。
