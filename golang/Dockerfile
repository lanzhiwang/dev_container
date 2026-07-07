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
