FROM python:3.12.13

ARG USERNAME=lanzhiwang
ARG USER_UID=1000
ARG USER_GID=$USER_UID

############################## tools
RUN set -eux; \
	apt-get update; \
	apt-get install -y --no-install-recommends ca-certificates tree iputils-ping vim wget curl iproute2 iptables lsof dnsutils sudo; \
	rm -rf /var/lib/apt/lists/*; \
	echo "alias ll='ls -alFi --color=auto'" >> /etc/bash.bashrc; \
	groupadd --gid $USER_GID $USERNAME; \
	useradd --uid $USER_UID --gid $USER_GID -m -s /bin/bash $USERNAME; \
	echo "$USERNAME ALL=(root) NOPASSWD:ALL" > /etc/sudoers.d/$USERNAME; \
	chmod 0440 /etc/sudoers.d/$USERNAME
############################## tools


############################## python package and tools
RUN set -eux; \
	pip install uv black claude-tap; \
	pip cache purge
############################## python package and tools


############################## node
ENV NODE_VERSION=24.15.0

RUN ARCH= && dpkgArch="$(dpkg --print-architecture)" \
	&& case "${dpkgArch##*-}" in \
	amd64) ARCH='x64';; \
	ppc64el) ARCH='ppc64le';; \
	s390x) ARCH='s390x';; \
	arm64) ARCH='arm64';; \
	armhf) ARCH='armv7l';; \
	i386) ARCH='x86';; \
	*) echo "unsupported architecture"; exit 1 ;; \
	esac \
	# use pre-existing gpg directory, see https://github.com/nodejs/docker-node/pull/1895#issuecomment-1550389150
	&& export GNUPGHOME="$(mktemp -d)" \
	# gpg keys listed at https://github.com/nodejs/node#release-keys
	&& set -ex \
	&& for key in \
	5BE8A3F6C8A5C01D106C0AD820B1A390B168D356 \
	DD792F5973C6DE52C432CBDAC77ABFA00DDBF2B7 \
	CC68F5A3106FF448322E48ED27F5E38D5B0A215F \
	8FCCA13FEF1D0C2E91008E09770F7A9A5AE15600 \
	890C08DB8579162FEE0DF9DB8BEAB4DFCF555EF4 \
	C82FA3AE1CBEDC6BE46B9360C43CEC45C17AB93C \
	108F52B48DB57BB0CC439B2997B01419BD92F80A \
	A363A499291CBBC940DD62E41F10027AF002F8B0 \
	; do \
	{ gpg --batch --keyserver hkps://keys.openpgp.org --recv-keys "$key" && gpg --batch --fingerprint "$key"; } || \
	{ gpg --batch --keyserver keyserver.ubuntu.com --recv-keys "$key" && gpg --batch --fingerprint "$key"; } ; \
	done \
	&& curl -fsSLO --compressed "https://nodejs.org/dist/v$NODE_VERSION/node-v$NODE_VERSION-linux-$ARCH.tar.xz" \
	&& curl -fsSLO --compressed "https://nodejs.org/dist/v$NODE_VERSION/SHASUMS256.txt.asc" \
	&& gpg --batch --decrypt --output SHASUMS256.txt SHASUMS256.txt.asc \
	&& gpgconf --kill all \
	&& rm -rf "$GNUPGHOME" \
	&& grep " node-v$NODE_VERSION-linux-$ARCH.tar.xz\$" SHASUMS256.txt | sha256sum -c - \
	&& tar -xJf "node-v$NODE_VERSION-linux-$ARCH.tar.xz" -C /usr/local --strip-components=1 --no-same-owner \
	&& rm "node-v$NODE_VERSION-linux-$ARCH.tar.xz" SHASUMS256.txt.asc SHASUMS256.txt \
	&& ln -s /usr/local/bin/node /usr/local/bin/nodejs \
	# smoke tests
	&& node --version \
	&& npm --version \
	&& rm -rf /tmp/*

ENV YARN_VERSION=1.22.22

RUN set -ex \
	# use pre-existing gpg directory, see https://github.com/nodejs/docker-node/pull/1895#issuecomment-1550389150
	&& export GNUPGHOME="$(mktemp -d)" \
	&& for key in \
	6A010C5166006599AA17F08146C2130DFD2497F5 \
	; do \
	{ gpg --batch --keyserver hkps://keys.openpgp.org --recv-keys "$key" && gpg --batch --fingerprint "$key"; } || \
	{ gpg --batch --keyserver keyserver.ubuntu.com --recv-keys "$key" && gpg --batch --fingerprint "$key"; } ; \
	done \
	&& curl -fsSLO --compressed "https://yarnpkg.com/downloads/$YARN_VERSION/yarn-v$YARN_VERSION.tar.gz" \
	&& curl -fsSLO --compressed "https://yarnpkg.com/downloads/$YARN_VERSION/yarn-v$YARN_VERSION.tar.gz.asc" \
	&& gpg --batch --verify yarn-v$YARN_VERSION.tar.gz.asc yarn-v$YARN_VERSION.tar.gz \
	&& gpgconf --kill all \
	&& rm -rf "$GNUPGHOME" \
	&& mkdir -p /opt \
	&& tar -xzf yarn-v$YARN_VERSION.tar.gz -C /opt/ \
	&& ln -s /opt/yarn-v$YARN_VERSION/bin/yarn /usr/local/bin/yarn \
	&& ln -s /opt/yarn-v$YARN_VERSION/bin/yarnpkg /usr/local/bin/yarnpkg \
	&& rm yarn-v$YARN_VERSION.tar.gz.asc yarn-v$YARN_VERSION.tar.gz \
	# smoke test
	&& yarn --version \
	&& rm -rf /tmp/*

RUN set -ex && npm install -g pnpm@10 && pnpm --version && npm cache clean --force
############################## node


############################## kt-connect
ARG KTCTL_VERSION=0.3.7

RUN set -eux \
	&& OS=$(uname -s) \
	&& RAW_ARCH=$(uname -m) \
	&& case "${RAW_ARCH}" in \
	x86_64) ARCH="x86_64" ;; \
	aarch64) ARCH="arm_64" ;; \
	i686 | i386) ARCH="i386" ;; \
	*) echo "Unsupported architecture: ${RAW_ARCH}" && exit 1 ;; \
	esac \
	&& FILENAME="ktctl_${KTCTL_VERSION}_${OS}_${ARCH}.tar.gz" \
	&& TEMP_DIR=$(mktemp -d) \
	&& cd ${TEMP_DIR} \
	&& curl -OL "https://github.com/alibaba/kt-connect/releases/download/v${KTCTL_VERSION}/${FILENAME}" \
	&& curl -OL "https://github.com/alibaba/kt-connect/releases/download/v${KTCTL_VERSION}/v${KTCTL_VERSION}_checksums.txt" \
	&& grep "${FILENAME}" v${KTCTL_VERSION}_checksums.txt | sha256sum -c - \
	&& tar zxf "${FILENAME}" \
	&& mv ktctl /usr/local/bin/ktctl \
	&& cd / \
	&& rm -rf ${TEMP_DIR} \
	&& ktctl --version
############################## kt-connect


############################## kubectl
ARG KUBECTL_VERSION=v1.32.1

RUN set -eux; \
	dpkgArch="$(dpkg --print-architecture)"; \
	case "${dpkgArch##*-}" in \
	amd64) ARCH='amd64' ;; \
	arm64) ARCH='arm64' ;; \
	armhf) ARCH='arm' ;; \
	ppc64el) ARCH='ppc64le' ;; \
	s390x) ARCH='s390x' ;; \
	*) echo "unsupported architecture for kubectl"; exit 1 ;; \
	esac; \
	# 下载二进制文件
	curl -fsSL -o /usr/local/bin/kubectl "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/${ARCH}/kubectl"; \
	# 获取对应的 sha256 校验和并进行校验
	checksum=$(curl -fsSL "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/${ARCH}/kubectl.sha256"); \
	echo "${checksum}  /usr/local/bin/kubectl" | sha256sum -c -; \
	# 赋予可执行权限
	chmod +x /usr/local/bin/kubectl; \
	# 基础功能验证
	kubectl version --client
############################## kubectl


############################## golang
ENV GO_VERSION=1.26.3

RUN set -eux; \
	dpkgArch="$(dpkg --print-architecture)"; \
	case "${dpkgArch##*-}" in \
	amd64) \
	GO_ARCH='amd64'; \
	GO_SHA256='2b2cfc7148493da5e73981bffbf3353af381d5f93e789c82c79aff64962eb556'; \
	;; \
	arm64) \
	GO_ARCH='arm64'; \
	GO_SHA256='9d89a3ea57d141c2b22d70083f2c8459ba3890f2d9e818e7e933b75614936565'; \
	;; \
	i386) \
	GO_ARCH='386'; \
	GO_SHA256='0ef3626a149b5811c813838c62b7d6618d03ea36047b32c90b0e4851cc42b1fa'; \
	;; \
	armhf) \
	GO_ARCH='armv6l'; \
	GO_SHA256='d44133d4c66b1451a1e247da26db7716f76a081c0169a75e6c84e1871e394320'; \
	;; \
	*) echo "unsupported architecture for golang"; exit 1 ;; \
	esac; \
	# 下载对应的架构包
	curl -fsSL -o go.tgz "https://go.dev/dl/go${GO_VERSION}.linux-${GO_ARCH}.tar.gz"; \
	# 校验 SHA256 Checksum
	echo "${GO_SHA256} *go.tgz" | sha256sum -c -; \
	# 解压并清理
	tar -C /usr/local -xzf go.tgz; \
	rm go.tgz

ENV PATH="/usr/local/go/bin:/home/${USERNAME}/go/bin:/root/go/bin:${PATH}"
############################## golang


USER $USERNAME
WORKDIR /home/$USERNAME
