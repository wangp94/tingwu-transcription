---
name: "tingwu-transcription"
description: "Upload audio/video files to Alibaba Tongyi Tingwu (通义听悟), start transcription tasks, and export results to local files via browser automation. Invoke when user wants to transcribe audio files, upload media to Tingwu, retrieve transcription results, or asks for speech-to-text conversion."
---

# 通义听悟音频转写 (Tingwu Transcription)

通过浏览器自动化在阿里云通义听悟平台上传音视频文件、启动转写任务，并将转写结果导出保存到本地。

## 适用场景

- 用户有本地音视频文件需要转写为文字
- 用户提到"通义听悟"、"听悟"、"转写"、"语音转文字"
- 用户要求上传音频/视频文件并生成文字记录
- 用户要求获取/下载/导出转写结果到本地

## 前置条件

1. **浏览器已就绪**：需要使用内置浏览器工具
2. **用户已登录**：通义听悟需要阿里云账号登录，如未登录需提示用户手动完成登录
3. **文件路径有效**：用户提供本地音视频文件的绝对路径

## 支持的文件格式

| 类型 | 格式 | 单文件大小限制 |
|------|------|---------------|
| 视频 | mp4/wmv/m4v/flv/rmvb/dat/mov/mkv/webm/avi/mpeg/3gp/ogg | 6GB |
| 音频 | mp3/wav/m4a/wma/aac/ogg/amr/flac/aiff | 500MB |

> 单个文件最长 6 小时，单次最多可上传 50 个文件。

## 工作流程

### 第一阶段：上传文件并启动转写

#### Step 1: 导航到通义听悟首页

```javascript
await tools.browser_navigate({ url: 'https://tingwu.aliyun.com/home' });
await tools.browser_wait_for({ time: 3 });
const snap = await tools.browser_snapshot();
text(snap);
```

**检查登录状态**：
- 如果快照中出现"立即登录"按钮，说明用户尚未登录
- 此时必须调用 `browser_waiting_for_user_interaction`（作为独立工具调用，不在 Exec 内）提示用户手动登录
- 登录完成后继续后续步骤

#### Step 2: 点击"上传音视频"按钮

首页主内容区有两个功能卡片。需要点击右侧的"上传音视频"卡片。

**方法 A：通过 evaluate 查找并点击**（推荐，因为页面使用自定义组件，快照可能无法捕获完整元素）

```javascript
await tools.browser_evaluate({
  script: `(() => {
    const all = document.querySelectorAll('*');
    for (const el of all) {
      const text = (el.textContent || '').trim();
      if (text.includes('上传音视频') && text.length < 20) {
        el.click();
        return 'clicked: ' + el.tagName;
      }
    }
    return 'not-found';
  })()`
});
await tools.browser_wait_for({ time: 2 });
const snap = await tools.browser_snapshot();
text(snap);
```

点击后会弹出上传方式选择对话框，包含两个选项：
- **上传本地音视频文件**（本地上传）
- **导入阿里云盘文件**（云盘导入）

#### Step 3: 选择"上传本地音视频文件"

```javascript
await tools.browser_evaluate({
  script: `(() => {
    const all = document.querySelectorAll('*');
    for (const el of all) {
      const text = (el.textContent || '').trim();
      if (text.includes('上传本地音视频文件') && text.length < 30) {
        el.click();
        return 'clicked: ' + el.tagName;
      }
    }
    return 'not-found';
  })()`
});
await tools.browser_wait_for({ time: 2 });
const snap = await tools.browser_snapshot();
text(snap);
```

此时会出现文件拖拽/点击上传区域，快照中会有一个包含"点击 / 拖拽 本地音视频文件到这里"文字的按钮元素（如 `ref: e3`）。

弹窗右侧还包含以下配置选项（自定义组件，快照无法捕获，需通过 `browser_evaluate` 操作）：

| 配置项 | 可选值 | 说明 |
|--------|--------|------|
| 音视频语言 | 中文 / 英语 / 日语 / 粤语 / 中英文自由说 | 默认选中"中文" |
| 翻译 | 不翻译 / 其他语言 | 默认"不翻译" |
| **区分发言人** | **暂不体验 / 单人演讲 / 2人对话 / 多人讨论** | **默认未选中，需手动选择** |

#### Step 4: 选择"区分发言人"模式（推荐）

在上传文件之前或之后、点击"开始转写"之前，需要选择发言人区分模式。**这一步很容易被忽略，但对转写质量至关重要。**

根据音频场景选择合适的模式：
- **暂不体验**：不区分发言人
- **单人演讲**：单人讲话场景（如讲座、独白）
- **2人对话**：双人对谈场景（如采访、播客）
- **多人讨论**：多人会议/讨论场景（最常用）

**通过 evaluate 点击对应选项**：

```javascript
await tools.browser_evaluate({
  script: `(() => {
    const all = document.querySelectorAll('*');
    for (const el of all) {
      if (el.children.length === 0) {
        const text = (el.textContent || '').trim();
        if (text === '多人讨论') {  // 替换为实际需要的模式：暂不体验/单人演讲/2人对话/多人讨论
          el.click();
          return;
        }
      }
    }
  })()`
});
await tools.browser_wait_for({ time: 1 });
```

> **验证方法**：截图确认目标选项是否变为浅紫色高亮状态（选中态）。
>
> ```javascript
> const screenshot = await tools.browser_take_screenshot({});
> text(screenshot);
> ```

#### Step 5: 上传音视频文件

使用 `browser_upload_file` 工具将本地文件上传到文件输入区域。需要使用上一步快照中上传区域的 ref。

```javascript
const result = await tools.browser_upload_file({
  ref: 'e3',  // 替换为实际快照中上传区域的 ref
  filePath: '/path/to/your/audio.mp3'  // 替换为用户提供的文件路径
});
text(result);
```

上传后快照会显示文件信息（文件名、大小），且"开始转写"按钮从 disabled 变为可点击状态。

#### Step 6: 点击"开始转写"

```javascript
// 从上一步快照中获取"开始转写"按钮的 ref
await tools.browser_click({ ref: 'e4' });  // 替换为实际 ref
await tools.browser_wait_for({ time: 3 });
const snap = await tools.browser_snapshot();
text(snap);
```

#### Step 7: 确认任务提交状态

点击"开始转写"后，页面会跳转到任务列表。观察快照中的状态信息：

- **"已上传 (XX%)"**：文件正在上传中，需等待
- **"等待转写"**：文件上传完成，已进入转写队列
- **"转写中"**：正在处理转写
- **"转写完成"**：转写已完成，可查看结果

如显示"已上传 (XX%)"，需等待并重新检查：

```javascript
await tools.browser_wait_for({ time: 5 });
const snap = await tools.browser_snapshot();
text(snap);
```

重复直到状态变为"等待转写"或"转写中"。

### 第二阶段：获取转写结果并保存到本地

#### Step 8: 等待转写完成

转写任务通常需要几分钟到几十分钟不等（取决于音频时长）。需要定期检查转写状态。

**方法：进入"我的记录"页面检查状态**

```javascript
await tools.browser_navigate({ url: 'https://tingwu.aliyun.com/folders/0' });
await tools.browser_wait_for({ time: 3 });
const snap = await tools.browser_snapshot();
text(snap);
```

在"我的记录"页面中，可以通过截图查看卡片状态。如果卡片上显示了转写文本预览、关键词标签等信息，说明转写已完成。如果仍显示"转写中"或"等待转写"，需要继续等待。

**检查方法**：截图并观察卡片内容

```javascript
const screenshot = await tools.browser_take_screenshot({});
text(screenshot);
```

> **重要提示**：通义听悟的卡片列表使用自定义渲染组件，`browser_snapshot` 通常无法捕获卡片内部元素和 ref。必须通过截图或 `browser_evaluate` 来检查状态。

#### Step 9: 进入转写结果详情页

转写完成后，需要点击对应的卡片进入详情页。**这是最关键的一步**，因为卡片是自定义组件，`browser_snapshot` 无法捕获卡片 ref，`browser_click` 的坐标点击也可能无效。

**推荐方法：通过 evaluate 查找叶子节点并向上遍历点击卡片容器**

```javascript
await tools.browser_navigate({ url: 'https://tingwu.aliyun.com/folders/0' });
await tools.browser_wait_for({ time: 3 });

await tools.browser_evaluate({
  script: `(() => {
    const all = document.querySelectorAll('*');
    let target = null;
    // 查找包含目标文件名或关键词的叶子节点
    for (const el of all) {
      if (el.children.length === 0) {
        const text = (el.textContent || '').trim();
        // 用文件名的关键部分匹配（如文件名前几位或独特关键词）
        if (text.includes('文件名关键词') && text.length < 150) {
          target = el;
          break;
        }
      }
    }
    // 从叶子节点向上找到卡片容器并点击
    if (target) {
      let card = target;
      while (card.parentElement && card.parentElement.children.length < 4) {
        card = card.parentElement;
      }
      card.click();
    }
  })()`
});
await tools.browser_wait_for({ time: 3 });
const snap = await tools.browser_snapshot();
text(snap);
```

**成功标志**：页面 URL 变为 `https://tingwu.aliyun.com/doc/transcripts/{transcriptId}`，页面标题显示文件名。

> **关键技巧**：
> - 使用文件名的独特部分作为匹配关键词（如 UUID 前几位）
> - 从叶子节点向上遍历时，通过 `children.length < 4` 找到卡片级容器
> - 如果第一次点击没有跳转，可能是匹配到了非可点击的元素，尝试调整 `children.length` 的阈值
> - **不要使用坐标点击**（`offsetX/offsetY`），对自定义组件无效

#### Step 10: 点击"导出"按钮

在转写结果详情页，左侧边栏有"导出"按钮（带"得时长"角标）。

```javascript
await tools.browser_evaluate({
  script: `(() => {
    const all = document.querySelectorAll('*');
    for (const el of all) {
      if (el.children.length === 0) {
        const text = (el.textContent || '').trim();
        if (text === '导出') {
          el.click();
          break;
        }
      }
    }
  })()`
});
await tools.browser_wait_for({ time: 2 });
const snap = await tools.browser_snapshot();
text(snap);
```

弹出导出对话框，包含两个选项：
- ** 导出到本地**
- ** 导出到阿里云盘**

#### Step 11: 点击" 导出到本地"

```javascript
// 从快照中获取" 导出到本地"按钮的 ref
await tools.browser_click({ ref: 'e16' });  // 替换为实际 ref
await tools.browser_wait_for({ time: 3 });
```

点击后，页面会显示"加载中"状态，表示正在生成导出文件。此时浏览器会触发系统级文件保存对话框，**浏览器自动化无法操作此对话框**。

#### Step 12: 通过拦截导出接口响应捕获 OSS 下载 URL

> **实测经验（重要）**：点击"导出到本地"后，浏览器触发的是**原生下载**，那条 OSS 下载 GET **不会出现在 `browser_network_requests` 列表里**（该工具只能看到页面发起的请求，抓不到原生下载）。所以**不要**依赖网络请求列表去找 URL，否则会卡死在这一步。
>
> 正确做法：在页面内 hook `fetch` / `XMLHttpRequest`，拦截 `POST /api/export/request?c=web` 的**响应体**，从响应 JSON 里提取 OSS 预签名 URL。该逻辑运行在页面 JS 上下文，**与操作系统无关，macOS 与 Windows 通用**。

**Step 12.1：在点击"导出到本地"之前，注入拦截钩子**（顺序很关键，必须先注入再点击）

```javascript
await tools.browser_evaluate({
  script: `(() => {
    if (window.__ph) return;
    window.__ph = true;
    window.__exp = [];
    // 从响应文本中提取 OSS/导出相关 URL
    const pick = (t) => {
      const m = (t || '').match(/https?:\\/\\/[^"\\s<>\\\\]+/g) || [];
      return m.filter(u => u.includes('oss') || u.includes('export'));
    };
    const of = window.fetch;
    window.fetch = async function (...a) {
      const url = (typeof a[0] === 'string') ? a[0] : (a[0] && a[0].url) || '';
      const res = await of.apply(this, a);
      try {
        if (String(url).includes('/api/export/request')) {
          const c = res.clone();
          const t = await c.text();
          window.__exp.push.apply(window.__exp, pick(t));
        }
      } catch (e) {}
      return res;
    };
    const oOpen = XMLHttpRequest.prototype.open, oSend = XMLHttpRequest.prototype.send;
    XMLHttpRequest.prototype.open = function (m, u) { this.__u = u; return oOpen.apply(this, arguments); };
    XMLHttpRequest.prototype.send = function () {
      try {
        this.addEventListener('load', () => {
          if (String(this.__u || '').includes('/api/export/request')) {
            try { window.__exp.push.apply(window.__exp, pick(String(this.responseText))); } catch (e) {}
          }
        });
      } catch (e) {}
      return oSend.apply(this, arguments);
    };
  })()`
});
```

**Step 12.2：点击"导出到本地"**

```javascript
// 从快照中获取"导出到本地"按钮的 ref
await tools.browser_click({ ref: 'e16' });  // 替换为实际 ref
await tools.browser_wait_for({ time: 4 });
```

**Step 12.3：读取捕获到的 OSS URL**

`browser_evaluate` 不回传返回值，需把结果写到可被快照读取的位置（这里用 `document.title`），再通过快照读取。

```javascript
await tools.browser_evaluate({
  script: `(() => { document.title = 'EXP:' + JSON.stringify(window.__exp || []); })()`
});
await tools.browser_wait_for({ time: 1 });
const snap = await tools.browser_snapshot();
text(snap);
```

在快照的 `Page Title`（形如 `# EXP:["https://..."]`）中读取完整 OSS URL。

**OSS 下载 URL 的特征**：
- 域名：`prod-new-tingwu-saas-common-shanghai-temp-resource.oss-cn-shanghai.aliyuncs.com`
- 路径包含：`/tingwu/prod/export_result/`
- URL 参数包含：`Expires=`、`OSSAccessKeyId=`、`Signature=`、`response-content-disposition=attachment`
- `response-content-disposition` 参数中包含文件名，格式为 `filename*=UTF-8''{文件名}_原文.docx`
- 完整示例（一次实测捕获到的返回值）：
```
https://prod-new-tingwu-saas-common-shanghai-temp-resource.oss-cn-shanghai.aliyuncs.com/tingwu/prod/export_result/2026/10/14/1841087/48487629/e498d8cdd70a43b2acc0a262aa9ade3a?Expires=1792025654&OSSAccessKeyId=LTAI5tL5omczaShYRZYbCQDT&Signature=870i3MahX8DBKNRxzWYrwNqe2eU%3D&response-content-disposition=attachment%3Bfilename%2A%3DUTF-8%27%27%E9%9F%B3%E9%A2%91%E7%89%87%E6%AE%B5_%E5%8E%9F%E6%96%87.docx
```

> **重要提示**：
> - OSS 预签名 URL 有时效限制（通常几小时），需要在获取后尽快下载
> - 如果 URL 已过期或丢失，钩子仍保留在页面内，重新走一次"导出 → 导出到本地"即可再次捕获新 URL
> - 该方案与操作系统无关，macOS / Windows 表现一致

#### Step 13: 下载文件到本地（跨平台）

获取到 OSS 下载 URL 后，使用 `RunCommand` 通过命令行直接下载，绕过浏览器系统级文件保存对话框。

```bash
# macOS / Linux：curl 通常自带
curl -L -o "保存路径/文件名.docx" "完整的OSS URL" -w "\n%{http_code} %{size_download}"
```

```powershell
# Windows 10+：系统自带 curl.exe，可直接用上面的命令；
# 若无 curl，改用 PowerShell：
Invoke-WebRequest -Uri "完整的OSS URL" -OutFile "保存路径\文件名.docx"
```

**注意事项**：
- URL 必须用双引号包裹（包含特殊字符如 `&`、`%`）
- `-L` 参数跟随重定向
- `-o` 指定保存路径（建议保存到 workspace 目录）
- `-w` 输出 HTTP 状态码和文件大小用于验证
- 成功的响应：HTTP 200 + 文件大小 > 0

**验证下载结果**：

```bash
ls -lh "保存路径/文件名.docx"
```

确认文件存在且大小合理（转写结果通常为几 KB 到几十 KB）。

#### Step 14: 向用户确认结果

下载完成后，使用 `computer://` 链接向用户展示下载的文件：

```
转写结果已保存为 [文件名.docx](computer:///path/to/文件名.docx)
```

## 执行要点（示例代码见上文各 Step，此处不再重复）

整个流程需拆分为多个 Exec 依次执行，关键切分点如下：

1. **上传（Step 2–3）**：点击"上传音视频"→ 选择"上传本地音视频文件"，本 Exec 结束前从快照读取上传区域 ref。
2. **选场景并开始转写（Step 4–6）**：选择"区分发言人"模式 → 上传文件 → 点击"开始转写"，每步依赖上一步快照的 ref。
3. **等待转写完成（Step 8）**：轮询"我的记录"页面，直到卡片显示转写完成。
4. **导出并拦截响应（Step 11–12）**：先注入钩子，再点击"导出到本地"，然后读取捕获到的 OSS URL。
5. **下载（Step 13）**：用命令行下载 OSS URL 到本地。

**易错点**：上传后需检查上传进度是否推进；转写需轮询到"转写完成"再进入详情页；导出钩子必须在点击"导出到本地"**之前**注入。

## 关键 URL 结构

| 页面 | URL 模式 |
|------|---------|
| 首页 | `https://tingwu.aliyun.com/home` |
| 我的记录（默认文件夹） | `https://tingwu.aliyun.com/folders/0` |
| 转写结果详情页 | `https://tingwu.aliyun.com/doc/transcripts/{transcriptId}` |
| 导出 API | `POST https://tingwu.aliyun.com/api/export/request?c=web` |
| OSS 下载链接 | `GET https://prod-new-tingwu-saas-common-shanghai-temp-resource.oss-cn-shanghai.aliyuncs.com/tingwu/prod/export_result/...` |

## 异常处理与重试机制

在实际操作中，通义听悟页面可能出现响应不及时、加载缓慢、弹窗未弹出等问题。以下是各环节的 fallback 处理策略。

### 通用原则

- **增量等待**：优先使用短时增量等待（1-3 秒）+ 状态检查，而非单次长等待
- **截图确认**：每次关键操作后截图确认实际页面状态，而非假设操作已生效
- **最多重试 3 次**：同一操作最多重试 3 次，仍失败则停止并向用户报告
- **不要盲目重试**：每次重试前先截图/快照分析失败原因，调整策略后再试

### 场景 1：导航后页面未加载完成

**症状**：`browser_navigate` 后快照元素很少（仅 2-3 个），页面内容为空白

**处理**：

```javascript
// 增量等待 + 重试，最多 3 次
await tools.browser_navigate({ url: 'https://tingwu.aliyun.com/home' });
let loaded = false;
for (let i = 0; i < 3; i++) {
  await tools.browser_wait_for({ time: 2 });
  const snap = await tools.browser_snapshot();
  // 检查是否有足够多的元素（说明页面已渲染）
  if (snap.content[0].text.includes('通义听悟')) {
    loaded = true;
    break;
  }
}
if (!loaded) {
  // 最后一次尝试：刷新页面
  await tools.browser_navigate({ url: 'https://tingwu.aliyun.com/home' });
  await tools.browser_wait_for({ time: 5 });
}
const snap = await tools.browser_snapshot();
text(snap);
```

### 场景 2：evaluate 点击后弹窗未出现

**症状**：通过 `browser_evaluate` 点击"上传音视频"或"上传本地音视频文件"后，截图显示弹窗未弹出

**处理**：

```javascript
// 合并为一次点击：先点元素本身，弹窗未出现时改为点击父元素兜底
await tools.browser_evaluate({
  script: `(() => {
    const all = document.querySelectorAll('*');
    for (const el of all) {
      const text = (el.textContent || '').trim();
      if (text.includes('上传音视频') && text.length < 20) {
        el.click();  // 若弹窗未出现，改为点击父元素兜底：if (el.parentElement) el.parentElement.click();
        return;
      }
    }
  })()`
});
await tools.browser_wait_for({ time: 2 });
const screenshot = await tools.browser_take_screenshot({});
text(screenshot);
```

### 场景 3：上传后"开始转写"按钮仍为 disabled

**症状**：文件已通过 `browser_upload_file` 上传，但快照显示"开始转写"按钮仍为 `states: [disabled]`

**处理**：

```javascript
// 等待文件处理完成，增量检查
let snap = await tools.browser_snapshot();
let retryCount = 0;
while (snap.content[0].text.includes('disabled') && retryCount < 5) {
  await tools.browser_wait_for({ time: 3 });
  snap = await tools.browser_snapshot();
  retryCount++;
}
text(snap);
// 如果 5 次后仍 disabled，可能是文件格式不支持或上传失败
// 截图检查是否有错误提示，向用户报告
if (snap.content[0].text.includes('disabled')) {
  const screenshot = await tools.browser_take_screenshot({});
  text(screenshot);
  // 向用户报告：文件可能上传失败，请检查文件格式和大小
}
```

### 场景 4：文件上传进度卡住

**症状**：点击"开始转写"后，状态长时间显示"已上传 (XX%)"且百分比不变化

**处理**：每隔约 5s 快照一次，提取"已上传 (XX%)"中的百分比并与上次比较；连续 3 次百分比数字不变即视为卡住，若中途出现"等待转写"/"转写中"则说明上传已完成。关键判断：

`if (match && match[1] === lastProgress) stuckCount++;  // stuckCount ≥ 3 判定卡住`

### 场景 5：点击卡片后未跳转到详情页

**症状**：通过 `browser_evaluate` 点击转写卡片后，URL 仍为 `/folders/0`，未跳转到 `/doc/transcripts/{id}`

**处理**：逐步加大向上遍历的 `children.length` 阈值（如 4 → 6 → 8 → 10）并重试点击，每次等待约 3s 后检查是否跳转；仍失败则回到列表重新打开卡片。关键判断：

`if (snap.content[0].text.includes('/doc/transcripts/')) break;  // 已跳转则停止重试`

### 场景 6：网络请求里找不到 OSS 下载 URL（预期现象，改用钩子）

**症状**：点击"导出到本地"后，`browser_network_requests` 列表中找不到 OSS 下载链接

**原因**：导出到本地走的是浏览器**原生下载**，不会进入页面网络请求层，因此该工具抓不到——这是**正常现象，不是故障**，不要反复重试网络抓取。

**处理**：改用 Step 12 的钩子方案拦截导出接口响应：

```javascript
// 1) 先注入钩子（完整代码见 Step 12.1；若此前已注入可跳过）
await tools.browser_evaluate({ script: `/* 见 Step 12.1 的完整钩子代码 */` });

// 2) 打开导出弹窗
await tools.browser_evaluate({
  script: `(() => {
    const all = document.querySelectorAll('*');
    for (const el of all) {
      if (el.children.length === 0 && (el.textContent || '').trim() === '导出') { el.click(); break; }
    }
  })()`
});
await tools.browser_wait_for({ time: 2 });

// 3) 快照获取"导出到本地"按钮 ref 并点击
const snap = await tools.browser_snapshot();
text(snap);

// 4) 点击"导出到本地"后，把捕获结果写入 document.title，再快照读取
await tools.browser_evaluate({
  script: `(() => { document.title = 'EXP:' + JSON.stringify(window.__exp || []); })()`
});
await tools.browser_snapshot();
```

### 场景 7：curl 下载失败（URL 过期或网络错误）

**症状**：curl 返回非 200 状态码，或文件大小为 0

**处理**：

```bash
# 第一次下载失败后，检查 HTTP 状态码
# 如果 403（URL 过期）：需要重新回到浏览器触发导出，获取新的 OSS URL
# 如果 408/504（超时）：增加超时参数重试
# 如果 0（网络错误）：检查 URL 是否完整，特别是 & 和 % 特殊字符是否被截断

# 重试下载（增加超时和重试次数）
curl -L --retry 3 --retry-delay 5 --connect-timeout 30 \
  -o "保存路径/文件名.docx" \
  "完整的OSS URL" \
  -w "\n%{http_code} %{size_download}"

# 如果仍失败，验证 URL 完整性
echo "URL长度: ${#URL}"
echo "URL是否包含Expires: $(echo $URL | grep -c 'Expires')"
echo "URL是否包含Signature: $(echo $URL | grep -c 'Signature')"
```

> 上面为 macOS/Linux 写法；Windows 没有 `${#URL}`/`grep`，改用 PowerShell：
> `Invoke-WebRequest -Uri "完整URL" -OutFile "保存路径\文件名.docx"`，再用 `(Get-Item "保存路径\文件名.docx").Length` 检查文件大小。
> 若 403（URL 过期），钩子仍在页面内，重新走一次"导出 → 导出到本地"再捕获新 URL 即可。

## 注意事项

1. **Ref 时效性**：每次页面 DOM 变化（点击、导航）后，之前的 ref 会失效，必须重新获取快照
2. **Exec 拆分**：当需要从快照中读取 ref 来决定下一步操作时，必须结束当前 Exec，在下一个 Exec 中使用新 ref
3. **登录处理**：如遇未登录状态，必须使用 `browser_waiting_for_user_interaction`（独立调用，不在 Exec 内）让用户手动登录
4. **文件大小**：大文件上传可能需要较长时间，需多次等待并检查上传进度
5. **自定义组件**：通义听悟使用自定义 UI 组件，快照可能无法捕获所有可交互元素，优先使用 `browser_evaluate` 查找和点击元素
6. **并发限制**：不要并行执行依赖顺序的浏览器操作，确保每一步完成后再执行下一步
7. **卡片点击技巧**：详见 Step 9。
8. **导出下载（重要）**：详见 Step 12、Step 13。
9. **OSS URL 有效期**：预签名 URL 有时效限制（通常几小时），获取后需尽快下载；如过期需重新触发导出
10. **导出格式**：通义听悟导出为 Word（.docx）格式，文件名格式为 `{音频文件名}_原文.docx`
11. **转写等待时间**：转写任务可能需要几分钟到几十分钟，取决于音频时长；需定期检查状态
12. **区分发言人**：详见 Step 4。
