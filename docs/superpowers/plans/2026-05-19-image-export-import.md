# 图片导出/导入功能实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 支持将 IndexedDB 中的图片一并导出为 ZIP 包，导入时自动还原，支持按时间范围筛选导出。

**Architecture:** 在现有导出/导入逻辑基础上，引入 JSZip 库，新增导出对话框（含时间范围选择），导出时将图片从 IndexedDB 读取并打包到 ZIP 的 images/ 文件夹中，导入时自动判断文件格式（ZIP 或 JSON）并走对应流程。

**Tech Stack:** JSZip（CDN），IndexedDB（已引入的 idb 库）

**Files:**
- Modify: `task_notice.html`（唯一文件，所有改动都在此）

---

### Task 1: 引入 JSZip CDN

**Files:**
- Modify: `task_notice.html:7`（`<head>` 区域）

- [ ] **Step 1: 在 `<head>` 中添加 JSZip CDN 链接**

在第 7 行 `idb` 的 `<script>` 标签后面，添加 JSZip 的 CDN 链接：

```html
<script src="https://cdn.jsdelivr.net/npm/idb@8/build/umd.js"></script>
<script src="https://cdn.jsdelivr.net/npm/jszip@3/dist/jszip.min.js"></script>
```

- [ ] **Step 2: 验证 JSZip 加载**

在浏览器打开页面，控制台输入 `JSZip`，确认返回函数对象而非 `undefined`。

- [ ] **Step 3: Commit**

```bash
git add task_notice.html
git commit -m "feat: add JSZip CDN for ZIP export/import"
```

---

### Task 2: 添加导出对话框 HTML

**Files:**
- Modify: `task_notice.html`（在 `<dialog id="taskDialog">` 之前，约第 979 行前插入）

- [ ] **Step 1: 添加导出对话框 HTML**

在 `<dialog id="taskDialog">` 之前插入以下 HTML：

```html
<dialog id="exportDialog">
  <form method="dialog">
    <div class="dialog-head">
      <h3>导出数据</h3>
      <button class="icon-btn" value="cancel" type="submit" aria-label="关闭">×</button>
    </div>
  </form>
  <div class="dialog-body">
    <form class="form" id="exportForm">
      <label class="field">
        <span>起始日期（留空表示不限）</span>
        <input id="exportStartDate" type="date">
      </label>
      <label class="field">
        <span>结束日期（留空表示不限）</span>
        <input id="exportEndDate" type="date">
      </label>
      <div class="dialog-actions">
        <button class="btn" type="button" id="cancelExportBtn">取消</button>
        <button class="btn primary" type="submit">导出</button>
      </div>
    </form>
  </div>
</dialog>
```

- [ ] **Step 2: Commit**

```bash
git add task_notice.html
git commit -m "feat: add export dialog HTML with date range inputs"
```

---

### Task 3: 绑定导出对话框事件

**Files:**
- Modify: `task_notice.html`（`els` 对象，约第 1365 行；`bindEvents()` 函数，约第 1450 行）

- [ ] **Step 1: 在 `els` 对象中添加导出对话框相关元素引用**

在 `els` 对象中（`importFileInput` 那行之后）添加：

```javascript
      exportDialog: document.querySelector("#exportDialog"),
      exportForm: document.querySelector("#exportForm"),
      exportStartDate: document.querySelector("#exportStartDate"),
      exportEndDate: document.querySelector("#exportEndDate"),
      cancelExportBtn: document.querySelector("#cancelExportBtn"),
```

- [ ] **Step 2: 修改导出按钮的事件绑定**

将 `bindEvents()` 中的：

```javascript
      els.exportDataBtn.addEventListener("click", exportData);
```

改为：

```javascript
      els.exportDataBtn.addEventListener("click", openExportDialog);
```

- [ ] **Step 3: 添加导出对话框事件绑定**

在 `bindEvents()` 中（上一步修改的代码后面）添加：

```javascript
      els.cancelExportBtn.addEventListener("click", () => els.exportDialog.close());
      els.exportForm.addEventListener("submit", (e) => {
        e.preventDefault();
        els.exportDialog.close();
        exportData();
      });
```

- [ ] **Step 4: 添加 `openExportDialog` 函数**

在 `exportData()` 函数之前添加：

```javascript
    function openExportDialog() {
      els.exportStartDate.value = "";
      els.exportEndDate.value = "";
      els.exportDialog.showModal();
    }
```

- [ ] **Step 5: Commit**

```bash
git add task_notice.html
git commit -m "feat: bind export dialog events and open handler"
```

---

### Task 4: 实现 ZIP 导出逻辑

**Files:**
- Modify: `task_notice.html`（替换 `exportData()` 函数，约第 1615-1632 行）

- [ ] **Step 1: 添加辅助函数 `sanitizeFileName`**

在 `exportData()` 函数之前添加：

```javascript
    function sanitizeFileName(name) {
      return name.replace(/[\\/:*?"<>|]/g, "_");
    }
```

- [ ] **Step 2: 添加辅助函数 `buildImageFileName`**

在 `sanitizeFileName` 函数之后添加：

```javascript
    function buildImageFileName(imageId, taskName, date) {
      const safeName = sanitizeFileName(taskName || "未命名");
      if (date) {
        return `${imageId}_${safeName}_${date}.jpg`;
      }
      return `${imageId}_${safeName}.jpg`;
    }
```

- [ ] **Step 3: 添加辅助函数 `collectExportImageIds`**

在 `buildImageFileName` 函数之后添加：

```javascript
    function collectExportImageIds(filteredDays) {
      const ids = new Set();
      // 模板图片始终包含
      state.templates.forEach((t) => {
        (t.images || []).forEach((id) => ids.add(id));
      });
      // 每日任务图片：只包含筛选后的 days
      Object.values(filteredDays).forEach((day) => {
        (day.tasks || []).forEach((t) => {
          (t.images || []).forEach((id) => ids.add(id));
        });
      });
      return ids;
    }
```

- [ ] **Step 4: 添加辅助函数 `buildImageIdToNameMap`**

在 `collectExportImageIds` 函数之后添加：

```javascript
    function buildImageIdToNameMap(filteredDays) {
      const map = new Map();
      // 模板图片
      state.templates.forEach((t) => {
        (t.images || []).forEach((id) => {
          if (!map.has(id)) map.set(id, buildImageFileName(id, t.name, null));
        });
      });
      // 每日任务图片
      Object.entries(filteredDays).forEach(([date, day]) => {
        (day.tasks || []).forEach((t) => {
          (t.images || []).forEach((id) => {
            if (!map.has(id)) map.set(id, buildImageFileName(id, t.name, date));
          });
        });
      });
      return map;
    }
```

- [ ] **Step 5: 替换 `exportData()` 函数**

将现有的 `exportData()` 函数（约第 1615-1632 行）完全替换为：

```javascript
    async function exportData() {
      const startDate = els.exportStartDate.value;
      const endDate = els.exportEndDate.value;

      // 按时间范围筛选 days
      const filteredDays = {};
      Object.entries(state.days).forEach(([date, day]) => {
        if (startDate && date < startDate) return;
        if (endDate && date > endDate) return;
        filteredDays[date] = day;
      });

      // 检查是否有图片
      const imageIds = collectExportImageIds(filteredDays);
      const hasImages = imageIds.size > 0;

      if (!hasImages) {
        // 无图片，走现有 JSON 导出逻辑
        const payload = {
          app: "daily-checkin",
          version: 1,
          exportedAt: new Date().toISOString(),
          data: state
        };
        const blob = new Blob([JSON.stringify(payload, null, 2)], { type: "application/json" });
        const url = URL.createObjectURL(blob);
        const link = document.createElement("a");
        link.href = url;
        link.download = `日程打卡数据-${formatDate(new Date())}.json`;
        document.body.appendChild(link);
        link.click();
        link.remove();
        URL.revokeObjectURL(url);
        toast("数据已导出");
        return;
      }

      // 有图片，生成 ZIP
      toast("正在打包导出...");
      const zip = new JSZip();
      const imageIdToName = buildImageIdToNameMap(filteredDays);

      // 构建 state 副本，将 images 中的 ID 替换为文件名
      const exportState = {
        templates: state.templates.map((t) => ({
          ...t,
          images: (t.images || []).map((id) => imageIdToName.get(id) || id)
        })),
        days: Object.fromEntries(
          Object.entries(filteredDays).map(([date, day]) => [
            date,
            {
              ...day,
              tasks: (day.tasks || []).map((t) => ({
                ...t,
                images: (t.images || []).map((id) => imageIdToName.get(id) || id)
              }))
            }
          ])
        ),
        settings: state.settings
      };

      const dataJson = {
        app: "daily-checkin",
        version: 2,
        exportedAt: new Date().toISOString(),
        hasImages: true,
        data: exportState
      };

      zip.file("data.json", JSON.stringify(dataJson, null, 2));

      // 逐个从 IndexedDB 读取图片添加到 ZIP
      const imgFolder = zip.folder("images");
      for (const id of imageIds) {
        try {
          const record = await getImageFromDB(id);
          if (record) {
            const fileName = imageIdToName.get(id) || `${id}.jpg`;
            imgFolder.file(fileName, record.blob);
          }
        } catch (err) {
          console.warn("导出图片失败，已跳过:", id, err);
        }
      }

      // 生成 ZIP 并下载
      const zipBlob = await zip.generateAsync({ type: "blob" });
      const url = URL.createObjectURL(zipBlob);
      const link = document.createElement("a");
      link.href = url;
      link.download = `日程打卡数据-${formatDate(new Date())}.zip`;
      document.body.appendChild(link);
      link.click();
      link.remove();
      URL.revokeObjectURL(url);
      toast("数据已导出");
    }
```

- [ ] **Step 6: 在浏览器中验证导出功能**

打开页面，创建几个任务并上传图片，点击「导出数据」，选择时间范围后导出。确认：
- 无图片时下载 .json 文件
- 有图片时下载 .zip 文件
- ZIP 中包含 data.json 和 images/ 文件夹
- 图片文件名格式正确（ID_任务名_日期.jpg）

- [ ] **Step 7: Commit**

```bash
git add task_notice.html
git commit -m "feat: implement ZIP export with image packaging"
```

---

### Task 5: 实现 ZIP 导入逻辑

**Files:**
- Modify: `task_notice.html`（替换 `importData()` 函数，约第 1634-1661 行；修改文件选择器 `accept` 属性）

- [ ] **Step 1: 修改文件选择器的 accept 属性**

在 HTML 中找到 `<input id="importFileInput" type="file" accept="application/json,.json" hidden>`，将 `accept` 改为：

```html
            <input id="importFileInput" type="file" accept=".zip,.json" hidden>
```

- [ ] **Step 2: 添加 `importZipData` 函数**

在 `importData()` 函数之前添加：

```javascript
    async function importZipData(file) {
      const zip = await JSZip.loadAsync(file);

      // 读取 data.json
      const dataFile = zip.file("data.json");
      if (!dataFile) {
        toast("导入失败：ZIP 中缺少 data.json");
        return;
      }
      const jsonText = await dataFile.async("string");
      const parsed = JSON.parse(jsonText);
      const importedState = normalizeImportedState(parsed);
      if (!importedState) {
        toast("导入失败：文件格式不正确");
        return;
      }

      if (!confirm("导入后会替换当前浏览器里的所有打卡数据，继续吗？")) return;

      // 还原图片到 IndexedDB
      const imgFolder = zip.folder("images");
      if (imgFolder) {
        const fileNames = [];
        imgFolder.forEach((relativePath) => {
          if (relativePath && !relativePath.endsWith("/")) {
            fileNames.push(relativePath);
          }
        });
        for (const fileName of fileNames) {
          try {
            // 从文件名提取图片 ID（第一个 _ 前的部分）
            const id = fileName.split("_")[0];
            if (!id) continue;
            const fileData = imgFolder.file(fileName);
            if (!fileData) continue;
            const blob = await fileData.async("blob");
            if (!blob) continue;
            await imageDB.put(IMAGE_STORE, {
              id,
              blob,
              type: blob.type || "image/jpeg",
              width: 0,
              height: 0,
              originalName: fileName,
              createdAt: Date.now()
            });
          } catch (err) {
            console.warn("导入图片失败，已跳过:", fileName, err);
          }
        }
      }

      // 将 images 数组中的文件名转回纯 ID
      // 从文件名提取 ID 的逻辑：取第一个 _ 前的部分
      const fileNameToId = (fileName) => {
        const id = fileName.split("_")[0];
        return id || fileName;
      };

      importedState.templates.forEach((t) => {
        t.images = (t.images || []).map(fileNameToId);
      });
      Object.values(importedState.days).forEach((day) => {
        (day.tasks || []).forEach((t) => {
          t.images = (t.images || []).map(fileNameToId);
        });
      });

      // 覆盖 state
      state.templates = importedState.templates;
      state.days = importedState.days;
      saveState();
      ensureDay(selectedDate);
      renderAll();
      toast("数据导入成功");
    }
```

- [ ] **Step 3: 修改 `importData` 函数支持格式判断**

将现有的 `importData()` 函数替换为：

```javascript
    async function importData(event) {
      const file = event.target.files?.[0];
      if (!file) return;

      const fileName = file.name.toLowerCase();
      if (fileName.endsWith(".zip")) {
        try {
          await importZipData(file);
        } catch (error) {
          console.error("ZIP import failed", error);
          toast("导入失败：无法解析 ZIP 文件");
        }
        return;
      }

      // 旧格式 JSON 导入
      const reader = new FileReader();
      reader.addEventListener("load", () => {
        try {
          const parsed = JSON.parse(String(reader.result || "{}"));
          const importedState = normalizeImportedState(parsed);
          if (!importedState) {
            toast("导入失败：文件格式不正确");
            return;
          }
          if (!confirm("导入后会替换当前浏览器里的所有打卡数据，继续吗？")) return;

          state.templates = importedState.templates;
          state.days = importedState.days;
          saveState();
          ensureDay(selectedDate);
          renderAll();
          toast("数据导入成功");
        } catch (error) {
          console.error("Import failed", error);
          toast("导入失败：无法读取 JSON");
        }
      });
      reader.readAsText(file, "utf-8");
    }
```

- [ ] **Step 4: 在浏览器中验证导入功能**

1. 先用 Task 4 导出一个 ZIP 文件
2. 清除浏览器数据（或换一个浏览器）
3. 导入刚才导出的 ZIP 文件
4. 确认：图片正确显示，任务数据完整
5. 再测试导入旧格式 JSON 文件，确认走旧逻辑

- [ ] **Step 5: Commit**

```bash
git add task_notice.html
git commit -m "feat: implement ZIP import with image restoration"
```

---

### Task 6: 验证完整流程

- [ ] **Step 1: 端到端测试**

在浏览器中完成以下测试：
1. 创建多个模板任务，部分上传图片
2. 在不同日期打卡，部分上传打卡图片
3. 点击「导出数据」，选择全部时间范围，导出 ZIP
4. 解压 ZIP，确认 data.json 和 images/ 文件夹内容正确
5. 在文件管理器中浏览 images/ 文件夹，确认图片可直接查看
6. 选择部分时间范围导出，确认只包含该范围内的数据和图片
7. 清除浏览器数据，导入 ZIP，确认数据和图片完整恢复
8. 导入旧格式 JSON 文件，确认向后兼容

- [ ] **Step 2: 边界情况测试**

1. 无图片时导出 → 应下载 .json 文件
2. 导出后再次导入相同 ZIP → 应无变化（upsert 语义）
3. 导入损坏的 ZIP → 应显示错误提示

- [ ] **Step 3: Commit（如有修复）**

```bash
git add task_notice.html
git commit -m "fix: address issues found during end-to-end testing"
```
