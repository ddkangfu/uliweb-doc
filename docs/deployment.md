# 部署指南

uliweb支持任何标准的wsgi方式的部署。缺省情况下，在创建一个项目后，会在项目目录
下生成: `fast_handler.fcgi` 和 `wsgi_handler.py`.

并且uliweb目录还支持：gae, sae和dotcloud。因此，如果想要在这些环境上部署，一般
需要执行:


```
uliweb support [gae|sae|dotcloud]
```

则会分别生成相应的环境包含部署用的处理程序。

并且uliewb还提供了静态文件的提取功能，它可以把所有安装的app下的static目录汇总到
指定的目录下去，然后利用web server来提供静态文件的处理。如使用:


```
uliweb exportstatic /your/static/path
```


## Apache


### mod_wsgi


1. 按 [mod_wsgi](http://code.google.com/p/modwsgi/) 的说明安装mod_wsgi到apache下。

    * 拷贝mod_wsgi.so到apache的modules目录下

    Window环境可以看：

    > http://code.google.com/p/modwsgi/wiki/InstallationOnWindows
    Linux环境可以看：

    > http://code.google.com/p/modwsgi/wiki/InstallationOnLinux
1. 配置 apache 的httpd.conf文件

    > LoadModule wsgi_module modules/mod_wsgi.so
    > WSGIScriptAlias / /path/to/uliweb/wsgi_handler.py
    > 
    > <Directory /path/to/youruliwebproject>
    > Order deny,allow
    > Allow from all
    > </Directory>
    > 
    > 上面是将起始的URL设为/，你可以根据需要换为其它的起始URL，如/myproj。
    > 
    > 如果在windows下，示例为：
    > 
    > WSGIScriptAlias / d:/project/svn/uliweb/wsgi_handler.py
    > 
    > <Directory d:/project/svn/uliweb>
    > Order deny,allow
    > Allow from all
    > </Directory>
1. 重启apache
1. 测试。启动浏览器输入： [http://localhost/YOURURL](http://localhost/YOURURL) 来检测你的网站可否可以正常访问。


### 静态文件

当需要使用apache来配置静态文件时，可以在配置文件中配置为:


```
Alias /static/ /your/static/path/
```


## Lighttpd + SCGI


1. 配置lighttpd.conf：

    ```
    scgi.server=(
       "/uliweb.scgi"=> (
                        "main" => (
                               "socket" => "/tmp/uliweb.sock",
                               "check-local" => "disable",
                               ),
                       ),
       )
       url.rewrite-once = (
                        "^(/.*)$" => "/uliweb.scgi$1",
       )
    ```

1. 运行：

    ```
    python runcgi.py protocol=scgi socket=/tmp/uliweb.sck method=threaded daemonize=true
    ```



{% alert class=info %}
runcgi.py需要使用flup,下地址：[http://trac.saddi.com/flup](http://trac.saddi.com/flup)

{% endalert %}

## IIS + SCGI


1. 下载安装pyISAPI_SCGI 地址: [http://code.google.com/p/pyisapi-scgi/](http://code.google.com/p/pyisapi-scgi/)
1. pyISAPI_SCGI配置方法 [http://code.google.com/p/pyisapi-scgi/wiki/PyISAPI_SCGI_0_6_17](http://code.google.com/p/pyisapi-scgi/wiki/PyISAPI_SCGI_0_6_17)
1. 编辑scgi.conf:

    ```
    port=3033 #设置一个空闲的端口号
    ```

1. 运行:

    ```
    python runcgi.py protocol=scgi host=127.0.0.1 port=3033 method=threaded
    ```



{% alert class=info %}
runcgi.py需要使用flup,下地址：[http://trac.saddi.com/flup](http://trac.saddi.com/flup)

{% endalert %}

## 虚拟主机(DreamHost,BlueHost,HostMonsger等)


### FastCGI


1. 安装python, 参考http://wiki.dreamhost.com/Python
1. 新建dispatch.fcgi,内容：

    ```
    #!/home/yourname/bin/python (你安装的python的路径)
    import sys
    from runcgi import run
    run(method='threaded',protocol='fcgi')
    ```

1. 编辑.htaccess，内容：

    ```
    Options +FollowSymLinks +ExecCGI
    RewriteEngine On
    RewriteBase /
    RewriteRule ^(dispatch\.fcgi/.*)$ - [L]
    RewriteRule ^(.*)$ dispatch.fcgi/$1 [L]
    AddHandler fastcgi-script .fcgi #或者是AddHandler fcgid-script .fcgi
    ```



### CGI


1. 安装python, 参考http://wiki.dreamhost.com/Python
1. 修改runcgi.py,将第一行内容修改为：

    ```
    #!/home/yourname/bin/python (你安装的python的路径)
    ```

1. 修改.htaccess,内容：

    ```
    Options +FollowSymLinks +ExecCGI
    RewriteEngine On
    RewriteBase /
    RewriteRule ^(runcgi\.py/.*)$ - [L]
    RewriteRule ^(.*)$ runcgi.py/$! [L]
    AddHandler cgi-script .py
    ```



{% alert class=info %}
以CGI方式运行，需flup 1.0以上版本。

{% endalert %}

## Nginx+uwsgi


### Nginx

使用Nginx运行Uliweb，可以考虑使用nginx+uwsgi的模式，其中nginx采用反向代理的方式
来配置。uwsgi可以采用手工处理，也可以考虑使用supervisor来管理。

在Nginx中配置比较简单，在nginx.conf中添加:


```
http {
#...
include /etc/nginx/conf.d/*.conf;
#...
}
```

这里将其它不涉及的内容忽略掉了。上面的代码的作用是包含在conf.d目录下的所有.conf文件。
因此，你可以把特别的配置写到conf.d目录下的某个文件，如: `uliweb.conf` 。内容
可以是:


```
server {
        listen 80;

        location / {
                include uwsgi_params;
                #proxy_pass localhost:8000;
                uwsgi_pass unix:///tmp/uwsgi.sock;
        }
}
```

将uwsgi设置为反向代理有两种方式，一种是通过服务的方式，即上面注释掉的那一行，但
是当请求过多，这种方式会报错。因此一般都采用socket文件的方法。

上面就把Nginx设置好了。如果要使用Nignx提供静态文件服务，可以在上面的server中添加:


```
location ~ ^/static/ {
    root /your/path/to;
}
```

这样就将URL以 `/static/` 开头的资源文件作为静态文件来处理了，同时会在 `/your/path/to`
下来查找文件。

{% alert class=warn %}
在 `/your/path/to` 下应有 `static` 子目录。
{% endalert %}

如果要支持favicon.ico，可以在配置文件中添加：

```
location = /favicon.ico {
    alias /home/webd/www/favicon.ico;
  }
```

### uwsgi

uwsgi可以支持命令行方式启动，也可以由supervisor来管理（个人以为supervisor要简单
得多）。下面是一个命令行启动uwsgi的一个示例脚本(start.sh):


```
#!/bin/bash

sockfile=/tmp/uwsgi.sock
projectdir=/your/project/path
logfile=/opt/web/logs/uwsgi.log


if [ $1 = start ];then
 psid=`ps aux|grep "uwsgi"|grep -v "grep"|wc -l`
 if [ $psid -gt 2 ];then
   echo "uwsgi is running!"
   exit 0
 else
   uwsgi -s $sockfile --chdir $projectdir -w wsgi_handler -p 10 -M -t 120 -T -C -d $logfile
 fi
 echo "Start uwsgi service [OK]"
elif [ $1 = stop ];then
 killall -9 uwsgi
 echo "Stop uwsgi service [OK]"
elif [ $1 = restart ];then
 killall -9 uwsgi
 uwsgi -s $sockfile --chdir $projectdir -w wsgi_handler -p 10 -M -t 120 -T -C -d $logfile
 echo "Restart uwsgi service [OK]"
else
 echo "Usages: sh start.sh [start|stop|restart]"
fi
```

开始的三个变量可以根据你的实际情况进行修改。这个命令提供了启动、停止、重启三个
功能。并且相应的参数你可以根据情况进行设置。因为uwsgi有许多的参数可以使用，并
且配置参数可以有三种提供方式，如：


1. 命令行参数
1. ini文件
1. xml文件

另外还可以使用supervisor来管理uwsgi程序，如下面是一个示例:


```
[program:yourproject]
command = uwsgi
 --socket /tmp/yourproject.sock
 --harakiri 60
 --reaper
 --module wsgi_handler
 --processes 2
 --master
 --home /python/env
 --chmod-socket=666
 --limit-as 256
 --socket-timeout 5
 --max-requests 2
directory=/path/to/yourproject
stopsignal=QUIT
autostart=true
autorestart=true
stdout_logfile=/tmp/yourproject.log
redirect_stderr=true
exitcodes=0,1,2
```

这里把其它的配置都忽略掉了，只显示uliweb相关的配置，上面的许多参数可以根据要求
进行修改。

* `yourproject` 应改为实际的项目名称
* `directory` 改为项目目录
* `stdout_logfile` 的值改为实际的日志文件名

其中 `--home xxx` 的作用是设置python环境，它主要是用于使用virtualenv来创建
python环境的情况。

然后使用supervisorctl就可以进行管理了。

