# 钟镇刚的个人博客仓库

## 概述
使用Hexo + GitHub的pages搭建了自己的个人博客

## 部署

### 本地部署

step1 先在HexoBlog目录执行静态文件文件生成命令

```bash
$ hexo generate 
```

step2 在本地启动服务器

```bash
$ hexo server
```

step3 访问`http://localhost:4000`地址就可以看见博客内容了。

### Github部署

step1 先在HexoBlog目录执行静态文件文件生成命令

```bash
$ hexo generate 
```

step2 部署静态文件到Github pages仓库。部署仓库地址在`_config.yml`文件的`deploy`项中有写。

```bash
$ hexo deploy
```

step3 访问githu的个人地址: https://zhongzhengang.github.io



## Github部署仓库信息

github部署仓库地址：git@github.com:zhongzhengang/zhongzhengang.github.io.git 分支：master

访问地址: http://zhongzhengang.github.io



## 配置信息

### 站点配置

根目录下的`_config.yml`为站点配置文件，主要配置的是Hexo相关的参数。



### 主题配置

主题配置文件是根目录下的`_config.<主题名称>.yml`，例如NexT主题的配置文件是`_config.next.yml`文件，其中主要配置的是主题相关的参数。
