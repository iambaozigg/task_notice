# 日程打卡

一个纯前端的每日任务打卡应用，单 HTML 文件，无需后端，直接浏览器打开即可使用。

## 分支说明

本仓库维护两个独立版本，近期无合并计划：

| 分支 | 版本 | 说明 |
|------|------|------|
| `master` | 基础版 | 纯文本打卡，数据存储在 localStorage |
| `dev` | 图片版 | 在基础版上新增图片存储功能，使用 IndexedDB 存储图片 |

## 两个版本的差异

### master（基础版）

- 任务模板管理（增删改）
- 每日打卡（完成/取消）
- 历史回看（按月日历）
- 个性设置（名称、座右铭）
- 数据导入/导出（JSON）
- 数据存储：仅 localStorage

### dev（图片版）

包含基础版全部功能，另外新增：

- 任务关联图片（模板参考图 + 每日打卡照片）
- 图片上传（点击选择、拖拽上传）
- 图片自动压缩（最大 1920px，JPEG 质量 0.8）
- 缩略图展示 + 全屏查看器（左右切换、键盘导航）
- 图片自动清理（可配置保留天数，默认 180 天）
- 数据存储：localStorage（元数据）+ IndexedDB（图片二进制）

## 注意事项

### 关于数据

- **两个版本的 localStorage 数据格式兼容**：master 的数据可以在 dev 中打开，dev 会自动迁移缺少的字段
- **dev 版的图片数据存在 IndexedDB 中**，不在 localStorage 的 JSON 导出文件里
- **导出/导入限制**：dev 版导出的 JSON 只包含图片 ID 引用，不包含图片本身。在新设备导入后，图片需要重新上传
- **图片清理机制**：dev 版每次打开页面时，会自动清除超过保留天数的图片。默认 180 天，可在「个性设置」中调整

### 关于浏览器兼容性

- 需要支持 IndexedDB 的现代浏览器（Chrome、Firefox、Edge、Safari 均可）
- 隐私模式下 IndexedDB 可能不可用，图片功能会降级（控制台有警告，不影响其他功能）

### 关于存储空间

- localStorage 通常限制 5-10MB，仅存储文本数据，一般不会满
- IndexedDB 限制较大（通常几百 MB 到几 GB），但建议定期清理过期图片
- 单张图片上传限制 20MB，超过会提示

### 关于跨分支维护

如果需要在两个版本之间同步基础功能的修复，使用 cherry-pick：

```bash
# 在 master 上修复 bug 后，把该 commit 挑选到 dev
git checkout dev
git cherry-pick <commit-hash>

# 反之亦然
git checkout master
git cherry-pick <commit-hash>
```

**注意**：dev 版新增的代码（IndexedDB、图片 UI 等）不能 cherry-pick 到 master，因为 master 没有这些基础设施。

## 使用方式

直接用浏览器打开 `task_notice.html` 即可，无需安装任何依赖。

dev 版会从 CDN 加载 [idb](https://github.com/jakearchibald/idb) 库（约 1.2KB），需要网络连接。首次加载后浏览器会缓存。

## 数据存储位置

所有数据都存在浏览器本地，不会上传到任何服务器。

| 数据类型 | 存储方式 | 位置 |
|----------|----------|------|
| 任务、模板、设置 | localStorage | 浏览器本地，按域名隔离 |
| 图片二进制数据（仅 dev 版） | IndexedDB | 浏览器本地，按域名隔离 |

各浏览器的 IndexedDB 存储路径（Windows）：

| 浏览器 | 路径 |
|--------|------|
| Chrome | `%LOCALAPPDATA%\Google\Chrome\User Data\Default\IndexedDB\` |
| Edge | `%LOCALAPPDATA%\Microsoft\Edge\User Data\Default\IndexedDB\` |
| Firefox | `%APPDATA%\Mozilla\Firefox\Profiles\<profile>\storage\default\` |

**注意事项：**

- localStorage 和 IndexedDB 都是浏览器本地存储，不经过网络
- 数据按浏览器隔离：Chrome 里存的数据在 Firefox 里看不到
- 清除浏览器数据时如果勾选了「站点数据」，localStorage 和 IndexedDB 都会被删除
- `file://` 协议打开的 HTML 文件，数据绑定在当前浏览器的本地配置中
