https://github.com/alexta69/metube

以下是 MeTube 项目 README.md 文档的总结及可设置的环境变量列表：

### 📝 项目总结
**MeTube** 是一个为 `youtube-dl`（实际使用的是其更强大的分支 `yt-dlp`）提供 Web 图形用户界面（GUI）的开源项目。它支持播放列表解析，允许用户通过浏览器直接从 YouTube 以及其他数十个受支持的视频网站下载视频或音频。它非常适合搭建在个人服务器或 NAS 上，官方推荐使用 Docker 或 Docker Compose 进行部署，并且提供了丰富的浏览器扩展（Chrome、Firefox）、iOS 快捷指令以及书签等便捷功能，支持反向代理和 HTTPS，也可以加载浏览器 Cookies 以下载受限视频。

---

### ⚙️ 可配置环境变量列表

MeTube 提供了丰富的环境变量供配置，可以通过 Docker 命令行中的 `-e` 参数或在 `docker-compose.yml` 中的 `environment` 节点下进行设置。这些变量主要分为以下几个类别：

#### 1. 下载行为 (Download Behavior)
* **`MAX_CONCURRENT_DOWNLOADS`**: 允许同时进行的最大下载任务数量。例如设置为 5，则最多 5 个任务同时下载，其他的排队等待（默认值：`3`）。
* **`DELETE_FILE_ON_TRASHCAN`**: 设置为 `true` 时，在 UI 的“已完成”列表中删除项目会同步删除服务器上的实际下载文件（默认值：`false`）。
* **`DEFAULT_OPTION_PLAYLIST_ITEM_LIMIT`**: 允许下载的播放列表项的最大数量（默认值：`0`，即不限制）。
* **`CLEAR_COMPLETED_AFTER`**: 多少秒后自动从 UI 列表中清除已完成或失败的下载记录（默认值：`0`，即禁用自动清除）。

#### 2. 存储与目录配置 (Storage & Directories)
* **`DOWNLOAD_DIR`**: 视频下载的保存目录（Docker 镜像内默认为 `/downloads`）。
* **`AUDIO_DOWNLOAD_DIR`**: 仅音频下载的保存目录。如果不设置，则默认与 `DOWNLOAD_DIR` 相同。
* **`CUSTOM_DIRS`**: 是否开启在下载目录内选择自定义文件夹的功能（默认值：`true`）。
* **`CREATE_CUSTOM_DIRS`**: 是否支持通过文本输入自动创建不存在的自定义下载文件夹（默认值：`true`）。
* **`CUSTOM_DIRS_EXCLUDE_REGEX`**: 用于排除某些不希望显示在自定义目录下拉菜单中的文件夹的正则表达式（默认隐藏以 `.` 或 `@` 开头的文件夹）。
* **`DOWNLOAD_DIRS_INDEXABLE`**: Web 服务器上是否允许直接索引（浏览）下载目录的文件（默认值：`false`）。
* **`STATE_DIR`**: 保存队列持久化文件（配置与进度）的目录（Docker 中默认为 `/downloads/.metube`）。
* **`TEMP_DIR`**: 临时下载文件的保存路径。建议将其指向 SSD 或 RAM 盘（如 `tmpfs`）以提高性能。
* **`CHOWN_DIRS`**: 容器启动时是否自动更改相关目录的权限所有者。如果你已经配置好了用户权限，可以将其设为 `false`（默认值：`true`）。

#### 3. 文件命名与底层配置 (File Naming & yt-dlp)
* **`OUTPUT_TEMPLATE`**: 下载视频的文件名模板格式（默认值：`%(title)s.%(ext)s`）。
* **`OUTPUT_TEMPLATE_CHAPTER`**: 视频按章节分割时的文件名模板（默认值：`%(title)s - %(section_number)s %(section_title)s.%(ext)s`）。
* **`OUTPUT_TEMPLATE_PLAYLIST`**: 作为播放列表下载时的文件名模板（默认值：`%(playlist_title)s/%(title)s.%(ext)s`）。
* **`OUTPUT_TEMPLATE_CHANNEL`**: 频道下载时的文件名模板（默认值：`%(channel)s/%(title)s.%(ext)s`）。
* **`YTDL_OPTIONS`**: 以 JSON 格式传递给 `yt-dlp` 的额外高级配置选项（如格式控制、代理等）。
* **`YTDL_OPTIONS_FILE`**: 指定一个包含上述 JSON 数据的外部文件路径，当文件内容发生更改时，MeTube 会自动重载。

#### 4. Web 服务器与网络访问 (Web Server & URLs)
* **`HOST`**: Web 服务器绑定的主机 IP（默认值：`0.0.0.0`，即所有网络接口）。
* **`PORT`**: Web 界面监听的端口（默认值：`8081`）。
* **`URL_PREFIX`**: Web 服务器的根路径（常用于反向代理的子目录部署，默认值：`/`）。
* **`PUBLIC_HOST_URL`**: 若你的文件可以通过其他 URL 访问，可用此变量替换 UI 中显示的下载链接的前缀地址。
* **`PUBLIC_HOST_AUDIO_URL`**: 同上，专门针对音频下载的 URL 前缀。
* **`HTTPS`**: 是否启用 HTTPS 模式（默认值：`false`）。
* **`CERTFILE`**: 当启用 HTTPS 时，指定证书文件的路径。
* **`KEYFILE`**: 当启用 HTTPS 时，指定密钥文件的路径。
* **`ROBOTS_TXT`**: 挂载到容器内的 `robots.txt` 文件的路径。

#### 5. 系统基础设置 (Basic Setup)
* **`PUID`**: 运行 MeTube 进程的系统用户 ID（默认值：`1000`）。
* **`PGID`**: 运行 MeTube 进程的系统用户组 ID（默认值：`1000`）。
* **`UMASK`**: 创建新文件和文件夹的 umask 掩码（默认值：`022`）。
* **`DEFAULT_THEME`**: 用户界面的默认主题，可选 `light`、`dark` 或 `auto`（默认值：`auto`）。
* **`LOGLEVEL`**: 日志输出的等级，可选 `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL` 等（默认值：`INFO`）。
* **`ENABLE_ACCESSLOG`**: 是否启用网络请求访问日志（默认值：`false`）。
