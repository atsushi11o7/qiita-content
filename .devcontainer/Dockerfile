FROM node:24-trixie-slim

# ビルド中の対話プロンプト抑止 & npm のアップデート通知抑止
ENV DEBIAN_FRONTEND=noninteractive \
    npm_config_update_notifier=false

WORKDIR /workspace

# Claude Code のインストールスクリプト取得用 & GitHub CLI apt リポジトリ取得用
RUN apt-get update \
 && apt-get install -y --no-install-recommends curl ca-certificates \
 && rm -rf /var/lib/apt/lists/*

# GitHub CLI
RUN curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg \
      | dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg \
 && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" \
      | tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
 && apt-get update \
 && apt-get install -y --no-install-recommends gh \
 && rm -rf /var/lib/apt/lists/*

USER node

# Claude Code
RUN curl -fsSL https://claude.ai/install.sh | bash

# Qiita CLI のプレビューサーバ用
EXPOSE 8888

CMD ["bash"]
