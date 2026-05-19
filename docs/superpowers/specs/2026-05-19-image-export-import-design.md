# 图片导出/导入功能设计文档

## 概述

为「日程打卡」应用增加完整的数据导出/导入能力，支持将 IndexedDB 中的图片二进制数据一并导出为 ZIP 包，导入时自动还原图片。保持对旧格式 JSON 导入的向后兼容。

## 架构设计

### 存储分层（不变）

- **localStorage**：应用状态（state），包含模板、日期记录、设置、图片 ID 引用
- **IndexedDB**：图片二进制数据（Blob）

### 导出数据流

```
用户点击「导出数据」
    ↓
创建 JSZip 实例
    ↓
构建 data.json（state + 图片文件名映射）
    ↓
遍历所有图片 ID，从 IndexedDB 读取 Blob
    ↓
按命名规则生成文件名，添加到 ZIP 的 images/ 文件夹
    ↓
生成 ZIP 文件并触发下载
```

### 导入数据流

```
用户选择文件（ZIP 或 JSON）
    ↓
判断文件类型
    ↓
如果是 ZIP：
  解压 → 读取 data.json → 将 images/ 中的图片写入 IndexedDB → 加载 state
如果是 JSON（旧格式）：
  走现有导入逻辑（图片需重新上传）
```

## 导出格式

### ZIP 包结构

```
日程打卡数据-2026-05-19.zip
├── data.json
└── images/
    ├── a1b2c3d4_阅读30分钟_2026-05-19.jpg
    ├── e5f6g7h8_查看JIRA看板.jpg
    └── ...
```

### data.json 格式

```json
{
  "app": "daily-checkin",
  "version": 2,
  "exportedAt": "2026-05-19T10:30:00.000Z",
  "hasImages": true,
  "data": {
    "templates": [
      {
        "id": "uuid",
        "name": "任务名称",
        "category": "分类",
        "repeat": "daily",
        "note": "备注",
        "images": ["a1b2c3d4_阅读30分钟.jpg"]
      }
    ],
    "days": {
      "2026-05-19": {
        "tasks": [
          {
            "templateId": "uuid",
            "done": true,
            "images": ["e5f6g7h8_查看JIRA看板_2026-05-19.jpg"]
          }
        ]
      }
    },
    "settings": { ... }
  }
}
```

**与旧格式的差异：**
- `version` 从 1 改为 2
- 新增 `hasImages: true` 字段标识包含图片
- `images` 数组中的值从纯 ID 改为文件名（`{id}_{任务名}_{日期}.jpg`）

## 图片文件命名规则

### 命名格式

- **模板图片**：`{图片ID}_{模板名称}.jpg`
- **每日任务图片**：`{图片ID}_{任务名}_{日期}.jpg`

### 特殊字符处理

文件名中不允许的字符（`/ \ : * ? " < > |`）替换为下划线 `_`。

### 示例

| 来源 | ID | 任务名 | 日期 | 文件名 |
|------|-----|--------|------|--------|
| 模板 | a1b2c3d4 | 阅读30分钟 | - | `a1b2c3d4_阅读30分钟.jpg` |
| 每日任务 | e5f6g7h8 | 查看JIRA看板 | 2026-05-19 | `e5f6g7h8_查看JIRA看板_2026-05-19.jpg` |

### 导入时的匹配逻辑

从文件名中提取第一个 `_` 前的部分作为图片 ID。例如：
- `a1b2c3d4_阅读30分钟.jpg` → ID = `a1b2c3d4`
- `e5f6g7h8_查看JIRA看板_2026-05-19.jpg` → ID = `e5f6g7h8`

然后与 JSON 中 `images` 数组的文件名进行匹配。

## 导入兼容性

### ZIP 导入（新格式）

1. 使用 JSZip 解压
2. 读取 `data.json`，解析 state
3. 遍历 `images/` 文件夹中的文件
4. 从文件名提取图片 ID
5. 读取文件为 Blob，写入 IndexedDB
6. 更新 state 中的图片引用（将文件名转回纯 ID）
7. 保存 state 到 localStorage

### JSON 导入（旧格式，向后兼容）

保持现有逻辑不变：
- 读取 JSON 文件
- 图片 ID 引用被保留
- 对应的图片数据需要用户重新上传

### 导入时的文件选择

使用 `<input type="file" accept=".zip,.json">` 让用户选择单个文件。根据文件扩展名自动判断格式：
- `.zip` → ZIP 导入流程
- `.json` → 旧格式 JSON 导入流程

## 技术依赖

- **JSZip**（~10KB）：ZIP 文件创建和解压库，通过 CDN 引入
  - CDN: `https://cdn.jsdelivr.net/npm/jszip@3/dist/jszip.min.js`
- **idb**（已引入）：IndexedDB 的 Promise 封装库

## 用户交互

### 导出按钮

保持现有的「导出数据」按钮不变。点击后：
1. 如果没有图片：走现有逻辑，下载 JSON 文件
2. 如果有图片：生成 ZIP 包并下载

### 导入按钮

保持现有的「导入数据」按钮不变。点击后：
1. 打开文件选择器（accept=".zip,.json"）
2. 根据文件类型自动走对应流程
3. 导入完成后显示 toast 提示

## 边界情况

### 图片不存在

如果 JSON 中引用的某个图片 ID 在 IndexedDB（导出时）或 ZIP 的 images/ 文件夹（导入时）中不存在：
- 导出时：跳过该图片，JSON 中保留文件名引用
- 导入时：跳过该图片，保留 ID 引用（图片显示为加载失败）

### 文件名冲突

由于使用图片 ID（UUID）作为文件名前缀，理论上不会冲突。即使任务名相同，ID 也不同。

### 大文件处理

- 导出时：逐个从 IndexedDB 读取 Blob，添加到 ZIP，避免一次性加载所有图片到内存
- 导入时：逐个从 ZIP 中读取图片，写入 IndexedDB

### 空图片列表

如果没有图片，导出的 ZIP 中只有 `data.json`，没有 `images/` 文件夹。
