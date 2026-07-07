
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


------------------------------------------------------------------------------------------------------


------------------------------------------------------------------------------------------------------


------------------------------------------------------------------------------------------------------


------------------------------------------------------------------------------------------------------


------------------------------------------------------------------------------------------------------


------------------------------------------------------------------------------------------------------


------------------------------------------------------------------------------------------------------


------------------------------------------------------------------------------------------------------


------------------------------------------------------------------------------------------------------

