---
layout: drafts
title: NVM配置ndoejs环境
tags: node
categories:
  - js
date: 2022-04-11 17:45:00
updated: 2022-04-11 17:45:00
---


## Requirement

1. 发行版本的node版本过低
2. 官方apt源需要频繁手动更新
3. 支持多个版本的node间切换

## Step

1. 安装脚本

```bash
# install 
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash

```

2. 配置shell

在`.zshrc`中配置插件`nvm`, 并移除安装脚本append文末的配置

3. 安装nodejs

```
nvm install --lts
```

4. 安装最新版本npm

```
nvm install-latest-npm
```




