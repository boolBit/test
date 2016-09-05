---
title: hexo与github集成搭建blog
tags: [hexo, git, blog]
categories: 教程
---

之前在csdn，博客园上写博客，想着一方面总结学过的知识，另一方面可以提高自己的写作总结能力。很多知识点理解运用并不难，但是把逻辑清楚的讲解给别人听，还是需要一定的技巧和经验。[Hexo](https://hexo.io/zh-cn/)是一款开源的Node.js静态博客框架，支持markdown语法，主题很丰富。之前在阿里云上搭过，可惜写了几篇没有坚持下来。痛下心痛决定结合github pages重新开始...

----------


###  配置环境
 - git & github （将页面推送到github pages，公司，家中分布式开发。。）
 - nodejs (用于安装HEXO)
 - markdown编辑器 （使用[sublimes3](http://www.sublimetext.com/3) + [MarkdownEditing](https://github.com/SublimeText-Markdown/MarkdownEditing)+[OmniMarkupPreviewer](https://github.com/timonwong/OmniMarkupPreviewer)）
 
###  hexo安装

```
$ npm install -g hexo-cli
$ hexo init <folder>
$ cd <folder>
$ npm install
```
- 安装完成后目录如下：
> .
 ├── _config.yml(网站全局配置信息，例如标题，关键词，配置文章、目录基本信息，选择主题，配置deploy信息等)
 ├── package.json （应用程序信息）
 ├── scaffolds（文章模板文件夹）
 ├── source （资源文件夹，markdown和html文件会被生成到public文件夹，其它会被拷贝过去）
 |   └── _posts （为一个页面目录，存放默认发表的文章）
 └── themes （存放主题插件和配置）

- 基本命令
  * new
  ``` hexo new <layout> pageName ```
    新建一篇文章。如果没有设置 layout 的话，默认使用 _config.yml 中的 default_layout           参数代替。如果标题包含空格的话，需用引号括起来。
  * server
 本地启动nodejs服务器，可通过url访问
  * generate
    生成静态页面和css等
  * publish
    生成草稿 （hexo --draft 显示草稿）
  * clean
    清除缓存文件 (db.json) 和已生成的静态文件 (public)。
  * deploy
    部署public目录下的文件到配置的对应目的地（ 参数-g 首先执行generate页面）

- 自定义主题
  选择一个心仪的[主题](https://hexo.io/themes/),进入主题github主页，按照教程进行相应配置即可。也可在此基础上进行开发。

- 部署到github
  编辑_config.yml文件

```
deploy:
    type: git
    repo: https://github.com/githubusername/githubusername.github.io.git （需要一个 用户名.github.io 为名称的github项目，用于deploy generate后的public目录。）
    branch: master
```
执行 `npm install hexo-deployer-git --save` 此外需要在github上配置了本机的ssh key。
```
hexo new "page" 创建一片新文章
hexo clean
hexo deploy -g
```
浏览器访问githubusername.github.io即可看到相应的页面。

- 绑定域名


