# 任务图片存储功能实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 为日程打卡应用添加图片存储功能，支持任务模板和每日打卡记录关联图片，使用 IndexedDB 存储图片数据，并提供自动清理过期图片的机制。

**Architecture:** 图片二进制数据存入 IndexedDB（通过 idb 库封装），图片 ID 引用列表存入现有 localStorage state 中。上传时自动压缩（最大宽度 1920px，JPEG 质量 0.8）。应用启动时自动清除超过保留天数的过期图片。

**Tech Stack:** 原生 HTML/CSS/JS, idb (CDN), IndexedDB, Canvas API

---

## File Structure

所有修改集中在一个文件：`task_notice.html`

| 区域 | 变更内容 |
|------|----------|
| `<head>` | 添加 idb CDN 引用 |
| CSS | 新增：图片上传区、缩略图网格、全屏查看器、图片操作按钮样式 |
| HTML settingsPanel | 新增「图片保留天数」输入框 |
| HTML templateForm | 新增图片上传区 |
| HTML taskDialog | 新增图片上传区 |
| HTML body 末尾 | 新增全屏查看器 overlay |
| JS 顶部 | 新增 IndexedDB 常量和初始化 |
| JS state 模型 | settings 增加 imageRetentionDays，templates/tasks 增加 images 数组 |
| JS 新增函数 | imageDB 操作、图片压缩、上传处理、缩略图渲染、全屏查看、启动清理 |

---

### Task 1: 添加 idb CDN 引用和 IndexedDB 基础模块

**Files:**
- Modify: `task_notice.html` `<head>` 部分（约第 6 行后）
- Modify: `task_notice.html` `<script>` 部分（约第 847 行后）

- [ ] **Step 1: 在 `<head>` 中添加 idb CDN**

在 `<head>` 标签内的 `<title>` 之后添加：

```html
<script src="https://cdn.jsdelivr.net/npm/idb@8/build/umd.js"></script>
```

- [ ] **Step 2: 添加 IndexedDB 常量和初始化代码**

在 `<script>` 标签内，`const STORAGE_KEY` 之前添加：

```javascript
const IMAGE_DB_NAME = "daily-checkin-images";
const IMAGE_DB_VERSION = 1;
const IMAGE_STORE = "images";
const IMAGE_MAX_WIDTH = 1920;
const IMAGE_QUALITY = 0.8;

let imageDB = null;

async function initImageDB() {
  imageDB = await idb.openDB(IMAGE_DB_NAME, IMAGE_DB_VERSION, {
    upgrade(db) {
      if (!db.objectStoreNames.contains(IMAGE_STORE)) {
        db.createObjectStore(IMAGE_STORE, { keyPath: "id" });
      }
    }
  });
}
```

- [ ] **Step 3: 在 init() 中调用 initImageDB()**

修改 `init()` 函数，在 `els.dateInput.value = selectedDate;` 之前添加：

```javascript
initImageDB().then(() => cleanupExpiredImages());
```

完整 `init()` 函数变为：

```javascript
function init() {
  initImageDB().then(() => cleanupExpiredImages());
  els.dateInput.value = selectedDate;
  ensureDay(selectedDate);
  bindEvents();
  updateTemplateWeekdayVisibility();
  applyStoredTemplateListHeight();
  renderSettings();
  renderAll();
}
```

- [ ] **Step 4: 验证页面加载无报错**

在浏览器中打开 `task_notice.html`，确认控制台无错误。

- [ ] **Step 5: Commit**

```bash
git add task_notice.html
git commit -m "feat: add idb CDN and IndexedDB initialization"
```

---

### Task 2: 更新 state 模型和数据迁移

**Files:**
- Modify: `task_notice.html` — `defaultSettings`、`loadState()`、`normalizeImportedState()`、`ensureDay()`

- [ ] **Step 1: 更新 defaultSettings**

修改 `defaultSettings`（约第 854 行）：

```javascript
const defaultSettings = {
  ownerName: "",
  motto: "固定任务、每日执行、历史回看都在这里。",
  imageRetentionDays: 180
};
```

- [ ] **Step 2: 更新 loadState() 确保 images 字段存在**

修改 `loadState()` 函数（约第 1065 行），在 `return saved;` 之前添加迁移逻辑：

```javascript
function loadState() {
  try {
    const saved = JSON.parse(localStorage.getItem(STORAGE_KEY));
    if (saved && Array.isArray(saved.templates) && saved.days) {
      saved.settings = { ...defaultSettings, ...(saved.settings || {}) };
      saved.templates.forEach((t) => { if (!t.images) t.images = []; });
      Object.values(saved.days).forEach((day) => {
        day.tasks.forEach((t) => { if (!t.images) t.images = []; });
      });
      return saved;
    }
  } catch (error) {
    console.warn("Load failed", error);
  }
  return { templates: defaultTemplates, days: {}, settings: { ...defaultSettings } };
}
```

- [ ] **Step 3: 更新 defaultTemplates 添加 images 字段**

修改 `defaultTemplates`（约第 849 行）：

```javascript
const defaultTemplates = [
  { id: uid(), name: "晨间整理", category: "生活", repeat: "daily", note: "规划今天最重要的三件事", images: [] },
  { id: uid(), name: "运动 20 分钟", category: "健康", repeat: "daily", note: "", images: [] },
  { id: uid(), name: "复盘记录", category: "学习", repeat: "workday", note: "写下一个收获或问题", images: [] }
];
```

- [ ] **Step 4: 更新 normalizeImportedState() 保留 images 字段**

修改 `normalizeImportedState()` 函数（约第 1130 行）中的 templates 映射：

```javascript
return {
  templates: candidate.templates.map((template) => ({
    id: template.id || uid(),
    name: String(template.name || "").trim() || "未命名任务",
    category: template.category || "其他",
    repeat: template.repeat || "daily",
    weekday: template.repeat === "weekly" ? Number(template.weekday ?? 1) : null,
    note: template.note || "",
    images: Array.isArray(template.images) ? template.images : []
  })),
  days: candidate.days,
  settings: { ...defaultSettings, ...(candidate.settings || {}) }
};
```

- [ ] **Step 5: 更新 ensureDay() 确保任务有 images 字段**

修改 `ensureDay()` 函数（约第 1149 行）中 tasks 的创建：

```javascript
function ensureDay(date) {
  if (state.days[date]) return;
  state.days[date] = {
    tasks: state.templates
      .filter((task) => shouldTemplateRunOnDate(task, date))
      .map((task) => ({
        ...task,
        repeat: task.repeat || "daily",
        id: uid(),
        templateId: task.id,
        done: false,
        createdFromTemplate: true,
        images: []
      }))
  };
  saveState();
}
```

- [ ] **Step 6: 验证页面加载无报错，数据正常显示**

- [ ] **Step 7: Commit**

```bash
git add task_notice.html
git commit -m "feat: update state model with images array and imageRetentionDays"
```

---

### Task 3: 添加图片压缩和 IndexedDB 操作函数

**Files:**
- Modify: `task_notice.html` `<script>` 部分 — 在 `initImageDB()` 函数之后添加新函数

- [ ] **Step 1: 添加图片压缩函数**

在 `initImageDB()` 函数之后添加：

```javascript
function compressImage(file) {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = () => {
      const img = new Image();
      img.onload = () => {
        let width = img.width;
        let height = img.height;
        if (width > IMAGE_MAX_WIDTH) {
          height = Math.round((height * IMAGE_MAX_WIDTH) / width);
          width = IMAGE_MAX_WIDTH;
        }
        const canvas = document.createElement("canvas");
        canvas.width = width;
        canvas.height = height;
        const ctx = canvas.getContext("2d");
        ctx.drawImage(img, 0, 0, width, height);
        canvas.toBlob(
          (blob) => resolve({ blob, width, height, type: "image/jpeg" }),
          "image/jpeg",
          IMAGE_QUALITY
        );
      };
      img.onerror = reject;
      img.src = reader.result;
    };
    reader.onerror = reject;
    reader.readAsDataURL(file);
  });
}
```

- [ ] **Step 2: 添加 IndexedDB 图片存取函数**

在 `compressImage()` 之后添加：

```javascript
async function saveImageToDB(file) {
  const { blob, width, height, type } = await compressImage(file);
  const id = uid();
  const record = {
    id,
    blob,
    type,
    width,
    height,
    originalName: file.name,
    createdAt: Date.now()
  };
  await imageDB.put(IMAGE_STORE, record);
  return id;
}

async function getImageFromDB(id) {
  return imageDB.get(IMAGE_STORE, id);
}

async function deleteImageFromDB(id) {
  await imageDB.delete(IMAGE_STORE, id);
}

async function getAllImageKeys() {
  return imageDB.getAllKeys(IMAGE_STORE);
}
```

- [ ] **Step 3: 验证函数定义无语法错误**

在浏览器控制台中输入 `typeof compressImage`，应返回 `"function"`。

- [ ] **Step 4: Commit**

```bash
git add task_notice.html
git commit -m "feat: add image compression and IndexedDB CRUD functions"
```

---

### Task 4: 添加图片上传 UI — 模板表单

**Files:**
- Modify: `task_notice.html` — 模板表单 HTML（约第 736 行）
- Modify: `task_notice.html` — CSS 样式
- Modify: `task_notice.html` — `templateForm` submit 处理、`handleTemplateAction` 编辑逻辑

- [ ] **Step 1: 添加图片上传区 CSS**

在 `</style>` 之前添加：

```css
.image-upload-area {
  border: 2px dashed var(--line);
  border-radius: 10px;
  padding: 16px;
  text-align: center;
  cursor: pointer;
  transition: border-color 0.2s, background 0.2s;
  margin-top: 6px;
}

.image-upload-area:hover,
.image-upload-area.dragover {
  border-color: var(--primary);
  background: var(--soft);
}

.image-upload-area input[type="file"] {
  display: none;
}

.image-upload-area .upload-hint {
  color: var(--muted);
  font-size: 13px;
}

.image-upload-area .upload-icon {
  font-size: 24px;
  margin-bottom: 4px;
}

.image-preview-grid {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-top: 8px;
}

.image-preview-grid .thumb {
  width: 64px;
  height: 64px;
  border-radius: 8px;
  object-fit: cover;
  cursor: pointer;
  border: 1px solid var(--line);
}

.image-preview-grid .thumb-wrapper {
  position: relative;
  display: inline-block;
}

.image-preview-grid .thumb-remove {
  position: absolute;
  top: -6px;
  right: -6px;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: var(--danger);
  color: white;
  font-size: 12px;
  line-height: 20px;
  text-align: center;
  cursor: pointer;
  border: none;
  padding: 0;
}

.task-images {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  margin-top: 6px;
}

.task-images .task-thumb {
  width: 48px;
  height: 48px;
  border-radius: 6px;
  object-fit: cover;
  cursor: pointer;
  border: 1px solid var(--line);
}

.fullscreen-viewer {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.9);
  z-index: 1000;
  display: none;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.fullscreen-viewer.active {
  display: flex;
}

.fullscreen-viewer img {
  max-width: 90vw;
  max-height: 85vh;
  object-fit: contain;
}

.fullscreen-viewer .viewer-close {
  position: absolute;
  top: 16px;
  right: 16px;
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.2);
  color: white;
  font-size: 24px;
  line-height: 40px;
  text-align: center;
  cursor: pointer;
  border: none;
}

.fullscreen-viewer .viewer-nav {
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
  width: 48px;
  height: 48px;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.2);
  color: white;
  font-size: 24px;
  line-height: 48px;
  text-align: center;
  cursor: pointer;
  border: none;
}

.fullscreen-viewer .viewer-prev {
  left: 16px;
}

.fullscreen-viewer .viewer-next {
  right: 16px;
}

.fullscreen-viewer .viewer-counter {
  color: rgba(255, 255, 255, 0.7);
  margin-top: 12px;
  font-size: 14px;
}
```

- [ ] **Step 2: 在模板表单中添加图片上传区**

在模板表单的 `备注` textarea 之后、`提交按钮` 之前（约第 736 行后）添加：

```html
<label class="field">
  <span>参考图片</span>
  <div class="image-upload-area" id="templateImageUpload">
    <div class="upload-icon">🖼️</div>
    <div class="upload-hint">点击选择图片，或拖拽图片到此处</div>
    <input type="file" id="templateImageInput" accept="image/*" multiple>
  </div>
  <div class="image-preview-grid" id="templateImagePreview"></div>
</label>
```

- [ ] **Step 3: 添加模板图片上传逻辑**

在 JS 的 `bindEvents()` 函数中添加模板图片上传事件绑定。在 `els.templateRepeat.addEventListener(...)` 之后添加：

```javascript
bindImageUpload("templateImageUpload", "templateImageInput", "templateImagePreview", "templateImages");
```

添加全局变量和通用绑定函数（在 `let editingTemplateId = null;` 之后）：

```javascript
let templateImages = [];
```

在工具函数区域添加：

```javascript
function bindImageUpload(areaId, inputId, previewId, stateKey) {
  const area = document.getElementById(areaId);
  const input = document.getElementById(inputId);
  const preview = document.getElementById(previewId);

  area.addEventListener("click", () => input.click());

  area.addEventListener("dragover", (e) => {
    e.preventDefault();
    area.classList.add("dragover");
  });

  area.addEventListener("dragleave", () => {
    area.classList.remove("dragover");
  });

  area.addEventListener("drop", (e) => {
    e.preventDefault();
    area.classList.remove("dragover");
    handleImageFiles(e.dataTransfer.files, stateKey, preview);
  });

  input.addEventListener("change", () => {
    handleImageFiles(input.files, stateKey, preview);
    input.value = "";
  });

  document.addEventListener("paste", (e) => {
    const files = e.clipboardData?.files;
    if (files && files.length) {
      handleImageFiles(files, stateKey, preview);
    }
  });
}

async function handleImageFiles(files, stateKey, previewEl) {
  for (const file of files) {
    if (!file.type.startsWith("image/")) continue;
    try {
      const id = await saveImageToDB(file);
      if (stateKey === "templateImages") {
        templateImages.push(id);
      }
      renderImagePreview(previewEl, stateKey === "templateImages" ? templateImages : []);
    } catch (err) {
      console.error("Image upload failed:", err);
      toast("图片上传失败");
    }
  }
}

function renderImagePreview(container, imageIds) {
  container.innerHTML = "";
  imageIds.forEach((id, index) => {
    const wrapper = document.createElement("div");
    wrapper.className = "thumb-wrapper";

    const img = document.createElement("img");
    img.className = "thumb";
    img.alt = "预览";
    getImageFromDB(id).then((record) => {
      if (record) img.src = URL.createObjectURL(record.blob);
    });

    const removeBtn = document.createElement("button");
    removeBtn.className = "thumb-remove";
    removeBtn.type = "button";
    removeBtn.textContent = "×";
    removeBtn.addEventListener("click", () => {
      imageIds.splice(index, 1);
      renderImagePreview(container, imageIds);
    });

    wrapper.appendChild(img);
    wrapper.appendChild(removeBtn);
    container.appendChild(wrapper);
  });
}
```

- [ ] **Step 4: 更新模板表单提交逻辑**

修改 `els.templateForm` 的 submit 处理（约第 975 行），在 payload 中加入 images：

```javascript
els.templateForm.addEventListener("submit", (event) => {
  event.preventDefault();
  const payload = readTemplateForm();
  if (!payload.name) return;

  if (editingTemplateId) {
    const template = state.templates.find((item) => item.id === editingTemplateId);
    Object.assign(template, payload);
    template.images = [...templateImages];
    editingTemplateId = null;
    els.saveTemplateBtn.textContent = "添加到每日模板";
    toast("模板已更新");
  } else {
    const template = { id: uid(), ...payload, images: [...templateImages] };
    state.templates.push(template);
    syncTemplateToDay(template, selectedDate);
    toast("已添加到每日模板");
  }

  templateImages = [];
  els.templateForm.reset();
  document.getElementById("templateImagePreview").innerHTML = "";
  updateTemplateWeekdayVisibility();
  saveState();
  renderAll();
});
```

- [ ] **Step 5: 更新模板编辑逻辑加载已有图片**

修改 `handleTemplateAction` 中的 `edit-template` 分支（约第 1376 行），在设置表单值之后加载图片：

```javascript
if (action === "edit-template") {
  editingTemplateId = id;
  els.templateName.value = task.name;
  els.templateCategory.value = task.category;
  els.templateRepeat.value = task.repeat || "daily";
  els.templateWeekday.value = String(task.weekday ?? 1);
  updateTemplateWeekdayVisibility();
  els.templateNote.value = task.note || "";
  templateImages = [...(task.images || [])];
  renderImagePreview(document.getElementById("templateImagePreview"), templateImages);
  els.saveTemplateBtn.textContent = "保存模板修改";
  els.templateName.focus();
}
```

- [ ] **Step 6: 验证模板图片上传功能**

打开页面，添加一个模板并上传图片，确认缩略图显示正常。编辑模板时确认已有图片能加载。

- [ ] **Step 7: Commit**

```bash
git add task_notice.html
git commit -m "feat: add image upload UI to template form"
```

---

### Task 5: 添加图片上传 UI — 每日任务对话框

**Files:**
- Modify: `task_notice.html` — taskDialog HTML（约第 831 行后）
- Modify: `task_notice.html` — `openTaskDialog()`、`taskForm` submit 处理

- [ ] **Step 1: 在 taskDialog 中添加图片上传区**

在 taskDialog 的 `推迟到日期` label 之后、`dialog-actions` 之前（约第 835 行后）添加：

```html
<label class="field">
  <span>打卡图片</span>
  <div class="image-upload-area" id="taskImageUpload">
    <div class="upload-icon">🖼️</div>
    <div class="upload-hint">点击选择图片，或拖拽图片到此处</div>
    <input type="file" id="taskImageInput" accept="image/*" multiple>
  </div>
  <div class="image-preview-grid" id="taskImagePreview"></div>
</label>
```

- [ ] **Step 2: 添加任务图片状态变量**

在 `let templateImages = [];` 之后添加：

```javascript
let taskImages = [];
```

- [ ] **Step 3: 在 bindEvents() 中绑定任务图片上传**

在 `els.cancelDialogBtn.addEventListener(...)` 之后添加：

```javascript
bindImageUpload("taskImageUpload", "taskImageInput", "taskImagePreview", "taskImages");
```

- [ ] **Step 4: 更新 handleImageFiles 支持 taskImages**

修改 `handleImageFiles` 函数，添加 taskImages 分支：

```javascript
async function handleImageFiles(files, stateKey, previewEl) {
  for (const file of files) {
    if (!file.type.startsWith("image/")) continue;
    try {
      const id = await saveImageToDB(file);
      if (stateKey === "templateImages") {
        templateImages.push(id);
      } else if (stateKey === "taskImages") {
        taskImages.push(id);
      }
      renderImagePreview(previewEl, stateKey === "templateImages" ? templateImages : taskImages);
    } catch (err) {
      console.error("Image upload failed:", err);
      toast("图片上传失败");
    }
  }
}
```

- [ ] **Step 5: 更新 openTaskDialog() 加载任务图片**

修改 `openTaskDialog()` 函数（约第 1402 行）：

```javascript
function openTaskDialog(scope, task = null) {
  const fallback = { id: uid(), name: "", category: "学习", note: "", images: [] };
  const data = task || fallback;

  els.dialogTitle.textContent = scope === "template" ? "编辑模板任务" : "编辑当天任务";
  els.editingScope.value = task ? scope : "day-new";
  els.editingId.value = data.id;
  els.taskName.value = data.name;
  els.taskCategory.value = data.category;
  els.taskNote.value = data.note || "";
  els.taskPostponeDate.value = selectedDate;
  taskImages = [...(data.images || [])];
  renderImagePreview(document.getElementById("taskImagePreview"), taskImages);
  els.taskDialog.showModal();
  els.taskName.focus();
}
```

- [ ] **Step 6: 更新 taskForm submit 处理保存图片**

修改 `els.taskForm` 的 submit 处理（约第 999 行），在 payload 中加入 images：

```javascript
els.taskForm.addEventListener("submit", (event) => {
  event.preventDefault();
  const payload = {
    name: els.taskName.value.trim(),
    category: els.taskCategory.value,
    note: els.taskNote.value.trim(),
    images: [...taskImages]
  };
  if (!payload.name) return;

  const scope = els.editingScope.value;
  const id = els.editingId.value;
  const targetDate = els.taskPostponeDate.value;
  if (scope === "template") {
    const template = state.templates.find((item) => item.id === id);
    Object.assign(template, payload);
  } else if (scope === "day-new") {
    const createDate = targetDate || selectedDate;
    ensureDay(createDate);
    state.days[createDate].tasks.push({
      id: uid(),
      done: false,
      createdFromTemplate: false,
      ...payload
    });
    if (createDate !== selectedDate) {
      toast(`任务已添加到 ${createDate}`);
    }
  } else {
    const task = state.days[selectedDate].tasks.find((item) => item.id === id);
    Object.assign(task, payload);
    if (targetDate && targetDate !== selectedDate) {
      postponeTaskToDate(task, targetDate);
      toast(`任务已推迟到 ${targetDate}`);
    }
  }
  taskImages = [];
  saveState();
  els.taskDialog.close();
  renderAll();
});
```

- [ ] **Step 7: 验证每日任务图片上传功能**

打开任务编辑对话框，上传图片，确认保存后图片 ID 被正确关联。

- [ ] **Step 8: Commit**

```bash
git add task_notice.html
git commit -m "feat: add image upload UI to daily task dialog"
```

---

### Task 6: 添加缩略图展示 — 任务卡片和模板列表

**Files:**
- Modify: `task_notice.html` — `renderToday()` 函数
- Modify: `task_notice.html` — `renderTemplates()` 函数

- [ ] **Step 1: 添加任务卡片缩略图渲染函数**

在工具函数区域添加：

```javascript
function renderTaskImages(imageIds, containerClass) {
  if (!imageIds || !imageIds.length) return "";
  const thumbsHtml = imageIds.map((id) => `
    <img class="task-thumb" data-img-id="${id}" alt="任务图片" loading="lazy">
  `).join("");
  return `<div class="${containerClass}">${thumbsHtml}</div>`;
}

async function loadTaskThumbnails() {
  document.querySelectorAll(".task-thumb[data-img-id]").forEach(async (img) => {
    if (img.src) return;
    const id = img.dataset.imgId;
    try {
      const record = await getImageFromDB(id);
      if (record) img.src = URL.createObjectURL(record.blob);
    } catch (err) {
      console.warn("Failed to load thumbnail:", err);
    }
  });
}
```

- [ ] **Step 2: 更新 renderToday() 显示任务图片**

修改 `renderToday()` 函数（约第 1221 行）中的任务卡片 HTML，在 `task-meta` div 之后添加图片展示：

```javascript
els.taskList.innerHTML = day.tasks.map((task) => `
  <article class="task ${task.done ? "done" : ""}">
    <button class="check" type="button" data-action="toggle" data-id="${task.id}" aria-label="切换完成状态">${task.done ? "✓" : ""}</button>
    <div class="task-main">
      <div class="task-name">${escapeHtml(task.name)}</div>
      <div class="task-meta">
        <span class="tag">${escapeHtml(task.category)}</span>
        <span class="tag">${repeatLabel(task)}</span>
        ${task.postponedFrom ? `<span class="tag">由 ${escapeHtml(task.postponedFrom)} 推迟</span>` : ""}
        ${task.note ? `<span>${escapeHtml(task.note)}</span>` : ""}
      </div>
      ${renderTaskImages(task.images, "task-images")}
    </div>
    <div class="task-actions">
      <button class="icon-btn" type="button" data-action="edit" data-id="${task.id}" aria-label="编辑">✎</button>
      <button class="icon-btn" type="button" data-action="delete" data-id="${task.id}" aria-label="删除">×</button>
    </div>
  </article>
`).join("");
```

在 `renderToday()` 函数末尾、事件绑定之后添加：

```javascript
loadTaskThumbnails();
```

- [ ] **Step 3: 更新 renderTemplates() 显示模板图片**

修改 `renderTemplates()` 函数（约第 1251 行）中的模板卡片 HTML：

```javascript
els.templateList.innerHTML = state.templates.map((task) => `
  <div class="history-item">
    <div>
      <strong>${escapeHtml(task.name)}</strong>
      <div class="task-meta">
        <span class="tag">${escapeHtml(task.category)}</span>
        <span class="tag">${repeatLabel(task)}</span>
        ${task.note ? `<span>${escapeHtml(task.note)}</span>` : ""}
      </div>
      ${renderTaskImages(task.images, "task-images")}
    </div>
    <div class="task-actions">
      <button class="icon-btn" type="button" data-action="edit-template" data-id="${task.id}" aria-label="编辑模板">✎</button>
      <button class="icon-btn" type="button" data-action="delete-template" data-id="${task.id}" aria-label="删除模板">×</button>
    </div>
  </div>
`).join("");
```

在 `renderTemplates()` 函数末尾、事件绑定之后添加：

```javascript
loadTaskThumbnails();
```

- [ ] **Step 4: 验证缩略图显示**

创建一个带图片的任务，确认任务卡片和模板列表中显示缩略图。

- [ ] **Step 5: Commit**

```bash
git add task_notice.html
git commit -m "feat: display image thumbnails in task cards and template list"
```

---

### Task 7: 添加全屏图片查看器

**Files:**
- Modify: `task_notice.html` — HTML body（约第 844 行后）
- Modify: `task_notice.html` — JS 事件绑定和查看器函数

- [ ] **Step 1: 添加全屏查看器 HTML**

在 `<div class="toast" id="toast"></div>` 之后、`<script>` 之前添加：

```html
<div class="fullscreen-viewer" id="fullscreenViewer">
  <button class="viewer-close" id="viewerClose" type="button" aria-label="关闭">×</button>
  <button class="viewer-nav viewer-prev" id="viewerPrev" type="button" aria-label="上一张">‹</button>
  <img id="viewerImage" alt="图片查看">
  <button class="viewer-nav viewer-next" id="viewerNext" type="button" aria-label="下一张">›</button>
  <div class="viewer-counter" id="viewerCounter"></div>
</div>
```

- [ ] **Step 2: 添加查看器元素引用和状态变量**

在 `els` 对象中添加（约第 917 行后）：

```javascript
fullscreenViewer: document.querySelector("#fullscreenViewer"),
viewerImage: document.querySelector("#viewerImage"),
viewerClose: document.querySelector("#viewerClose"),
viewerPrev: document.querySelector("#viewerPrev"),
viewerNext: document.querySelector("#viewerNext"),
viewerCounter: document.querySelector("#viewerCounter"),
```

在全局变量区域添加：

```javascript
let viewerImages = [];
let viewerIndex = 0;
```

- [ ] **Step 3: 添加查看器函数**

在工具函数区域添加：

```javascript
async function openViewer(imageIds, startIndex = 0) {
  viewerImages = imageIds;
  viewerIndex = startIndex;
  await showViewerImage();
  els.fullscreenViewer.classList.add("active");
}

async function showViewerImage() {
  const id = viewerImages[viewerIndex];
  const record = await getImageFromDB(id);
  if (record) {
    els.viewerImage.src = URL.createObjectURL(record.blob);
  }
  els.viewerCounter.textContent = `${viewerIndex + 1} / ${viewerImages.length}`;
  els.viewerPrev.style.display = viewerImages.length > 1 ? "block" : "none";
  els.viewerNext.style.display = viewerImages.length > 1 ? "block" : "none";
}

function closeViewer() {
  els.fullscreenViewer.classList.remove("active");
  els.viewerImage.src = "";
  viewerImages = [];
  viewerIndex = 0;
}
```

- [ ] **Step 4: 绑定查看器事件**

在 `bindEvents()` 函数末尾添加：

```javascript
els.viewerClose.addEventListener("click", closeViewer);
els.viewerPrev.addEventListener("click", async () => {
  viewerIndex = (viewerIndex - 1 + viewerImages.length) % viewerImages.length;
  await showViewerImage();
});
els.viewerNext.addEventListener("click", async () => {
  viewerIndex = (viewerIndex + 1) % viewerImages.length;
  await showViewerImage();
});
els.fullscreenViewer.addEventListener("click", (e) => {
  if (e.target === els.fullscreenViewer) closeViewer();
});
document.addEventListener("keydown", (e) => {
  if (!els.fullscreenViewer.classList.contains("active")) return;
  if (e.key === "Escape") closeViewer();
  if (e.key === "ArrowLeft") els.viewerPrev.click();
  if (e.key === "ArrowRight") els.viewerNext.click();
});
```

- [ ] **Step 5: 绑定缩略图点击事件打开查看器**

修改 `loadTaskThumbnails()` 函数，在加载图片后绑定点击事件：

```javascript
async function loadTaskThumbnails() {
  document.querySelectorAll(".task-thumb[data-img-id]").forEach(async (img) => {
    if (img.src) return;
    const id = img.dataset.imgId;
    try {
      const record = await getImageFromDB(id);
      if (record) img.src = URL.createObjectURL(record.blob);
    } catch (err) {
      console.warn("Failed to load thumbnail:", err);
    }
  });

  document.querySelectorAll(".task-images").forEach((container) => {
    if (container.dataset.bound) return;
    container.dataset.bound = "true";
    container.addEventListener("click", (e) => {
      const thumb = e.target.closest(".task-thumb");
      if (!thumb) return;
      const allIds = [...container.querySelectorAll(".task-thumb")].map((t) => t.dataset.imgId);
      const index = allIds.indexOf(thumb.dataset.imgId);
      openViewer(allIds, index);
    });
  });
}
```

- [ ] **Step 6: 验证全屏查看器功能**

点击任务卡片中的缩略图，确认全屏查看器打开。测试左右切换、ESC 关闭、点击背景关闭。

- [ ] **Step 7: Commit**

```bash
git add task_notice.html
git commit -m "feat: add fullscreen image viewer with navigation"
```

---

### Task 8: 添加启动清理机制

**Files:**
- Modify: `task_notice.html` — 新增 `cleanupExpiredImages()` 函数

- [ ] **Step 1: 添加清理函数**

在工具函数区域添加：

```javascript
async function cleanupExpiredImages() {
  if (!imageDB) return;
  try {
    const retentionDays = state.settings.imageRetentionDays || 180;
    const cutoff = Date.now() - retentionDays * 86400000;
    const allRecords = await imageDB.getAll(IMAGE_STORE);
    let cleaned = 0;

    for (const record of allRecords) {
      if (record.createdAt < cutoff) {
        await deleteImageFromDB(record.id);
        removeImageReference(record.id);
        cleaned++;
      }
    }

    if (cleaned > 0) {
      saveState();
      console.log(`Cleaned up ${cleaned} expired images`);
    }
  } catch (err) {
    console.warn("Image cleanup failed:", err);
  }
}

function removeImageReference(imageId) {
  state.templates.forEach((t) => {
    if (t.images) t.images = t.images.filter((id) => id !== imageId);
  });
  Object.values(state.days).forEach((day) => {
    day.tasks.forEach((t) => {
      if (t.images) t.images = t.images.filter((id) => id !== imageId);
    });
  });
}
```

- [ ] **Step 2: 验证清理逻辑**

手动在 IndexedDB 中插入一条 createdAt 为很久以前的记录，重新加载页面，确认该记录被清除。

- [ ] **Step 3: Commit**

```bash
git add task_notice.html
git commit -m "feat: add auto-cleanup of expired images on app startup"
```

---

### Task 9: 添加设置面板 — 图片保留天数

**Files:**
- Modify: `task_notice.html` — settingsPanel HTML
- Modify: `task_notice.html` — `settingsForm` submit 处理、`renderSettings()`

- [ ] **Step 1: 在设置表单中添加图片保留天数输入框**

在 settingsPanel 的 `座右铭` textarea 之后、提交按钮之前（约第 763 行后）添加：

```html
<label class="field">
  <span>图片保留天数</span>
  <input id="imageRetentionDays" type="number" min="1" max="3650" placeholder="180">
  <small style="color: var(--muted); font-size: 12px;">超过此天数的图片将在下次打开时自动清除，默认 180 天（约 6 个月）</small>
</label>
```

- [ ] **Step 2: 添加元素引用**

在 `els` 对象中添加：

```javascript
imageRetentionDays: document.querySelector("#imageRetentionDays"),
```

- [ ] **Step 3: 更新 renderSettings() 填充保留天数**

修改 `renderSettings()` 函数（约第 1190 行），在 `els.mottoInput.value = motto;` 之后添加：

```javascript
els.imageRetentionDays.value = state.settings.imageRetentionDays || 180;
```

- [ ] **Step 4: 更新 settingsForm submit 保存保留天数**

修改 `els.settingsForm` 的 submit 处理（约第 964 行）：

```javascript
els.settingsForm.addEventListener("submit", (event) => {
  event.preventDefault();
  state.settings = {
    ownerName: els.ownerName.value.trim(),
    motto: els.mottoInput.value.trim() || defaultSettings.motto,
    imageRetentionDays: Math.max(1, Number(els.imageRetentionDays.value) || 180)
  };
  saveState();
  renderSettings();
  toast("个性设置已保存");
});
```

- [ ] **Step 5: 验证设置保存和加载**

修改图片保留天数，保存后刷新页面，确认值被正确保留。

- [ ] **Step 6: Commit**

```bash
git add task_notice.html
git commit -m "feat: add image retention days setting to settings panel"
```

---

### Task 10: 处理任务删除时的图片清理

**Files:**
- Modify: `task_notice.html` — `handleDayAction()`、`handleTemplateAction()`

- [ ] **Step 1: 更新 handleDayAction 删除逻辑**

修改 `handleDayAction()` 函数（约第 1338 行）中的删除分支，在删除任务时也删除关联图片：

```javascript
if (action === "delete" && confirm("删除这一天的任务？")) {
  const removedTask = tasks.find((item) => item.id === id);
  if (removedTask && removedTask.images) {
    removedTask.images.forEach((imgId) => deleteImageFromDB(imgId));
  }
  state.days[selectedDate].tasks = tasks.filter((item) => item.id !== id);
  toast("当天任务已删除");
}
```

- [ ] **Step 2: 更新 handleTemplateAction 删除逻辑**

修改 `handleTemplateAction()` 函数（约第 1388 行）中的删除分支：

```javascript
if (action === "delete-template" && confirm("删除这个每日模板？历史记录不会被删除。")) {
  const removedTemplate = state.templates.find((item) => item.id === id);
  if (removedTemplate && removedTemplate.images) {
    removedTemplate.images.forEach((imgId) => deleteImageFromDB(imgId));
  }
  state.templates = state.templates.filter((item) => item.id !== id);
  if (editingTemplateId === id) {
    editingTemplateId = null;
    templateImages = [];
    els.templateForm.reset();
    document.getElementById("templateImagePreview").innerHTML = "";
    updateTemplateWeekdayVisibility();
    els.saveTemplateBtn.textContent = "添加到每日模板";
  }
  saveState();
  renderAll();
  toast("模板已删除");
}
```

- [ ] **Step 3: 验证删除功能**

删除一个带图片的任务，确认 IndexedDB 中对应的图片记录也被清除。

- [ ] **Step 4: Commit**

```bash
git add task_notice.html
git commit -m "feat: clean up images when deleting tasks or templates"
```

---

### Task 11: 更新用户指南

**Files:**
- Modify: `task_notice.html` — guidePanel HTML

- [ ] **Step 1: 在用户指南中添加图片功能说明**

在 guidePanel 的 `历史与数据` section 之后（约第 795 行后）添加：

```html
<div class="guide-section">
  <h3>图片功能</h3>
  <ul>
    <li>在模板或每日任务中可以上传参考图片或打卡照片。</li>
    <li>支持点击选择、拖拽上传、粘贴上传三种方式。</li>
    <li>图片会自动压缩以节省存储空间。</li>
    <li>点击缩略图可以全屏查看大图，支持左右切换。</li>
    <li>在「个性设置」中可以设置图片保留天数，默认 6 个月。</li>
    <li>超过保留时间的图片会在打开页面时自动清除。</li>
  </ul>
</div>
```

- [ ] **Step  Commit**

```bash
git add task_notice.html
git commit -m "docs: add image feature guide to user guide panel"
```

---

### Task 12: 最终集成测试

**Files:** 无新文件修改

- [ ] **Step 1: 完整功能测试**

在浏览器中打开 `task_notice.html`，执行以下测试：

1. 添加模板并上传图片 → 确认缩略图显示
2. 编辑模板 → 确认已有图片加载
3. 删除模板 → 确认图片被清理
4. 打开每日任务对话框 → 上传图片 → 确认保存
5. 点击缩略图 → 确认全屏查看器打开
6. 全屏查看器中左右切换、ESC 关闭
7. 拖拽上传图片
8. 粘贴上传图片
9. 修改图片保留天数 → 保存 → 刷新确认
10. 导出数据 → 确认 JSON 中包含 images 字段
11. 导入数据 → 确认 images 字段被保留

- [ ] **Step 2: 最终 Commit（如有修复）**

```bash
git add task_notice.html
git commit -m "fix: address integration test issues"
```
