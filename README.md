# 日程打卡

一个纯前端的每日任务打卡应用。项目入口为 `index.html`，无需后端服务，直接用浏览器打开即可使用。

## 当前版本

当前发布版本：`V2.0.1`

相对 `V1.0.0`，`V2.x` 主要增加了 AI 分析能力，并保留图片、ZIP 导入导出等增强功能。

| 分支 | 用途 | 当前状态 |
|------|------|----------|
| `master` | 稳定发布分支 | 已同步到 `V2.0.1` |
| `dev` | 日常集成分支 | 已同步到 `V2.0.1` |

## 主要功能

- 每日任务打卡：完成、取消、历史回看。
- 任务模板管理：新增、编辑、删除、排序。
- 富文本任务描述：支持更丰富的任务说明。
- 个性设置：名称、座右铭、图片保留天数、压缩参数等。
- 图片记录：模板参考图和每日打卡照片。
- 图片上传：点击选择、拖拽上传、粘贴上传、移动端拍照。
- 图片压缩：可配置最大分辨率和压缩质量，默认 1920px / 0.8。
- 图片查看器：缩略图展示、全屏查看、左右切换、键盘导航。
- 图片自动清理：默认保留 180 天，可在设置中调整。
- 数据导入导出：支持 JSON，以及包含图片的 ZIP 包。
- AI 打卡分析报告：分析完成率趋势、行为模式和改进建议。
- AI 周报：总结本周表现、亮点、不足和下周建议。
- AI 今日任务建议：生成执行顺序、时间分配、执行技巧和完成率预估。
- AI 智能激励语：每天生成个性化鼓励内容。
- AI 分析风格：支持友好鼓励、严格督促、简洁高效、自定义 Prompt。

## 使用方式

直接用浏览器打开 `index.html`。

本应用为单文件前端应用，内嵌所需前端资源。除 AI 功能需要访问用户配置的大模型服务外，任务、图片和打卡数据均保存在浏览器本地。

## 数据存储

| 数据类型 | 存储方式 | 说明 |
|----------|----------|------|
| 任务、模板、设置、打卡记录 | localStorage | 文本和元数据 |
| 图片二进制数据 | IndexedDB | 模板参考图和每日打卡照片 |

注意：

- localStorage 和 IndexedDB 都是浏览器本地存储。
- 数据按浏览器隔离，Chrome、Edge、Firefox 之间不会自动共享。
- 清除浏览器站点数据时，localStorage 和 IndexedDB 可能一起被删除。
- JSON 导出不包含图片二进制，只包含图片 ID 引用。
- ZIP 导出包含 `data.json` 和 `images/`，适合完整备份和迁移。

## 浏览器兼容性

建议使用现代浏览器：

- Chrome
- Edge
- Firefox
- Safari

隐私模式下 IndexedDB 可能不可用，图片功能可能降级，但基础打卡功能仍可使用。

## 版本记录

| 标签 | 说明 |
|------|------|
| `V1.0.0` | 稳定版，包含图片存储、ZIP 导出导入等能力 |
| `V2.0.0` | 增加 AI 分析、AI 周报、今日任务建议、智能激励语等能力 |
| `V2.0.1` | 将入口文件从 `task_notice.html` 改为 `index.html` |

## 分支和远端

当前仓库同时维护 Gitee 和 GitHub 远端：

```bash
origin  https://gitee.com/jim1986-gitee/task_notice
github  git@github.com:iambaozigg/task_notice.git
```

常用发布流程：

```bash
git switch dev
# 开发和提交

git switch master
git merge dev

git tag -a Vx.y.z -m "Release Vx.y.z - ..."

git push origin master dev --tags
git push github master dev --tags
```

本仓库的 GitHub SSH 推送使用 443 端口：

```bash
git config core.sshCommand "ssh -o HostName=ssh.github.com -p 443"
```
