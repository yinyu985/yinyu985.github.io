---
title: 使用 Docker 运行 Hexo 博客
slug: using-docker-to-run-hexo-blog
tags: [Docker, Hexo]
date: 2023-01-07T18:24:34+08:00
---

原本 Hexo 是放在 Mac 上的，但是只是配过两次，对其了解不够深刻不敢乱动，这次我又回来了，近乎破釜沉舟，因为我把原本的博客毁得差不多了，只能重新搭，并且这次下定决心采用 Docker 运行，技术要用起来，才能理解的更深刻。<!--more-->

```bash
cat >Dockerfile<<EOF
FROM node:13.14-alpine3.10
WORKDIR /blog
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories && apk add bash git openssh 
RUN apk add tzdata && cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
&& echo "Asia/Shanghai" > /etc/timezone && apk del tzdata
RUN \
npm config set registry https://registry.npm.taobao.org \
&& npm install hexo-cli -g \
&& hexo init . \
&& npm install \
&& npm install hexo-server --save \
&& npm install hexo-abbrlink --save \
&& npm install hexo-deployer-git --save \
&& npm install hexo-symbols-count-time --save \
&& npm install hexo-asset-image --save \
&& npm install hexo-blog-encrypt --save \
&& npm install hexo-generator-searchdb --save \
&& git clone https://github.com/next-theme/hexo-theme-next.git themes/next \
&& sed -i "s/theme: landscape/theme: next/g" _config.yml \
&& sed -i "s@permalink: :year/:month/:day/:title/@permalink: posts/:abbrlink.html@g" _config.yml \
&& sed -i "s@type: ''@type: 'git'@g" _config.yml \
&& echo -e '  repo: git@github.com:yinyu985/yinyu985.github.io.git\n\
  branch: master\n'>>_config.yml \
&& rm -rf _config.landscape.yml \
&& git init \
&& git config --global user.name "yinyu985" \
&& git config --global user.email "yinyu985@gmail.com"
EXPOSE 4000
EOF
```

## Dockerfile

上面的 Dockerfile 安装了 Hexo，安装了 git、bash，设置了时区，修改了默认主题，修改了链接生成方式，通过 hexo-abbrlink 生成链接，这样中文标题的文章就不用转成一大串了。

然后设置了 Hexo 的部署方式为 git，注意仓库链接建议用 git，不要写 http 链接，不然每次 deploy 都要求验证账户。好久没动它，发现现在 GitHub 推代码还要在个人设置的开发者界面生成一个 token。

在 Dockerfile 中使用 echo 向文本中输入多行，搜了一下，别人是怎么做的，看了下，都是千篇一律的使用 `\n` 进行换行，没搜到其他办法，就这么用吧。结果写进去，构建，运行，文件拷出来，没生效。echo 加个参数好吗？带你学习一下 echo 怎么用吧！

```bash
echo(选项)(参数)
选项
-e：启用转义字符。
-E：不启用转义字符（默认）。
-n：结尾不换行。
使用 -e 选项时，若字符串中出现以下字符，则特别加以处理，而不会将它当成一般文字输出：

\a 发出警告声；
\b 删除前一个字符；
\c 不产生进一步输出（\c 后面的字符不会输出）；
\f 换行但光标仍旧停留在原来的位置；
\n 换行且光标移至行首；
\r 光标移至行首，但不换行；
\t 插入 tab；
\v 与 \f 相同；
\\ 插入 \ 字符；
\nnn 插入 nnn（八进制）所代表的 ASCII 字符。
```

实践出真知啊，网上搜的文章，都没有 -e 参数，也不知道他是怎么支持 `\n` 转义的，可能系统不同？

## 构建

```bash
docker build -t yinyu985/hexo:latest .
```

构建镜像啦，将第一步 Dockerfile 输出到一个空目录，然后在这个空目录里，命令最后的 `.` 就是在当前目录。

```bash
docker run -itd \
-p 4000:4000 \
--name 'hexo' \
yinyu985/hexo:latest
```

随便启动一下，啥也不挂载，其实端口也是多余的，但是千金难买爷乐意。

```bash
docker cp hexo:/blog ~
```

将容器里面的 `/blog` 目录拷贝到本 Mac 的家目录。

```bash
docker rm -f hexo
```

然后得把这个容器删掉，不然影响后面干正事。

## 运行

```bash
docker run -itd \
-p 4000:4000 \
--name 'hexo' \
-v ~/blog/themes:/blog/themes \
-v ~/blog/source:/blog/source \
-v ~/blog/source/_posts:/blog/source/_posts \
-v ~/blog/_config.yml:/blog/_config.yml \
-v ~/.ssh:/root/.ssh \
yinyu985/hexo:latest
```

这时候就是干正事，刚才虽然从容器拷贝出来，但是不是全都需要，基本上挂载的这些就够了。  
关于 Hexo 主要设置了啥在此记录一下，~~这玩意儿，几百年不动，怕忘了~~。

## 配置

> Hexo 主配置文件

### 设置中文

```yaml
# Site
title: yinyu985's blog
subtitle: ''
description: ''
keywords:
author: yinyu985
language: zh-CN
timezone: ''
```

### 设置主题

```yaml
# Extensions
## Plugins: https://hexo.io/plugins/
## Themes: https://hexo.io/themes/
theme: next
```

### 设置部署方式

```yaml
# Deployment
## Docs: https://hexo.io/docs/one-command-deployment
deploy:
  type: 'git'
  repo: git@github.com:yinyu985/yinyu985.github.io.git
  branch: master
```

### 设置链接格式

```yaml
# URL
## Set your site url here. For example, if you use GitHub Page, set url as 'https://username.github.io/project'
url: http://example.com
permalink: posts/:abbrlink.html
permalink_defaults:
```

> Next 主题配置文件

### 设置字数统计

```yaml
# Post wordcount display settings
# Dependencies: https://github.com/next-theme/hexo-word-counter
symbols_count_time:
  separated_meta: true
  item_text_post: true
  item_text_total: false
  awl: 4
  wpm: 275
```

没了，剩下的都是一些无关痛痒的样式修改，后续对 Next 主题配置优化再更新。

```bash
function h() {
  docker exec -it hexo hexo $1 $2
}
function n() {
  docker exec -it hexo hexo new $1 $2
}
alias c='docker exec -it hexo hexo clean'
alias g='docker exec -it hexo hexo generate'
alias d='docker exec -it hexo hexo deploy'
alias s='docker exec -it hexo hexo server'
alias gkd='docker exec -it hexo hexo clean && docker exec -it hexo hexo generate && docker exec -it hexo hexo deploy'
```