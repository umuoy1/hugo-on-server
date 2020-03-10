---
title: "搭建Hugo博客，通过Netlify自动部署"
date: 2020-03-06T22:02:46+08:00
draft: false
comments: true
---

> 本文章基于Windows10，Hugo_extended_0.66.0，博客发布的流程是本地修改同步到github，netlify检测到push动作后自动发布 

## 一、下载并安装Hugo

- [Hugo下载地址](https://github.com/gohugoio/hugo/releases)

  本教程选择了![1583507935042.png](https://i.loli.net/2020/03/07/zT8oSZCOAib1q4J.png)安装，该版本增加了对sass的支持 。

  #### 第1步：安装Hugo

  下载解压，完成之后，把hugo.exe添加到全局变量```Path```。![1583508112162.png](https://i.loli.net/2020/03/07/LPQvA3RNwEGMJhm.png)在控制台中验证安装成功。

- 在Github上新建仓库hugo-on-netlify，并在D:\Blog目录下打开Gitbash，输入指令。

  > $ git clone  https://github.com/xxxxx/hugo-on-netlify
  >
  > $ hugo new site hugo-on-netlify --force

  此时便会在hugo-on-netlify文件夹里生成网站需要的文件，文件结构如下。

  > ```
  > myblog
  > ├── archetypes
  > │   └── default.md
  > ├── content
  > ├── data
  > ├── layouts
  > ├── static
  > ├── themes
  > └── config.toml
  > ```

#### 第2步：安装主题Meme

Hugo是没有默认主题的，这里选用主题[Meme]( https://github.com/reuixiy/hugo-theme-meme )。

> $ git clone  https://github.com/reuixiy/hugo-theme-meme.git themes/meme

然后替换 `config.toml` 为 [config.toml]( https://github.com/reuixiy/hugo-theme-meme/blob/master/config-examples/zh-cn/config.toml) ，可以在其中进行个性化设置。

#### 第3步：测试

创建测试页面

> $ hugo new posts/my-first-post.md
>
> $ hugo new about/_index.md

此时，基本工作已经完成了，使用

> $ hugo server -D

在浏览器里访问```http://localhost:1313/```，预览博客效果。



## 二、上传Github

之前我们已经新建了hugo-on-netlify仓库，下一步把博客上传

> $ git add .  #将所有文件添加到仓库里
>
> $ git commit -m "commit message"
>
> $ git push -u origin master

以上也是更新文章所需的操作，如果嫌麻烦，可以写一个bat脚本减轻工作量。

## 三、使用Netlify发布网站

官网[Netlify](https://www.netlify.com/)可以直接通过Github登录，非常方便。

#### 第1步：配置文件

首先在网站根目录下添加```netlify.toml```文件，如官网所示：

> [build]
> publish = "public"
> command = "hugo --gc --minify"
>
> [context.production.environment]
> HUGO_VERSION = "0.66.0"
> HUGO_ENV = "production"
> HUGO_ENABLEGITINFO = "true"
>
> [context.split1]
> command = "hugo --gc --minify --enableGitInfo"
>
> [context.split1.environment]
> HUGO_VERSION = "0.66.0"
> HUGO_ENV = "production"
>
> [context.deploy-preview]
> command = "hugo --gc --minify --buildFuture -b $DEPLOY_PRIME_URL"
>
> [context.deploy-preview.environment]
> HUGO_VERSION = "0.66.0"
>
> [context.branch-deploy]
> command = "hugo --gc --minify -b $DEPLOY_PRIME_URL"
>
> [context.branch-deploy.environment]
> HUGO_VERSION = "0.66.0"
>
> [context.next.environment]
> HUGO_ENABLEGITINFO = "true"

#### 第2步：Netlify配置

根据官网的指引，连接Github中的blog-on-netlify仓库，然后修改设置。

首先修改```Build settings```，因为需要Netlify通过hugo构建，故做如下修改

![1583510667070.png](https://i.loli.net/2020/03/07/Cms9vcMBQaoZxfO.png)

然后修改

![1583510849748.png](https://i.loli.net/2020/03/07/nEK6tA4o2NzQyGq.png)

这里的修改是因为Netlify默认使用的Hugo版本过低，需要手动设置，否则不支持Meme主题。

#### 第3步：完成

构建完成后，Netlify会自动生成一个二级域名，指向你的博客，至此Hugo博客的搭建就完成了。

## 四、自定义域名

因为之前在腾讯云上搭建过博客，也是在上面注册的.com域名，所以这里以腾讯云的DNS解析操作为例。

操作很简单，首先在Domian management中添加域名![1583511456178.png](https://i.loli.net/2020/03/07/QN67BfgvWCZuHFE.png)然后在腾讯云域名的解析记录里添加两条记录。

![image.png](https://i.loli.net/2020/03/09/xan3kQCeifIWtvy.png)

等待10分钟后，就可以通过域名访问自己的网站了。

Netlify推荐使用 SSL/TLS 的域名，白嫖的方法很多，这里就不赘述了。