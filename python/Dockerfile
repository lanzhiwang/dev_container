FROM python:3.12.13

RUN set -eux; \
	apt-get update; \
	apt-get install -y --no-install-recommends tree iputils-ping vim wget curl; \
	alias ll='ls -l --color=auto'; \
	rm -rf /var/lib/apt/lists/*
