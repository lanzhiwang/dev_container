
------------------------------------------------------------------------------------------------------

Q: 假设你是一位精通 docker 的高级开发工程师, 现在我有如下 Dockerfile
```
FROM golang:1.24.1

ARG USERNAME=lanzhiwang
ARG USER_UID=1000
ARG USER_GID=$USER_UID

############################## tools
RUN set -eux; \
	apt-get update; \
	apt-get install -y --no-install-recommends ca-certificates tree iputils-ping vim wget curl iproute2 iptables lsof dnsutils sudo openssh-client openssh-server; \
	rm -rf /var/lib/apt/lists/*; \
	echo "alias ll='ls -alFi --color=auto'" >> /etc/bash.bashrc; \
	groupadd --gid $USER_GID $USERNAME; \
	useradd --uid $USER_UID --gid $USER_GID -m -s /bin/bash $USERNAME; \
	echo "$USERNAME ALL=(root) NOPASSWD:ALL" > /etc/sudoers.d/$USERNAME; \
	chmod 0440 /etc/sudoers.d/$USERNAME; \
	mkdir -p /run/sshd; \
	chmod 0755 /run/sshd
############################## tools

USER $USERNAME
WORKDIR /home/$USERNAME
```
我创建了一个普通用户, 并且安装了 openssh-client 和 openssh-server, 我应该如何修改 Dockerfile, 确保可以启动 ssh 服务, 并且普通用户 lanzhiwang 可以使用 ssh 进行登录

要确保容器启动时能够自动运行 SSH 服务, 并且允许创建的普通用户 `lanzhiwang` 通过 SSH 成功登录, 我们需要解决以下几个关键问题:

1. 设置用户密码或配置 SSH 密钥: 当前 Dockerfile 仅创建了用户, 但没有为其设置密码或配置免密登录的公钥, SSH 默认会拒绝空密码登录.
2. 启动 SSH 服务(sshd): `sshd` 必须以 `root` 权限运行, 以便监听 22 端口并处理用户登录时的身份切换(Privilege Separation).
3. 暴露端口: 在 Dockerfile 中声明 `EXPOSE 22`.

针对这些需求, 这里提供两种修改方案. 方案一(推荐)是容器运行的主进程为 `root`(用于启动 `sshd` 服务), 但你通过 SSH 登录时依然是 `lanzhiwang` 用户.

---

### 方案一: 标准方案(推荐)

容器默认以 `root` 用户运行并启动 `sshd`. 由于 `sshd` 会在登录成功后自动切换到对应用户的 shell, 你使用 `lanzhiwang` 登录后, 依然是普通用户身份.

#### 修改后的 Dockerfile

```dockerfile
FROM golang:1.24.1

ARG USERNAME=lanzhiwang
ARG USER_UID=1000
ARG USER_GID=$USER_UID
# 定义一个默认密码(建议在构建时通过 --build-arg 覆盖)
ARG USER_PASSWORD=lanzhiwang_password

############################## tools
RUN set -eux; \
	apt-get update; \
	apt-get install -y --no-install-recommends ca-certificates tree iputils-ping vim wget curl iproute2 iptables lsof dnsutils sudo openssh-client openssh-server; \
	rm -rf /var/lib/apt/lists/*; \
	echo "alias ll='ls -alFi --color=auto'" >> /etc/bash.bashrc; \
	groupadd --gid $USER_GID $USERNAME; \
	useradd --uid $USER_UID --gid $USER_GID -m -s /bin/bash $USERNAME; \
	# [修改 1]为普通用户设置密码
	echo "$USERNAME:$USER_PASSWORD" | chpasswd; \
	echo "$USERNAME ALL=(root) NOPASSWD:ALL" > /etc/sudoers.d/$USERNAME; \
	chmod 0440 /etc/sudoers.d/$USERNAME; \
	mkdir -p /run/sshd; \
	chmod 0755 /run/sshd
############################## tools

# [修改 2]声明暴露 22 端口
EXPOSE 22

# [修改 3]虽然你定义了 USER $USERNAME, 但为了让 sshd 顺利启动,
# 建议将默认容器运行用户设为 root, 或者在启动命令中切换.
# 如果你希望默认进入容器依然是 lanzhiwang 的工作目录, 可以保留 WORKDIR
WORKDIR /home/$USERNAME

# [修改 4]默认启动命令: 运行 sshd 守护进程
CMD ["/usr/sbin/sshd", "-D"]
```

---

### 方案二: 保持容器默认用户为普通用户

如果你强烈希望在不通过 SSH、而是通过 `docker run -it` 或 `docker exec` 直接进入容器时, 默认身份也是 `lanzhiwang`, 那么可以保持 `USER $USERNAME`.

此时, 由于 `lanzhiwang` 拥有无密码 `sudo` 权限, 我们可以利用 `sudo` 在启动时拉起 `sshd`.

#### 修改后的 Dockerfile

```dockerfile
FROM golang:1.24.1

ARG USERNAME=lanzhiwang
ARG USER_UID=1000
ARG USER_GID=$USER_UID
ARG USER_PASSWORD=lanzhiwang_password

############################## tools
RUN set -eux; \
	apt-get update; \
	apt-get install -y --no-install-recommends ca-certificates tree iputils-ping vim wget curl iproute2 iptables lsof dnsutils sudo openssh-client openssh-server; \
	rm -rf /var/lib/apt/lists/*; \
	echo "alias ll='ls -alFi --color=auto'" >> /etc/bash.bashrc; \
	groupadd --gid $USER_GID $USERNAME; \
	useradd --uid $USER_UID --gid $USER_GID -m -s /bin/bash $USERNAME; \
	# [修改 1]为普通用户设置密码
	echo "$USERNAME:$USER_PASSWORD" | chpasswd; \
	echo "$USERNAME ALL=(root) NOPASSWD:ALL" > /etc/sudoers.d/$USERNAME; \
	chmod 0440 /etc/sudoers.d/$USERNAME; \
	mkdir -p /run/sshd; \
	chmod 0755 /run/sshd
############################## tools

EXPOSE 22

USER $USERNAME
WORKDIR /home/$USERNAME

# [修改 2]由于当前是普通用户, 必须使用 sudo 启动 sshd
CMD ["sudo", "/usr/sbin/sshd", "-D"]
```

---

### 如何构建与运行

1. 构建镜像(可以传入自定义密码):

   ```bash
   docker build --build-arg USER_PASSWORD="YourSecurePassword" -t my-go-ssh-image .
   ```

2. 启动容器(将容器内的 22 端口映射到宿主机的端口, 例如 2222):

   ```bash
   docker run -d -p 2222:22 --name my-ssh-container my-go-ssh-image
   ```

3. 使用 SSH 登录:
   在宿主机或其他机器上运行:

   ```bash
   ssh lanzhiwang@localhost -p 2222
   # 接着输入你设置的密码(例如 YourSecurePassword)即可成功登录
   ```

---

### 进阶优化: 使用 SSH 密钥登录(更安全, 免密)

如果你不想在镜像中写入明文密码, 或者希望更安全地使用 SSH 密钥登录, 可以在 Dockerfile 中为 `lanzhiwang` 配置公钥.

只需在 Dockerfile 的 `tools` 部分之后添加以下命令:

```dockerfile
# 切换到普通用户创建 .ssh 目录
USER $USERNAME
RUN mkdir -p /home/$USERNAME/.ssh && chmod 700 /home/$USERNAME/.ssh

# 将宿主机的公钥(例如 id_rsa.pub)复制到容器中
# 确保在 docker build 的同级目录下存在 id_rsa.pub
COPY --chown=$USERNAME:$USERNAME id_rsa.pub /home/$USERNAME/.ssh/authorized_keys
RUN chmod 600 /home/$USERNAME/.ssh/authorized_keys

# 如果使用的是方案一, 最后记得切回 root
USER root
CMD ["/usr/sbin/sshd", "-D"]
```

------------------------------------------------------------------------------------------------------

Q: 假设你是一位精通 docker 的高级开发工程师, 现在我有如下 Dockerfile
```
FROM golang:1.24.1

ARG USERNAME=lanzhiwang
ARG USER_UID=1000
ARG USER_GID=$USER_UID
ARG USER_PASSWORD=lanzhiwang

############################## tools
RUN set -eux; \
	apt-get update; \
	apt-get install -y --no-install-recommends ca-certificates tree iputils-ping vim wget curl iproute2 iptables lsof dnsutils sudo openssh-client openssh-server; \
	rm -rf /var/lib/apt/lists/*; \
	echo "alias ll='ls -alFi --color=auto'" >> /etc/bash.bashrc; \
	groupadd --gid $USER_GID $USERNAME; \
	useradd --uid $USER_UID --gid $USER_GID -m -s /bin/bash $USERNAME; \
	echo "$USERNAME:$USER_PASSWORD" | chpasswd; \
	echo "$USERNAME ALL=(root) NOPASSWD:ALL" > /etc/sudoers.d/$USERNAME; \
	chmod 0440 /etc/sudoers.d/$USERNAME; \
	mkdir -p /run/sshd; \
	chmod 0755 /run/sshd
############################## tools

EXPOSE 22

USER $USERNAME
WORKDIR /home/$USERNAME

CMD ["sudo", "/usr/sbin/sshd"]

```
我利用这个 Dockerfile 构建镜像之后, 我用如下命令启动容器
```
docker run -ti --rm \
--entrypoint /usr/bin/env \
-v /Users/huzhi/work/code/py_code/higress:/higress \
-p 0.0.0.0:2022:22 \
--name golang-ssh-server \
lanzhiwang/golang:sha-1f4a07c bash
```
容器可以正常启动, 并且我也可以使用普通用户 ssh 登录到容器中, 但是如果我使用下面的命令启动容器
```
docker run -ti -d \
-v /Users/huzhi/work/code/py_code/higress:/higress \
-p 0.0.0.0:2022:22 \
--name golang-ssh-server \
lanzhiwang/golang:sha-1f4a07c
```
容器马上就会 exit, 并且我看到任何日志
请问这是什么原因, 应该如何解决

这是一个非常经典且容易遇到的 Docker 容器生命周期问题.

### 原因分析

容器立刻退出且没有日志, 主要是因为 `sshd` 默认是以守护进程(Daemon)模式在后台运行的, 而 Docker 容器的生命周期与它的主进程(PID 1)绑定.

1. 默认行为:
   在您的 Dockerfile 中, 默认启动命令是 `CMD ["sudo", "/usr/sbin/sshd"]`.
   当容器启动时, `sudo` 启动了 `/usr/sbin/sshd`. 由于没有指定前台运行参数, `sshd` 启动后会立即 fork 到后台运行(脱离终端).
   一旦 `sshd` 转入后台, 作为主进程的 `sudo` 认为任务已经执行完毕, 随即退出.
   PID 1 进程退出, Docker 容器便会立刻停止运行. 因为它是正常退出的(exit code 0), 所以可能看起来没有任何报错日志.

2. 为什么第一种方式可以运行:
   在第一种方式中, 您通过 `--entrypoint /usr/bin/env` 配合 `bash` 覆盖了默认的 `CMD`, 并且使用了 `-ti`(交互式终端)参数.
   此时容器的主进程(PID 1)是 `bash`. 只要您的 `bash` 终端没有关闭, 容器就会一直保持运行状态. 即使您在终端里手动启动了 `sshd`(它退到了后台), 由于 `bash` 依然在运行, 容器就不会退出, 因此您可以通过 SSH 登录.

---

### 解决方案

解决这个问题非常简单, 只需要强制 `sshd` 在前台运行, 不让它退到后台.

#### 1. 修改 Dockerfile 中的 `CMD`

将 `sshd` 的启动参数加上 `-D`. `-D` 参数的作用是让 `sshd` 在前台运行(do not daemonize), 这样它就会作为容器的主进程一直存在, 容器就不会退出了.

请将 Dockerfile 的最后一行修改为:

```dockerfile
# 加上 -D 参数, 确保 sshd 在前台运行
CMD ["sudo", "/usr/sbin/sshd", "-D"]
```

#### 2. 重新构建并运行

修改 Dockerfile 后, 重新构建您的镜像:

```bash
docker build -t lanzhiwang/golang:sha-1f4a07c .
```

然后, 使用您原先的后台运行命令启动容器:

```bash
docker run -d \
-v /Users/huzhi/work/code/py_code/higress:/higress \
-p 0.0.0.0:2022:22 \
--name golang-ssh-server \
lanzhiwang/golang:sha-1f4a07c
```
(注: 由于您使用了 `-d` 后台运行, 通常不需要再加 `-ti` 参数, 直接 `-d` 即可.)

此时容器会一直保持 `Up` 状态, 并且您可以使用 `docker logs golang-ssh-server` 查看到 SSH 服务的连接日志.

------------------------------------------------------------------------------------------------------

Q: 假设你是一位精通 docker 的高级开发工程师, 现在我有如下 Dockerfile
```
FROM golang:1.24.1

ARG USERNAME=lanzhiwang
ARG USER_UID=1000
ARG USER_GID=$USER_UID
ARG USER_PASSWORD=lanzhiwang

############################## tools
RUN set -eux; \
	apt-get update; \
	apt-get install -y --no-install-recommends ca-certificates tree iputils-ping vim wget curl iproute2 iptables lsof dnsutils sudo openssh-client openssh-server; \
	rm -rf /var/lib/apt/lists/*; \
	echo "alias ll='ls -alFi --color=auto'" >> /etc/bash.bashrc; \
	groupadd --gid $USER_GID $USERNAME; \
	useradd --uid $USER_UID --gid $USER_GID -m -s /bin/bash $USERNAME; \
	echo "$USERNAME:$USER_PASSWORD" | chpasswd; \
	echo "$USERNAME ALL=(root) NOPASSWD:ALL" > /etc/sudoers.d/$USERNAME; \
	chmod 0440 /etc/sudoers.d/$USERNAME; \
	mkdir -p /run/sshd; \
	chmod 0755 /run/sshd
############################## tools

EXPOSE 22

USER $USERNAME
WORKDIR /home/$USERNAME

CMD ["sudo", "/usr/sbin/sshd", "-D"]

```
构建镜像之后, 我使用如下命令启动容器
```
docker run -d \
-v /Users/huzhi/work/code/py_code/higress:/home/lanzhiwang/higress \
-p 0.0.0.0:2022:22 \
--name golang-ssh-server \
lanzhiwang/golang:sha-5147790
```
我使用 lanzhiwang ssh 登录到容器之后, 发现 go 命令本身不能使用了, 如下所示
```
lanzhiwang@afa2aa3e6337:~$ go version
-bash: go: command not found
lanzhiwang@afa2aa3e6337:~$
```
请问这是什么原因, 应该如何解决

这是一个在构建基于 SSH 的 Docker 开发环境时非常经典的"环境变量丢失"问题.

### 原因分析

在官方的 `golang:1.24.1` 镜像中, Go 的安装路径通常在 `/usr/local/go/bin`, 并且官方 Dockerfile 通过 `ENV PATH /usr/local/go/bin:$PATH` 将其写入了镜像中.

然而, 通过 SSH 登录容器时, `sshd` 服务会为用户启动一个全新的"登录 Shell"(Login Shell).
1. 登录 Shell 启动时会重置环境变量, 不会继承 Dockerfile 中通过 `ENV` 声明的环境变量.
2. 因此, 您登录后的 PATH 变量变回了系统的默认初始值(不包含 `/usr/local/go/bin` 和 `/go/bin`), 导致系统找不到 `go` 命令.

---

### 解决方案

在 Debian/Ubuntu(Go 官方镜像的底层系统)中, 解决此问题最标准、最优雅的方法是利用 `/etc/profile.d/` 目录.

当任何用户通过 SSH 登录(启动登录 Shell)时, 系统会自动加载 `/etc/profile`, 而 `/etc/profile` 会自动读取并执行 `/etc/profile.d/` 目录下所有以 `.sh` 结尾的脚本. 我们只需要在这个目录下创建一个配置脚本, 将 Go 的环境变量写入其中.

#### 修改后的 Dockerfile

您只需要在原有的 `RUN` 步骤中, 加上对 `/etc/profile.d/golang.sh` 的配置即可. 修改后的 Dockerfile 如下:

```dockerfile
FROM golang:1.24.1

ARG USERNAME=lanzhiwang
ARG USER_UID=1000
ARG USER_GID=$USER_UID
ARG USER_PASSWORD=lanzhiwang

############################## tools
RUN set -eux; \
	apt-get update; \
	apt-get install -y --no-install-recommends ca-certificates tree iputils-ping vim wget curl iproute2 iptables lsof dnsutils sudo openssh-client openssh-server; \
	rm -rf /var/lib/apt/lists/*; \
	echo "alias ll='ls -alFi --color=auto'" >> /etc/bash.bashrc; \
	# [关键修改]: 将 Go 相关的环境变量写入全局 profile 中, 确保 SSH 登录时自动加载
	echo 'export GOPATH=/go' > /etc/profile.d/golang.sh; \
	echo 'export PATH=$PATH:/usr/local/go/bin:/go/bin' >> /etc/profile.d/golang.sh; \
	groupadd --gid $USER_GID $USERNAME; \
	useradd --uid $USER_UID --gid $USER_GID -m -s /bin/bash $USERNAME; \
	echo "$USERNAME:$USER_PASSWORD" | chpasswd; \
	echo "$USERNAME ALL=(root) NOPASSWD:ALL" > /etc/sudoers.d/$USERNAME; \
	chmod 0440 /etc/sudoers.d/$USERNAME; \
	mkdir -p /run/sshd; \
	chmod 0755 /run/sshd
############################## tools

EXPOSE 22

USER $USERNAME
WORKDIR /home/$USERNAME

CMD ["sudo", "/usr/sbin/sshd", "-D"]
```

### 补充说明

1. `GOPATH=/go`: 官方 Go 镜像默认将 GOPATH 设为 `/go`. 我们在脚本中保留了这一设置, 并确保普通用户在登录后也能正确读写该目录(官方镜像默认已将 `/go` 目录权限设为 `777`, 所以普通用户无需担心权限问题).
2. `PATH` 追加:
   * `/usr/local/go/bin`: 这是 `go`、`gofmt` 等官方命令的所在路径.
   * `/go/bin`: 这是您后续使用 `go install` 安装的第三方工具(如 `dlv`、`golangci-lint` 等)的默认存放路径.

重新构建并启动容器后, 再次通过 SSH 登录 `lanzhiwang`, `go version` 命令即可正常使用.

------------------------------------------------------------------------------------------------------


------------------------------------------------------------------------------------------------------


------------------------------------------------------------------------------------------------------


------------------------------------------------------------------------------------------------------


------------------------------------------------------------------------------------------------------


------------------------------------------------------------------------------------------------------


------------------------------------------------------------------------------------------------------


------------------------------------------------------------------------------------------------------

