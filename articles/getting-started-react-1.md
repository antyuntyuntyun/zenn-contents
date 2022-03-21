---
title: "今更React入門①"
emoji: "🦁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "frontend", "docker"]
published: true
---

## 今更 React 入門

簡単な Web アプリを構築する必要があり、依頼するとお高いので自前で作ることにしました。
Vue や Nuxt は以前の業務/趣味の Web アプリで使っていたものの React は真面目に触る機会がいままでなかったので、React で構築しようと思い立ちました。
というわけで今更 React 入門します。

## つくったもの

とりあえず Docker コンテナ上での React 開発環境をセットアップしました。VScode 設定も盛り込んで作成しました。

https://github.com/antyuntyuntyun/react-template

## プロジェクト作成コマンド

以下記事参照させて頂きました。とりあえず `npx create-react-app react_app`といった形でプロジェクト作成コマンド使えば React 始められそう。

https://zenn.dev/web_tips/articles/abad1a544f3643

## lint 設定

以下記事参照させて頂きました。npx create で導入されたもの以外で lint に必要なパッケージをインストール。
最近好きな gimoji-cli も紛れ込ませることに。

```bash
yarn add -D eslint prettier eslint-config-prettier
yarn add -D typescript @typescript-eslint/{parser,eslint-plugin}
yarn add -D @types/{react,react-dom}
yarn add -D eslint-plugin-{react,react-hooks}
yarn add -D gitmoji-cli
```

https://zenn.dev/sprout2000/articles/9f20902d394aa2

## とりあえずコンテナ化

- create-react-app で構成されるものをプロジェクトルート直下に展開
  - lock ファイルが package-lock.json でなく yarn.lock になるように調整
- コンテナ作成・起動時に`yarn install`されるようにし、コンテナを開いて`yarn start`すればすぐに開発できるように
- 開発環境コンテナでは ssh/git 設定や開発用のシェル/ターミナル設定を反映、本番環境では必要な設定のみ反映
  - 本番用設定ファイルは Dockerfile.prod と docker-compose.prod.yml
- VScode 拡張機能設定を`.devcontainer/devcontainer.json`で設定しておき、コンテナを立ち上げたときに反映されるように

https://github.com/antyuntyuntyun/react-template

### Dockerfile

```docker
ARG NODE_VERSION=16
FROM node:${NODE_VERSION}

ENV DEBIAN_FRONTEND=noninteractive
ENV TZ=Asia/Tokyo
RUN echo $TZ > /etc/timezone

# workdir
ARG WORKDIR=/frontend
RUN mkdir -p ${WORKDIR}
# Give owner rights to the current user
RUN chown -Rh $USERNAME:$USERNAME ${WORKDIR}
WORKDIR ${WORKDIR}

# Or your actual UID, GID on Linux if not the default 1000
ARG USERNAME=developer
ARG USER_UID=1001
ARG USER_GID=$USER_UID
ENV HOME /home/$USERNAME

# change default shell
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
RUN chsh -s /bin/bash

# Increase timeout for apt-get to 300 seconds
RUN /bin/echo -e "\n\
    Acquire::http::Timeout \"300\";\n\
    Acquire::ftp::Timeout \"300\";" >> /etc/apt/apt.conf.d/99timeout

# Configure apt and install packages
RUN rm -rf /var/lib/apt/lists/* \
    && apt-get update \
    && apt-get -y --no-install-recommends install sudo vim tzdata less graphviz

# install gh
RUN wget https://github.com/cli/cli/releases/download/v1.0.0/gh_1.0.0_linux_amd64.deb
RUN dpkg -i gh_*_linux_amd64.deb
RUN chmod g+rwx -R /usr/local/bin/
RUN gh --version

# Create a non-root user to use if preferred - see https://aka.ms/vscode-remote/containers/non-root-user.
RUN groupadd --gid $USER_GID $USERNAME \
    && useradd -s /bin/bash --uid $USER_UID --gid $USER_GID -m $USERNAME \
    # Add sudo support for non-root user
    && echo $USERNAME ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/$USERNAME \
    && chmod 0440 /etc/sudoers.d/$USERNAME

# terminal setting
RUN wget --progress=dot:giga https://raw.githubusercontent.com/git/git/master/contrib/completion/git-completion.bash -O /home/$USERNAME/.git-completion.bash \
    && wget --progress=dot:giga https://raw.githubusercontent.com/git/git/master/contrib/completion/git-prompt.sh -O /home/$USERNAME/.git-prompt.sh \
    && chmod a+x /home/$USERNAME/.git-completion.bash \
    && chmod a+x /home/$USERNAME/.git-prompt.sh \
    && echo -e "\n\
    source ~/.git-completion.bash\n\
    source ~/.git-prompt.sh\n\
    export PS1='\\[\\e]0;\\u@\\h: \\w\\a\\]\${debian_chroot:+(\$debian_chroot)}\\[\\033[01;32m\\]\\u\\[\\033[00m\\]:\\[\\033[01;34m\\]\\w\\[\\033[00m\\]\\[\\033[1;30m\\]\$(__git_ps1)\\[\\033[0m\\] \\$ '\n\
    export EDITOR=vim\n\
    " >>  /home/$USERNAME/.bashrc

# Remove outdated yarn from /opt and install via package
# so it can be easily updated via apt-get upgrade yarn
RUN rm -rf /opt/yarn-* /usr/local/bin/{yarn,yarnpkg} \
    && apt-get install -y curl apt-transport-https lsb-release \
    && curl -sS https://dl.yarnpkg.com/$(lsb_release -is | tr '[:upper:]' '[:lower:]')/pubkey.gpg | apt-key add - 2>/dev/null \
    && echo "deb https://dl.yarnpkg.com/$(lsb_release -is | tr '[:upper:]' '[:lower:]')/ stable main" | tee /etc/apt/sources.list.d/yarn.list \
    && apt-get update \
    && apt-get -y install --no-install-recommends yarn

# Clean up
RUN apt-get autoremove -y \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/*
ENV DEBIAN_FRONTEND=dialog

# node package install setup
RUN mkdir -p ${WORKDIR}/node_modules
RUN chown -Rh $USERNAME:$USERNAME ${WORKDIR}/node_modules

# Set the default user / npm package install
USER $USERNAME
COPY package.json ${WORKDIR}
COPY yarn.lock ${WORKDIR}
RUN yarn install
```

### docker-compose.yml

```yml
version: "3"
services:
  frontend:
    build:
      context: .
      args:
        WORKDIR: /frontend
      dockerfile: Dockerfile
    volumes:
      - .:/frontend
      - /frontend/node_modules
      - ${USERPROFILE-~}/.ssh:/home/developer/.ssh
    image: frontend-image
    container_name: frontend-container
    ports:
      - "3000:3000"
    tty: true
```

#### devcontainer.json

```json
// For format details, see https://aka.ms/devcontainer.json. For config options, see the README at:
// https://github.com/microsoft/vscode-dev-containers/tree/v0.194.0/containers/docker-existing-dockerfile
{
  "name": "ReactContainer",
  // Sets the run context to one level up instead of the .devcontainer folder.
  // "context": "..",
  // Update the 'dockerFile' property if you aren't using the standard 'Dockerfile' filename.
  // "dockerFile": "../Dockerfile",
  "dockerComposeFile": ["../docker-compose.yml"],
  "service": "frontend",
  "workspaceFolder": "/frontend",
  "shutdownAction": "stopCompose",
  // Uncomment to connect as a non-root user if you've added one. See https://aka.ms/vscode-remote/containers/non-root.
  "remoteUser": "developer",
  // Set *default* container specific settings.json values on container create.
  "settings": {
    "files.eol": "\n",
    "editor.guides.bracketPairs": true,
    "editor.bracketPairColorization.enabled": true,
    "terminal.integrated.profiles.linux": {
      "bash": {
        "path": "/bin/bash"
      }
    },
    "[javascript]": {
      "editor.defaultFormatter": "esbenp.prettier-vscode",
      "editor.codeActionsOnSave": {
        "source.fixAll.eslint": true
      }
    },
    "[json]": {
      "editor.defaultFormatter": "esbenp.prettier-vscode",
      "editor.codeActionsOnSave": {
        "source.fixAll.eslint": true
      }
    },
    "[typescript]": {
      "editor.defaultFormatter": "esbenp.prettier-vscode",
      "editor.codeActionsOnSave": {
        "source.fixAll.eslint": true
      }
    },
    "[markdown]": {
      "editor.tabSize": 4,
      "editor.formatOnSave": true
    },
    "markdown-preview-enhanced.enableExtendedTableSyntax": true,
    "markdown-preview-enhanced.enableScriptExecution": true,
    "markdown-preview-enhanced.previewTheme": "github-light.css",
    "markdown.preview.breaks": true,
    "markdown.extension.orderedList.marker": "one",
    "markdown.extension.orderedList.autoRenumber": false,
    "markdown.extension.syntax.decorations": false,
    "markdown.extension.tableFormatter.enabled": false,
    // Add the IDs of extensions you want installed when the container is created.
    "extensions": [
      // base
      "vincaslt.highlight-matching-tag",
      "donjayamanne.githistory",
      "eamodio.gitlens",
      "mhutchie.git-graph",
      "gruntfuggly.todo-tree",
      "github.vscode-pull-request-github",
      "mutantdino.resourcemonitor",
      "wmaurer.change-case",
      "editorconfig.editorconfig",
      "mosapride.zenkaku",
      "davidanson.vscode-markdownlint",
      "VisualStudioExptTeam.vscodeintellicode",
      "ms-azuretools.vscode-docker",
      "redhat.vscode-yaml",
      "christian-kohler.path-intellisense",
      "esbenp.prettier-vscode",
      "pkief.material-icon-theme",
      "esbenp.prettier-vscode",
      "mutantdino.resourcemonitor",
      // front
      "dsznajder.es7-react-js-snippets",
      "dbaeumer.vscode-eslint",
      "christian-kohler.npm-intellisense",
      "ecmel.vscode-html-css",
      "ormulahendry.auto-close-tag",
      "formulahendry.auto-rename-tag",
      "zignd.html-css-class-completion"
    ],
    "postAttachCommand": "yarn install && yarn start"
    // Use 'forwardPorts' to make a list of ports inside the container available locally.
    // "forwardPorts": [],
    // Uncomment the next line to run commands after the container is created - for example installing curl.
    // "postCreateCommand": "apt-get update && apt-get install -y curl",
    // Uncomment when using a ptrace-based debugger like C++, Go, and Rust
    // "runArgs": [ "--cap-add=SYS_PTRACE", "--security-opt", "seccomp=unconfined" ],
    // Uncomment to use the Docker CLI from inside the container. See https://aka.ms/vscode-remote/samples/docker-from-docker.
    // "mounts": [ "source=/var/run/docker.sock,target=/var/run/docker.sock,type=bind" ],
  }
}
```

## 諦めたこと

### web アプリ起動自動化

コンテナ立ち上げたときに自動で`yarn start`するようにしたいができていない。
Dockerfile で CMD や ENTORYPOINT で起動すると、コンテナが閉じてしまうので指定していない。
苦し紛れに.devcontainer.json で postAttachCommand で yarn start 指定してみているものの効果なし。

### lint 設定値の理解

思考停止でとりあえず設定。Javascript しばらく触っていなかったので、これから触りながら色々思い出していこうと思いました。

## 最後に

本日はここまで。とりあえず開発環境設定はできた気がするので、アプリ開発に進みます。

## 追記：　コンテナの VScode 拡張機能が正常にインストールされない

よく見ると拡張機能が正常に機能しておらず、以下のようなメッセージが出ていた。
`This extension is disabled in this workspace because it is defined to run in the Remote Extension Host. vscode remote container`

下記リファレンスをみると、ローカルとリモートで競合がある場合には拡張機能が正常にインストールされないそうで、常にインストールしたい場合は異なるオプション指定が必要らしい.

https://code.visualstudio.com/docs/remote/containers

`"extensinos": []`ではなく`"remote.containers.defaultExtensions": []`で指定することで解決した。

### 修正した.devcontainer.json

```json
// For format details, see https://aka.ms/devcontainer.json. For config options, see the README at:
// https://github.com/microsoft/vscode-dev-containers/tree/v0.194.0/containers/docker-existing-dockerfile
{
  "name": "ReactContainer",
  // Sets the run context to one level up instead of the .devcontainer folder.
  // "context": "..",
  // Update the 'dockerFile' property if you aren't using the standard 'Dockerfile' filename.
  // "dockerFile": "../Dockerfile",
  "dockerComposeFile": ["../docker-compose.yml"],
  "service": "frontend",
  "workspaceFolder": "/frontend",
  "shutdownAction": "stopCompose",
  // Uncomment to connect as a non-root user if you've added one. See https://aka.ms/vscode-remote/containers/non-root.
  "remoteUser": "developer",
  // Set *default* container specific settings.json values on container create.
  "settings": {
    "files.eol": "\n",
    "editor.guides.bracketPairs": true,
    "editor.bracketPairColorization.enabled": true,
    "terminal.integrated.profiles.linux": {
      "bash": {
        "path": "/bin/bash"
      }
    },
    "[javascript]": {
      "editor.defaultFormatter": "esbenp.prettier-vscode",
      "editor.codeActionsOnSave": {
        "source.fixAll.eslint": true
      }
    },
    "[json]": {
      "editor.defaultFormatter": "esbenp.prettier-vscode",
      "editor.codeActionsOnSave": {
        "source.fixAll.eslint": true
      }
    },
    "[typescript]": {
      "editor.defaultFormatter": "esbenp.prettier-vscode",
      "editor.codeActionsOnSave": {
        "source.fixAll.eslint": true
      }
    },
    "[markdown]": {
      "editor.tabSize": 4,
      "editor.formatOnSave": true
    },
    "markdown-preview-enhanced.enableExtendedTableSyntax": true,
    "markdown-preview-enhanced.enableScriptExecution": true,
    "markdown-preview-enhanced.previewTheme": "github-light.css",
    "markdown.preview.breaks": true,
    "markdown.extension.orderedList.marker": "one",
    "markdown.extension.orderedList.autoRenumber": false,
    "markdown.extension.syntax.decorations": false,
    "markdown.extension.tableFormatter.enabled": false,
    // Add the IDs of extensions you want installed when the container is created.
    "remote.containers.defaultExtensions": [
      // base
      "vincaslt.highlight-matching-tag",
      "donjayamanne.githistory",
      "eamodio.gitlens",
      "mhutchie.git-graph",
      "gruntfuggly.todo-tree",
      "github.vscode-pull-request-github",
      "mutantdino.resourcemonitor",
      "wmaurer.change-case",
      "editorconfig.editorconfig",
      "mosapride.zenkaku",
      "davidanson.vscode-markdownlint",
      "VisualStudioExptTeam.vscodeintellicode",
      "ms-azuretools.vscode-docker",
      "redhat.vscode-yaml",
      "christian-kohler.path-intellisense",
      "esbenp.prettier-vscode",
      "pkief.material-icon-theme",
      "esbenp.prettier-vscode",
      "mutantdino.resourcemonitor",
      // front
      "dsznajder.es7-react-js-snippets",
      "dbaeumer.vscode-eslint",
      "christian-kohler.npm-intellisense",
      "ecmel.vscode-html-css",
      "ormulahendry.auto-close-tag",
      "formulahendry.auto-rename-tag",
      "zignd.html-css-class-completion"
    ],
    "postAttachCommand": "yarn install && yarn start"
    // Use 'forwardPorts' to make a list of ports inside the container available locally.
    // "forwardPorts": [],
    // Uncomment the next line to run commands after the container is created - for example installing curl.
    // "postCreateCommand": "apt-get update && apt-get install -y curl",
    // Uncomment when using a ptrace-based debugger like C++, Go, and Rust
    // "runArgs": [ "--cap-add=SYS_PTRACE", "--security-opt", "seccomp=unconfined" ],
    // Uncomment to use the Docker CLI from inside the container. See https://aka.ms/vscode-remote/samples/docker-from-docker.
    // "mounts": [ "source=/var/run/docker.sock,target=/var/run/docker.sock,type=bind" ],
  }
}
```

## 追記: Prettier が勝手にシングルクォーてションをダブルクォーテーションにする

setting.json に以下を追記して解決

```json
"prettier.singleQuote": true,
```
