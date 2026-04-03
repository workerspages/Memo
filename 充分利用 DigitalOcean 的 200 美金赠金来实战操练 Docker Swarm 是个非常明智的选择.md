# 充分利用 DigitalOcean 的 200 美金赠金来实战操练 Docker Swarm 是个非常明智的选择。

下面是一套从零开始的完整部署指南，包括环境安装、集群初始化、防火墙配置，以及一个包含 AList 和 qBittorrent 的实战部署配置。请依次在你的服务器上执行以下步骤，**所有提供的代码均是完整版本，可以直接复制运行**。

---

### 第一阶段：准备工作与环境安装

假设你已经创建了 3 台 Droplets，分别命名为 `node-1`（作为管理节点 Manager），`node-2` 和 `node-3`（作为工作节点 Worker）。请确保你在 DigitalOcean 面板中找到了它们的**内网 IP (Private IP)**。

**1. 安装 Docker (在所有 3 台节点上执行)**

登录每一台服务器，运行以下完整的自动安装和启动脚本：

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo systemctl enable docker
sudo systemctl start docker

```

**2. 配置防火墙 (在所有 3 台节点上执行)**

Docker Swarm 需要特定的端口来进行节点间的通信和流量路由。在所有服务器上执行以下完整命令来配置 UFW 防火墙：

```bash
sudo ufw allow 22/tcp
sudo ufw allow 2376/tcp
sudo ufw allow 2377/tcp
sudo ufw allow 7946/tcp
sudo ufw allow 7946/udp
sudo ufw allow 4789/udp
sudo ufw reload
sudo ufw enable

```

*(注：执行 `ufw enable` 时如果提示可能会中断 SSH 连接，输入 `y` 确认即可，因为我们已经放行了 22 端口)*

---

### 第二阶段：初始化 Docker Swarm 集群

**1. 初始化主节点 (仅在 `node-1` 上执行)**

将下面的 `<node-1内网IP>` 替换为你第一台服务器的 DigitalOcean Private IP：

```bash
docker swarm init --advertise-addr <node-1内网IP>

```

**2. 获取加入令牌并加入集群 (在 `node-2` 和 `node-3` 上执行)**

`node-1` 初始化成功后，终端会输出一段带有 `token` 的命令。把它复制下来，分别在 `node-2` 和 `node-3` 上粘贴并运行。完整的命令格式看起来像这样：

```bash
docker swarm join --token SWMTKN-1-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx <node-1内网IP>:2377

```

**3. 验证集群状态 (回到 `node-1` 上执行)**

```bash
docker node ls

```

如果一切顺利，你将看到 3 台节点都在列表中，并且状态为 `Ready`，其中 `node-1` 的 Manager Status 为 `Leader`。你的集群已经搭建完毕！

---

### 第三阶段：集群管理与实战部署

在 Swarm 中，我们使用 Docker Stack 来管理应用。这里为你提供一个非常实用的完整配置文件示例，它将在你的集群中部署一个高可用的 AList 挂载程序，以及一个被限制在主节点运行的 qBittorrent 下载器。

**1. 创建配置文件 (在 `node-1` 上执行)**

创建一个名为 `docker-compose.yml` 的文件：

```bash
nano docker-compose.yml

```

将以下完整的配置代码粘贴进去（没有任何删减）：

```yaml
version: '3.8'

services:
  alist:
    image: xhofe/alist:latest
    ports:
      - "5244:5244"
    volumes:
      - alist_data:/opt/alist/data
    deploy:
      replicas: 2
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Shanghai
      - WEBUI_PORT=8080
    ports:
      - "8080:8080"
      - "6881:6881/tcp"
      - "6881:6881/udp"
    volumes:
      - qbittorrent_config:/config
      - qbittorrent_downloads:/downloads
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager

volumes:
  alist_data:
  qbittorrent_config:
  qbittorrent_downloads:

```

*(按 `Ctrl+O` 保存，回车确认，然后按 `Ctrl+X` 退出)*

**2. 一键部署到集群 (在 `node-1` 上执行)**

运行以下命令，将这个 Stack 部署到你的三台服务器上：

```bash
docker stack deploy -c docker-compose.yml mystack

```

**3. 查看运行状态 (在 `node-1` 上执行)**

你可以随时使用以下命令查看服务运行情况和分配到了哪些机器上：

```bash
docker service ls
docker service ps mystack_alist
docker service ps mystack_qbittorrent

```

此时，你可以通过访问**任意一台**服务器的公网 IP 加上 `5244` 端口（例如 `http://node-2的公网IP:5244`），Swarm 的路由网格（Routing Mesh）会自动将你的请求转发到正在运行 AList 的容器中。

---

Q:集群部署完成后，你是否需要我提供在集群环境中配置 rclone 任务，或者将本地数据同步到这个新集群的完整命令？
A:

在 Docker Swarm 集群中运行 rclone，最实用的场景之一就是配合 AList 和 qBittorrent 进行自动化的文件转移和备份。例如，我们可以设置一个后台守护容器，让它定期将 qBittorrent 下载好的特定分类（如“电影”、“AV”）自动移动到移动云盘中，或者定时同步你的“手机备份”文件夹。

为了实现这一点，我们将在之前的配置基础上增加一个 `rclone_tasks` 服务。这个服务会通过挂载共享的数据卷，直接读取 qBittorrent 的下载目录以及宿主机上的备份目录，并执行循环任务。

以下是调整后的 **最终完整版本 `docker-compose.yml` 文件**，没有任何删减。

### 1. 更新集群配置文件 (在 node-1 上执行)

请在 `node-1`（主节点）上编辑之前的 `docker-compose.yml` 文件：

```bash
nano docker-compose.yml

```

用以下完整代码覆盖原有的内容：

```yaml
version: '3.8'

services:
  alist:
    image: xhofe/alist:latest
    ports:
      - "5244:5244"
    volumes:
      - alist_data:/opt/alist/data
    deploy:
      replicas: 2
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Shanghai
      - WEBUI_PORT=8080
    ports:
      - "8080:8080"
      - "6881:6881/tcp"
      - "6881:6881/udp"
    volumes:
      - qbittorrent_config:/config
      - qbittorrent_downloads:/downloads
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager

  rclone_tasks:
    image: rclone/rclone:latest
    volumes:
      # 挂载宿主机的 rclone 配置文件目录
      - /root/.config/rclone:/config/rclone
      # 共享 qbittorrent 的下载目录
      - qbittorrent_downloads:/downloads
      # 挂载宿主机上的本地备份目录（可用于存放外部传来的数据）
      - /root/backup:/backup
    deploy:
      replicas: 1
      placement:
        # 必须与 qbittorrent 运行在同一管理节点，以确保 bind mounts 和共享卷路径一致
        constraints:
          - node.role == manager
    # 设置一个死循环脚本，每 12 小时 (43200 秒) 自动执行一次同步/移动任务
    command: sh -c "while true; do echo '开始执行 rclone 自动化任务...'; rclone move /downloads/电影 alist:ChinaMobileDrive/电影 -v --delete-empty-src-dirs; rclone move /downloads/AV alist:ChinaMobileDrive/AV -v --delete-empty-src-dirs; rclone sync /backup/手机备份 alist:ChinaMobileDrive/手机备份 -v; echo '任务执行完毕，等待下一次循环...'; sleep 43200; done"

volumes:
  alist_data:
  qbittorrent_config:
  qbittorrent_downloads:

```

### 2. 准备 rclone 配置文件并创建本地目录

在重新部署应用之前，我们需要在 `node-1` 主机上准备好 rclone 的配置文件和本地备份目录，因为 `rclone_tasks` 容器会直接读取它们。

1. **创建本地数据目录**：
在 `node-1` 终端执行以下命令，创建对应的本地备份文件夹（用于放置你想同步的旧电脑或手机数据）：
```bash
mkdir -p /root/backup/手机备份
mkdir -p /root/.config/rclone

```


2. **生成或拷入 `rclone.conf**`：
你可以直接在 `node-1` 上运行交互式配置命令来绑定你的 AList 或其他网盘节点：
```bash
docker run --rm -it -v /root/.config/rclone:/config/rclone rclone/rclone config

```


*(注：在配置过程中，请将新建的远程节点名称命名为 `alist`，这样才能与上面 yaml 文件中的 `alist:ChinaMobileDrive` 路径对应上。)*

### 3. 应用更新并部署

配置环境就绪后，再次运行部署命令。Docker Swarm 会自动检测到 yaml 文件的变化，保留正在运行的 AList 和 qBittorrent，并平滑地启动全新的 rclone 任务容器：

```bash
docker stack deploy -c docker-compose.yml mystack

```

**验证与日志查看：**
你可以使用以下命令查看 rclone 任务容器的实时运行日志，确认文件是否在正常传输：

```bash
docker service logs -f mystack_rclone_tasks

```

如果你还没有在 AList 中配置好中国移动云盘，或者需要了解如何提取 AList 的 WebDAV 链接来完成 rclone 的 `config` 步骤，需要我提供进一步的对接指南吗？
