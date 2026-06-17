# 日程打卡

一个单文件前端日程打卡应用，入口为 `index.html`。支持每日任务、固定模板、历史回看、图片备份、AI 分析，以及基于腾讯云 CloudBase 的文字数据云同步。

## 当前状态

最新发布标签：`V2.1.0`

当前开发分支：`feature/cloud-sync`

`feature/cloud-sync` 在 `V2.1.0` 基础上继续增加 CloudBase 云同步、账号切换、移动端布局优化、AI 分析交互优化、每周多日重复等能力。合并到稳定分支前，建议先在腾讯云托管公网地址上做完整回归测试。

| 分支 | 用途 | 当前状态 |
|------|------|----------|
| `master` | 稳定发布分支 | 稳定版本 |
| `dev` | 日常集成分支 | 集成版本 |
| `feature/cloud-sync` | CloudBase 云同步开发分支 | 当前主要开发分支 |

## 主要功能

- 每日任务打卡：完成、取消、历史回看。
- 每日模板：新增、编辑、删除固定任务。
- 重复规则：每日重复、工作日重复、每周多日重复。
- 富文本任务描述：支持加粗、斜体、下划线、删除线、列表、链接、清除格式。
- 图片记录：模板参考图和每日打卡照片。
- 图片上传：点击选择、拖拽上传、粘贴上传、移动端拍照。
- 图片压缩：可配置最大分辨率和压缩质量，默认 1920px / 0.8。
- 图片查看器：缩略图展示、全屏查看、左右切换、键盘导航。
- 图片自动清理：默认保留 180 天，可在设置中调整。
- 数据导入导出：支持 JSON，以及包含图片的 ZIP 包。
- CloudBase 云同步：按账号同步模板、打卡记录、非敏感设置。
- 账号切换：切换前先同步当前账号，再退出并显示登录框。
- AI 分析：今日、本周、近 30 天、今日建议四种模式。
- AI 最近结果保留：每个分析模式在本机保留最近一次结果。
- AI 智能激励语：每天生成个性化鼓励内容。
- 移动端适配：优化顶部区域、Tab、弹窗、AI 面板和横向溢出。

## 使用方式

本地直接打开 `index.html` 可使用本机模式。

如果要测试 CloudBase 云同步，建议通过腾讯云 Web 应用托管或静态网站托管访问公网 HTTPS 地址。CloudBase Web SDK 对安全域名、登录态和网络环境更敏感，本地 `file://` 或未放行域名可能无法正常登录。

## CloudBase 云同步

当前云同步采用低改造方案：每个用户在 `user_states` 集合保存一条文档。

```js
{
  owner: "当前登录用户 uid",
  data: {
    templates: [],
    days: {},
    settings: {}
  },
  version: 1,
  updatedAt: Date.now()
}
```

同步范围：

| 数据 | 是否同步 | 说明 |
|------|----------|------|
| templates | 是 | 每日模板 |
| days | 是 | 每日任务和打卡记录 |
| settings 非敏感项 | 是 | 名称、座右铭、AI 模型等 |
| aiApiKey | 否 | 只保存在本机 localStorage |
| 图片 Blob | 否 | 只保存在当前设备 IndexedDB |
| AI 最近分析结果 | 否 | 只保存在本机 localStorage |

注意：

- 照片不会跨设备同步。
- 换设备后，如果云端文字数据里有图片 ID，但当前设备没有图片 Blob，页面会显示图片缺失提示。
- 顶部云同步状态会显示上次同步时间。
- 手动点击“立即同步”会先检查云端版本，再决定加载云端或上传本机数据。

## CloudBase 控制台要求

需要在 CloudBase 控制台完成：

1. 开通身份认证，并创建允许登录的用户。
2. 启用用户名密码登录方式。
3. 创建文档型数据库集合：`user_states`。
4. 配置数据库权限，确保用户只能访问自己的文档。
5. 将网页访问域名加入 Web 安全域名。

权限规则建议：

```json
{
  "read": "auth.uid != null && doc.owner == auth.uid",
  "create": "auth.uid != null && request.data.owner == auth.uid",
  "update": "auth.uid != null && doc.owner == auth.uid && request.data.owner == auth.uid",
  "delete": "auth.uid != null && doc.owner == auth.uid"
}
```

如果 iPad Safari 打开网页变成下载文件，请检查托管响应头，确保没有：

```text
content-disposition: attachment
```

并确保 `index.html` 的 `content-type` 为：

```text
text/html; charset=utf-8
```

## 数据存储

| 数据类型 | 存储方式 | 说明 |
|----------|----------|------|
| 本机文字缓存 | localStorage | 按账号隔离缓存 |
| 云端文字数据 | CloudBase `user_states` | 登录后同步 |
| 图片二进制数据 | IndexedDB | 当前设备本地照片 |
| AI API Key | localStorage | 本机私密配置，不写入云端 |
| AI 最近分析结果 | localStorage | 本机缓存，不写入云端 |

## 导入导出

- JSON 导出只包含文字数据和图片 ID，不包含图片 Blob。
- ZIP 导出包含 `data.json` 和 `images/`，代表当前设备完整备份。
- 导入 ZIP 会把图片恢复到当前设备 IndexedDB，但不会上传到 CloudBase。
- 登录后导入 JSON/ZIP 会覆盖当前账号的 CloudBase 文字数据。
- 清除本机缓存不会删除云端数据。
- 删除云端数据只删除当前登录账号的 `user_states` 文档，需要强确认。

## 浏览器兼容性

建议使用现代浏览器：

- Chrome
- Edge
- Firefox
- Safari

移动端建议使用系统默认现代浏览器。隐私模式下 localStorage 或 IndexedDB 可能受限，图片和登录态可能无法稳定保存。

## 版本记录

| 标签 | 说明 |
|------|------|
| `V1.0.0` | 稳定版，包含图片存储、ZIP 导出导入等能力 |
| `V2.0.0` | 增加 AI 分析、AI 周报、今日任务建议、智能激励语等能力 |
| `V2.0.1` | 将入口文件从 `task_notice.html` 改为 `index.html` |
| `V2.1.0` | 富文本编辑体验增强，支持删除线等格式能力 |

## 分支和远端

当前仓库同时维护 Gitee 和 GitHub 远端：

```bash
origin  https://gitee.com/jim1986-gitee/task_notice
github  git@github.com:iambaozigg/task_notice.git
```

推送当前开发分支：

```bash
git push origin feature/cloud-sync
git push github feature/cloud-sync
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

本仓库的 GitHub SSH 推送可使用 443 端口：

```bash
git config core.sshCommand "ssh -o HostName=ssh.github.com -p 443"
```
