# 任务图片存储功能设计文档

## 概述

为「日程打卡」应用添加图片存储能力，支持任务模板和每日打卡记录关联图片。使用 IndexedDB 存储图片二进制数据，localStorage 中仅保留图片 ID 引用。包含自动清理过期图片的机制。

## 架构设计

### 存储分层

- **localStorage**：存储应用状态（state），包括模板、日期记录、设置，以及图片 ID 引用列表
- **IndexedDB**：存储图片二进制数据（Blob）

这种分层设计保持了现有导入/导出逻辑的兼容性，同时利用 IndexedDB 的大容量存储能力。

### 数据流

```
用户上传图片
    ↓
读取文件（FileReader / Clipboard API）
    ↓
Canvas 压缩（最大宽度 1920px，质量 0.8）
    ↓
生成 Blob
    ↓
存入 IndexedDB，返回图片 ID
    ↓
将图片 ID 写入 state（模板或每日任务的 images 数组）
    ↓
保存 state 到 localStorage
```

## 数据模型

### localStorage state 变更

#### settings 新增字段

```js
state.settings = {
  ownerName: "...",
  motto: "...",
  imageRetentionDays: 180  // 图片保留天数，默认 180（6个月）
}
```

#### 模板（templates）新增字段

```js
{
  id: "uuid",
  name: "任务名称",
  category: "分类",
  repeat: "daily",
  note: "备注",
  images: ["img-id-1", "img-id-2"]  // 图片 ID 列表
}
```

#### 每日任务（days[date].tasks）新增字段

```js
{
  templateId: "uuid",
  done: false,
  images: ["img-id-3"]  // 打卡图片 ID 列表
}
```

### IndexedDB schema

```
数据库名: daily-checkin-images
版本: 1
ObjectStore: images
  keyPath: id
  value: {
    id: string,           // 唯一标识（与 state 中引用对应）
    blob: Blob,           // 压缩后的图片二进制数据
    type: string,         // MIME type（image/jpeg, image/png）
    width: number,        // 压缩后宽度
    height: number,       // 压缩后高度
    originalName: string, // 原始文件名
    createdAt: number     // 创建时间戳（Date.now()）
  }
```

## 图片处理

### 压缩策略

- 最大宽度：1920px（高度按比例缩放）
- 输出格式：JPEG，质量 0.8
- 如果原图小于最大宽度，不放大，直接压缩质量
- PNG 透明通道在压缩时会丢失（转为 JPEG），这是可接受的权衡

### 压缩流程

1. 创建 Image 对象加载图片
2. 计算缩放比例（如果宽度 > 1920px）
3. 创建 Canvas，绘制缩放后的图片
4. `canvas.toBlob('image/jpeg', 0.8)` 生成压缩后的 Blob
5. 存入 IndexedDB

## 用户交互

### 图片上传方式

1. **点击选择**：点击上传按钮，打开文件选择器（accept="image/*"）
2. **拖拽上传**：将图片拖拽到上传区域
3. **粘贴上传**：监听 `paste` 事件，支持从剪贴板粘贴图片

### 图片展示

- **缩略图**：任务卡片内以网格形式展示缩略图（每行最多 3-4 张）
- **全屏查看**：点击缩略图后进入全屏模式，支持左右箭头键/滑动切换图片

### 上传区域位置

- 任务模板编辑对话框：备注输入框下方
- 每日打卡任务列表：每个任务项内

## 启动清理机制

### 触发时机

应用启动时（`initState()` 函数中），在加载 state 之后执行清理。

### 清理流程

```
1. 读取 settings.imageRetentionDays（默认 180）
2. 计算截止时间：cutoff = Date.now() - imageRetentionDays * 86400000
3. 遍历 IndexedDB images store 中所有记录
4. 对每张图片，如果 createdAt < cutoff：
   a. 删除 IndexedDB 中的图片记录
   b. 遍历 state.templates，从 images 数组中移除该 ID
   c. 遍历 state.days[date].tasks，从 images 数组中移除该 ID
5. 保存更新后的 state 到 localStorage
```

### 边界情况

- 如果 settings 中没有 imageRetentionDays 字段，使用默认值 180
- 清理过程中如果 IndexedDB 操作失败，不影响应用正常启动
- 清理是异步操作，不阻塞 UI 渲染

## 设置面板

在现有设置界面中新增：

- 「图片保留时间」数字输入框，默认 180，最小值 1，单位天
- 说明文字：「超过保留时间的图片将在下次打开应用时自动清除」

## 兼容性处理

### state 迁移

- 旧版本 state 中的模板和任务没有 `images` 字段，加载时自动初始化为空数组 `[]`
- 旧版本 settings 中没有 `imageRetentionDays` 字段，加载时使用默认值 180

### 导入/导出

- JSON 导出保持现有格式，仅包含图片 ID 引用
- 导入时，图片 ID 引用被保留，但对应的图片数据需要用户重新上传（因为图片数据在 IndexedDB 中，不在 JSON 中）
- 这是有意的设计权衡：保持 JSON 导出文件的轻量性

## 技术依赖

- **idb**（~1.2KB）：IndexedDB 的 Promise 封装库，通过 CDN 引入
- 无其他外部依赖，保持单文件应用的特性
