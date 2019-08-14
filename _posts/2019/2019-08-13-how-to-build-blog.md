---
layout: post
title:  怎么用jekyll搭建自己酷炫的个人博客
no-post-nav: true
category: it
tags: [it]
excerpt: 资源共享
---

# 一个不爱折腾的程序猿不是一个好厨师  
jekyll是什么？可以看看官网说明...，我暂且把他理解成像tomcat一样的容器  
英文： https://www.jekyll.com/  
中文： https://www.jekyll.com.cn  

下面就直接上干货

## 1. 直接将博客托管在github上面


github上托管博客很简单，只需fork一个主题，并将fork的工程名修改为{github-username}.github.io(github-username代表你自己的gitgub名称)，并将原博主的文章删除，并放上自己的博文，样式根据自己喜爱调整就OK啦。

比如：我的博客使用的是https://github.com/ityouknow/ityouknow.github.io的主题，首先将这个项目fork一下，并将fork后的项目改名字为 weiqingeng.github.io，读者需要将weiqingeng替换成自己的github用户名。

然后打开网页weiqingeng.github.com就可以访问该主题的博客了。将修改后项目git clone下来，按照主题说明进行配置的修改，将原博主的文章删除，替换成自己的博文，git push修改后的工程到github上面,github pages就会自动构建，根据你的修改内容生成页面，访问{github-username}.github.io就可以看到修改后的内容。

如果你需要配置自己的域名，可以在阿里云或者腾讯云上申请自己的域名，比如的我的域名为weiqingeng.com。在阿里云的控制台的域名解析配置以下的内容：

有域名的可以在域名供应商后台进行解析(注意域名需要备案哦)  
weiqingeng.com CNAME weiqingeng.github.com

没有域名的可以直接访问 https://weiqingeng.github.com
![](/assets/images/web.jpg)
因为里面的好多样式都是调用的http，在https调用http有跨域问题，所以大家可以在weiqingeng.com域名里面讲对应的文件下载，放到自己的工程里面做相对地址访问


是不是很easy


## 2. 将博客搬到自己的服务器



需要的环境：


1.云服务器(阿里云，百度云，网易云，京东云，随便买一个就ok，我用的是阿里云)
2 node环境
3 jekyll
4 ruby


### 安装Node
安装Node环境，执行以下命令(这里是源码安装，安装路径 /usr/local, 安装路径大家根据自己习惯而定)：

```
cd /usr/local
mkdir blog
cd blog
wget https://nodejs.org/dist/v8.12.0/node-v8.12.0-linux-x64.tar.xz
xz -d node-v8.12.0-linux-x64.tar.xz
tar -xf node-v8.12.0-linux-x64.tar 
#设置软连接，这样就可以全局使用 node 和 npm指令
ln -s /usr/local/blog/node-v8.12.0-linux-x64/bin/node /usr/bin/node
ln -s /usr/local/blog/node-v8.12.0-linux-x64/bin/npm /usr/bin/npm
#查看node版本
node -v 
#查看 npm 版本
npm -v
```
看到下图，代表node安装成功，中间可能会出现各种问题，大家根据实际情况进行解决
![](/assets/images/2019/home/node.jpg)


### 安装ruby
Jekyll依赖于Ruby环境，需要安装Ruby，执行以下命令(源码安装)：

```
 cd /usr/local/blog
 wget https://cache.ruby-lang.org/pub/ruby/2.4/ruby-2.4.4.tar.gz
 mkdir -p /usr/local/ruby
 tar -zxvf ruby-2.4.4.tar.gz 
 cd ruby-2.4.4
 ./configure --prefix=/usr/local/ruby
 make && make install
 #查看ruby版本，判断ruby是否安装成功
 ruby -v
 ```
看到下图，代表ruby安装成功，中间可能会出现各种问题，大家根据实际情况进行解决
![](/assets/images/2019/home/ruby.jpg)

ruby环境变量设置：

#编辑.bash_profile文件
vim ~/.bash_profile

在.bash_profile 中设置以下内容：

```
PATH=/usr/local/ruby/bin:$PATH:$HOME/bin

export PATH
```
使环境变量生效：source .bash_profile


### 安装jekyll
安装Jekyll，执行以下命令

```
gem install jekyll
jekyll --version  或者 jekyll -v
gem update --system
```
![](/assets/images/2019/home/jekyll.jpg)


### 部署博客代码

```
#从github上clone代码，代码量有点大
git clone https://github.com/weiqingeng/weiqingeng.github.io
cd weiqingeng.github.io
#启动jekyll
jekyll serve
```
注意：
jekyll serve命令会编译我从github上下载的源码，可能会报以下的错误：

```
Deprecation: The 'gems' configuration option has been renamed to 'plugins'. Please update your config file accordingly.
  Dependency Error: Yikes! It looks like you don't have jekyll-paginate or one of its dependencies installed. 
```
是因为博客需要用到sitemap和paginate插件，安装下即可。

```
gem install jekyll-sitemap
gem install jekyll-paginate
```
还有其他我没有遇到的问题大家可以根据实际情况来解决

重新执行jekyll serve

判断服务是否启动成功：jekyll暂用4000端口

curl 127.0.0.1:4000

或者在浏览器直接访问 ip:4000  注意要开放4000端口


### 配置nginx访问，在nginx.conf文件中加入自如下(2选1)：

1.动态配置，将请求转发到jekyll服务上
```

 upstream blog{
                server 127.0.0.1:4000;
        }

 server {
    listen       80;
    #配置自己监听的域名，大家需改成自己的就ok了
    server_name  www.weiqingeng.com  weiqingeng.com;

        location / {
                proxy_redirect  off;
                proxy_set_header        Host $host;
                proxy_set_header        X-Real-IP $remote_addr;
                proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
                client_max_body_size    200m;
                proxy_pass      http://blog;
        }

    error_page  404              /404.html;

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
   
   }
```
2.将请求转发到nginx上
改方式需要将weiqingeng.github.io博客的内容编译成静态的文件，放到nginx下面
cd /usr/local/blog/weiqingeng.github.io
jekyll build --destination=/usr/share/nginx/html
```
server {
    listen       80;
    #配置自己监听的域名，大家需改成自己的就ok了
    server_name  www.weiqingeng.com  weiqingeng.com;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    error_page  404              /404.html;

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
   
   }
```

### 自动化部署
每当我们修改过代码push到github上面，也希望当前服务器能够监听，然后从github上pull有变动的内容，重新编译成nginx的静态html
方法如下：
1.在Github设置webhook时间(钩子,将push事件告知服务器)
到github站点
https://github.com/weiqingeng/weiqingeng.github.io，点击Setting->Webhooks->Add Webhook，在添加Webhooks的配置信息，我的配置信息如下：
```
Payload URL: http://www.weiqingeng.com/callback
Content type: application/json
Secret: a123456
```
![](/assets/images/2019/home/webhook.jpg)

2. 在服务器上接收github的事件，可以用java，php，nodejs等你熟悉的语言编写一个服务器接收请求，这里以nodejs为例：
使用开源的 github-webhook-handler
按如下方法安装,方法2成功率高

```
cd /usr/local/blog/node-v8.12.0-linux-x64/lib/node_modules
安装 github-webhook-handler 方法1
npm install -g github-webhook-handler  
  
#如果没有安装成功，可以选择法2来安装
npm install -g cnpm --registry=http://r.cnpmjs.org
npm install -g github-webhook-handler
```
![](/assets/images/2019/home/webhook-handler.png)

进入 github-webhook-handler mulu
cd github-webhook-handler

新建 deploy.js

vi deploy.js

deploy.js脚本内容如下：

```
var http = require('http')
var createHandler = require('github-webhook-handler')
var handler = createHandler({ path: '/callback', secret: '123456' }) //监听请求路径，和Github 配置的密码
 
function run_cmd(cmd, args, callback) {
  var spawn = require('child_process').spawn;
  var child = spawn(cmd, args);
  var resp = "";
 
  child.stdout.on('data', function(buffer) { resp += buffer.toString(); });
  child.stdout.on('end', function() { callback (resp) });
}
 
http.createServer(function (req, res) {
  handler(req, res, function (err) {
    res.statusCode = 404
    res.end('no such location')
  })
}).listen(3006)//监听的端口
 
handler.on('error', function (err) {
  console.error('Error:', err.message)
})
 
handler.on('push', function (event) {
  console.log('Received a push event for %s to %s',
    event.payload.repository.name,
    event.payload.ref);
  run_cmd('sh', ['./depoly.sh'], function(text){ console.log(text) });//成功后，执行的脚本。
})
```
脚本的作业就是启动一个监听端口来接收请求，接收到请求后执行部署脚本，脚本内容的关键点已经标注上注释。

部署博客的脚本如下：depoly.sh
```
#!/bin/bash

echo `date` >> ./start_blog.out
cd /usr/local/blog/weiqingeng.github.io
echo `pwd` >> ./start_blog.out
echo start pull from github >> ./start_blog.out
#增量更新
#配置git的信息
git config --global user.email "your email"
git config --global user.name "your name"
git stash
git pull
git stash pop
echo start build..  >> ./start_blog.out
#将内容重新编译到nginx
jekyll build --destination=/usr/share/nginx/html

echo `date` >> ./start_blog.out 
echo end  >> ./start_blog.out
```

启动nodejs服务： nodejs占用3001端口

```
启动指令
/usr/local/blog/node-v8.12.0-linux-x64/lib/node_modules/forever/bin/forever  start -a -l forever.log -o out.log -e err.log deploy.js

#查看deploy是否启动成功
[root@wei github-webhook-handler]# ps -ef | grep deploy
root      2196     1  0 18:58 ?        00:00:00 /usr/local/blog/node-v8.12.0-linux-x64/bin/node /usr/local/blog/node-v8.12.0-linux-x64/lib/node_modules/forever/bin/monitor deploy.js
root      2206  2196  0 18:58 ?        00:00:00 /usr/local/blog/node-v8.12.0-linux-x64/bin/node /usr/local/blog/node-v8.12.0-linux-x64/lib/node_modules/github-webhook-handler/deploy.js
root      2801  2719  0 22:48 pts/0    00:00:00 grep --color=auto deploy
```

最后在nginx中配置接收github的请求转发

location = /deploy {
     proxy_pass http://127.0.0.1:3001/callback;
}

```
#静态跳转
server {
    listen       80;
    server_name www.weiqingeng.com weiqingeng.com;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

     #接收github push事件
    location = /callback {
             proxy_pass http://127.0.0.1:3001/callback;
     }

    error_page  404              /404.html;

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
   
}
```
### 留言板功能

大家修改一下 _config.yml 中 gitalk 的配置信息。具体如何操作大家可以参考这篇文章 jekyll-theme-H2O 配置 gitalk 。注册完之后，需要在 _config.yml 配置以下信息：

```
# support provider: disqus, gitment, gitalk
comments_provider: gitalk
# !!!重要!!! 请修改下面这些信息为你自己申请的
# !!!Important!!! Please modify infos below to yours
# https://disqus.com
disqus:
    username: weiqingeng
# https://imsun.net/posts/gitment-introduction/
gitment:
    owner: weiqingeng
    #仓库名称
    repo: weiqingeng.github.io
    oauth:
        client_id: bab82b6a4d7b9623258a
        client_secret: 829be82ab1507f39a370b7e55d3b7085c771fce7
# https://github.com/gitalk/gitalk#install
gitalk:
    #对应 Application name
    owner: weiqingeng
    #github中当前项目的仓库名称
    repo: weiqingeng.github.io
    clientID: bab82b6a4d7b9623258a
    clientSecret: 829be82ab1507f39a370b7e55d3b7085c771fce7
# 在使用其它评论组件时可点击显示 Disqus
lazy_load_disqus : true
```








