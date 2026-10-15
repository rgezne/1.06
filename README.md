# gopeed-extension-pikpak

[Gopeed](https://gopeed.com) 的 PikPak 分享解析扩展。把 `https://mypikpak.com/s/<share_id>[/<parent_id>]` 形式的分享链接喂给 Gopeed，自动列出所有文件并解析直链下载。

> **视频可匿名下载；zip 等其他文件需要登录并转存。** PikPak 对分享里的非视频文件不给直链（网页端的“下载”按钮也只是引导你先转存）。登录后，扩展会把这些文件**转存到你自己的 PikPak 网盘**，再取网盘里的下载地址。
>
> - 转存会占用你的网盘空间；免费账号每天的转存次数有限
> - 同一分享里转存过的文件会记住，重复解析不会再次转存（除非你把网盘里那份删了）

## 功能

- 识别 `mypikpak.com` / `mypikpak.net` 上的分享链接
- 自动算 captcha_sign；视频无需账号
- 可选登录（账号密码 / refresh_token），登录态缓存在扩展存储里，过期自动刷新
- 登录后自动转存非视频文件到自己网盘再下载
- 递归遍历子目录 + 自动翻页
- `target_path` 可指定单个文件，也可指定文件夹并递归展开全部嵌套文件
- 文件夹结果交给 Gopeed 原生文件树，可展开目录并勾选需要下载的文件
- 单文件 / 多文件 / 嵌套目录都保留原始结构
- 支持密码型分享（在扩展设置里填密码）
- 视频可选转码地址（清晰度低但更稳）

## 安装

### 通过 Gopeed Scheme（推荐）

打开下面这个链接（桌面端 Gopeed 已运行时）：

```
gopeed://install/https%3A%2F%2Fgithub.com%2FTeamBreakerr%2Fgopeed-extension-pikpak
```

或在 Gopeed 扩展页粘贴：`https://github.com/TeamBreakerr/gopeed-extension-pikpak`

### 本地开发模式

1. 在 Gopeed 扩展页连续点击安装按钮 5 次开启开发者模式
2. 点目录选择按钮，选中本仓库目录
3. 创建任务时贴 PikPak 分享链接即可

### 命令行调试

不进 Gopeed 也能跑：仓库根目录的 `cli.js` 用 Node 模拟 Gopeed 运行时，设置项走环境变量（`PIKPAK_USERNAME` / `PIKPAK_PASSWORD` / `PIKPAK_REFRESH_TOKEN` / `PIKPAK_SHARE_PWD` / `PIKPAK_TARGET_PATH` 等），扩展存储落在仓库根目录的 `.storage.json`（可用 `PIKPAK_STORAGE_FILE` 自定义）。

```bash
PIKPAK_TARGET_PATH="DyDy_cos/update/26.09 patreon" node cli.js https://mypikpak.com/s/<share_id>
```

## 设置项

| 设置 | 默认 | 说明 |
|---|---|---|
| `api_host` | `api-drive.mypikpak.com` | PikPak API 域名。某些规则集（如 ChinaMax）会把 `.net` 误判为中国域名走直连，默认用 `.com` 命中率更高 |
| `user_host` | `user.mypikpak.com` | captcha/init 域名 |
| `use_transcoding` | 关闭 | 视频文件是否优先用转码地址 |
| `share_pwd` | 空 | 密码分享时填这里 |
| `target_path` | 空 | 可填写文件或文件夹路径；文件只解析单项，文件夹递归解析整个子树并在 Gopeed 文件树中多选；支持从分享根目录或链接所在目录开始，留空则扫描整个分享 |
| `username` / `password` | 空 | PikPak 账号密码。若登录被要求人机验证（扩展里无法完成），改用 `refresh_token` |
| `refresh_token` | 空 | 从网页端取得的 refresh_token，见下文 |

注意：Gopeed 设置项是明文保存的。

### `target_path` 示例

- 指定单个文件：`DyDy_cos/update/26.09 patreon/ALL IN_S.mp4`
- 指定文件夹：`DyDy_cos/update/26.09 patreon`

当 `target_path` 指向文件夹时，扩展会递归列出该文件夹以及所有嵌套子目录中的文件，并通过 `files[].path` 保留目录结构。Gopeed 会在解析结果中显示原生文件树，你可以展开文件夹、按文件勾选后再开始下载。扩展不支持解析阶段之后的远端懒加载，因此目标文件夹会在解析时完成枚举；大型目录可能需要更长时间。

### 为什么以前文件夹会"超时"，v1.3.0 怎么解决的

Gopeed 前端对"创建任务"的解析请求有硬超时（v1.9.x 为 60 秒，v2.x 为 120 秒）。旧版本解析文件夹时对每个文件都**串行**发两次 API（一次 `share/file_info` 取分享直链、转存后再一次 `files/{id}` 取网盘直链），文件一多总耗时轻松超过 60 秒，Gopeed 就直接报"超时"。v1.3.0 做了三件事：

1. **并发取链**：`share/file_info` 和转存后的 `files/{id}` 改为并发执行（上限 4，避免触发 PikPak 风控）。在 Gopeed v2.x（webview 运行时）是真并发；在 v1.9.x（goja 运行时）中 fetch 底层仍是串行的，但单项失败不再拖垮整个解析。
2. **整文件夹转存快速路径**：同一目录下待转存文件数 ≥ 5 时，直接把整个文件夹一次性转存到自己网盘，再对该目录做一次分页列表——列表响应自带每个文件的 `web_content_link`/`medias`，把 O(N) 次串行请求降为常数次。代价是该文件夹里的其他文件（如视频）也会一并转存，占用转存配额；快速路径失败时自动回退为逐文件转存。
3. **转存任务轮询提速**：从 1 秒/次加快到 500 毫秒/次，缩短等待转存任务完成的时间。

日志中现在会输出各阶段耗时（`listing done in X ms` / `file_info done in X ms` / `restore done in X ms` / `resolved N files in X ms`），便于定位慢在哪一步。

路径指向具体文件时仍保持单文件兼容行为。匿名模式下没有可用下载地址的文件仍会被跳过；如果目标文件夹内全部文件都没有下载地址，需要配置账号密码或 `refresh_token`，让扩展尝试转存后获取下载地址。

## 获取 refresh_token

1. 用浏览器**无痕窗口**打开 <https://mypikpak.com/drive> 并登录
2. F12 打开控制台，执行：
   ```js
   JSON.parse(localStorage.getItem("credentials_YUMx5nI8ZU8Ap8pm")).refresh_token
   ```
3. 把输出的字符串填进扩展设置的 `refresh_token`
4. **直接关闭无痕窗口，不要点退出登录**（退出会吊销这个 token）

refresh_token 每次刷新都会轮换，扩展会把新值存在自己的存储里；设置里的值只在你粘贴新 token 时生效。如果提示 refresh_token 已失效，重复上面的步骤即可。

## 网络要求

PikPak 对 IDC IP 段（DigitalOcean/AWS/各类云厂商）有屏蔽。如果直连返回 `share_status: PROHIBITED`，需要给 `mypikpak.com` 配住宅出口的代理（HK/JP/SG 一般可用）：

- **Surge / Quantumult X / Clash** 加分流：
  ```
  DOMAIN-SUFFIX,mypikpak.com,你的代理节点
  DOMAIN-SUFFIX,mypikpak.net,你的代理节点
  ```
- **Gopeed Docker** 容器需要把 HTTP 代理填到 Gopeed 全局设置里（容器流量不会自动走宿主机的系统代理），或在容器层面用 `HTTP_PROXY` 环境变量。

## 实现原理

参考 [alist 的 pikpak_share 驱动](https://github.com/alist-org/alist/tree/main/drivers/pikpak_share)，移植成纯 JS 在 Gopeed 的 goja 运行时里跑：

```
parseShareUrl(URL)
       │
       ▼
captcha/init  (POST user.mypikpak.com)
  body: { captcha_sign = md5_chain(client_id+ver+pkg+device_id+ts, salts...) }
       │  → captcha_token
       ▼
(可选) 登录  POST user.mypikpak.com/v1/auth/signin 或 /v1/auth/token
       │  → access_token（缓存在 gopeed.storage，之后请求带 Authorization）
       ▼
share/detail  (GET api-drive.mypikpak.com)
  query: { share_id, parent_id, pass_code_token, page_token, ... }
       │  → 递归子目录 + 翻页
       ▼
share/file_info (GET api-drive.mypikpak.com)
  query: { share_id, file_id, pass_code_token }
       │  → medias[].link（仅视频有）
       ▼
(登录时，对没有链接的文件) share/restore → tasks/{id} 轮询(500ms) → files/{own_id}?usage=FETCH
       │  同目录待转存文件 ≥ 5 时走快速路径：
       │  share/restore(整个文件夹) → files?parent_id={own_folder_id}（列表自带全部直链）
       │  → web_content_link
       ▼
ctx.res = { name, files: [{ name, path, size, req: { url, extra.header } }] }
```

## License

MIT

## 致谢

- [alist](https://github.com/alist-org/alist) — PikPak 分享 API 与签名算法
- [Gopeed](https://gopeed.com) — 优秀的开源下载器和扩展生态
- [Joseph Myers' MD5](https://www.myersdaily.org/joseph/javascript/md5-text.html) — goja 运行时下的 MD5 实现
