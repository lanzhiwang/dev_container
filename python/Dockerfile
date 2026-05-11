FROM python:3.12.13

RUN set -eux; \
	apt-get update; \
	apt-get install -y --no-install-recommends tree iputils-ping vim wget curl; \
	echo "alias ll='ls -alFi --color=auto'" >> /etc/profile; \
	source /etc/profile; \
	rm -rf /var/lib/apt/lists/*
