# springsnail
《Linux 高性能服务器》附带的项目程序详细解读

## 安装及使用
下载源码：
```shell
git clone https://github.com/liu-jianhao/springsnail.git
```
然后进入springsnil目录直接`make`即可生成可执行文件

填写配置文件，我测试的是网站是网易云音乐，首先我先看看网易云音乐服务器的ip有哪几个：
```shell
$ nslookup music.163.com
Server:         8.8.8.8
Address:        8.8.8.8#53

Non-authoritative answer:
Name:   music.163.com
Address: 103.65.41.125
Name:   music.163.com
Address: 103.65.41.126
```
可以看出网易云音乐的网站有两个ip地址，我配置文件中的逻辑地址就用这两个地址

`config.xml`:
```xml
Listen 127.0.0.1:8080

<logical_host>
  <name>103.65.41.126</name>
  <port>80</port>
  <conns>2</conns>
</logical_host>
<logical_host>
  <name>103.65.41.125</name>
  <port>80</port>
  <conns>2</conns>
</logical_host>
```
第一行是负载均衡服务器的地址，下面两个则是真正的服务器，`spingsnil`只是起到一个中转站的作用，将客户端的连接转发给比较“闲”的服务器

接下来开始运行程序：
```shell
$ ./springsnail -f config.xml
[ 09/13/18 21:49:56 ] mgr.cpp:0050 info: logcial srv host info: (103.65.41.126, 80)
[ 09/13/18 21:49:56 ] mgr.cpp:0050 info: logcial srv host info: (103.65.41.125, 80)
[ 09/13/18 21:49:58 ] mgr.cpp:0062 info: build connection 0 to server success
[ 09/13/18 21:49:59 ] mgr.cpp:0062 info: build connection 0 to server success
[ 09/13/18 21:49:59 ] mgr.cpp:0062 info: build connection 1 to server success
[ 09/13/18 21:50:00 ] mgr.cpp:0062 info: build connection 1 to server success
```
没有出错服务器就已经在运行了，然后我尝试连接该负载均衡服务器：
```shell
$ nc localhost 8080
GET / HTTP/1.1
Host: music.163.com

HTTP/1.1 302 Found
Server: nginx
Date: Thu, 13 Sep 2018 13:52:34 GMT
Content-Length: 0
Connection: keep-alive
Cache-Control: no-store
Pragrma: no-cache
Expires: Thu, 01 Jan 1970 00:00:00 GMT
Cache-Control: no-cache
Location: https://music.163.com/
X-Via: MusicEdgeServer
X-From-Src: 218.17.40.86
```
而另一端服务器则会有类似以下的输出：
```shell
[ 09/13/18 21:52:24 ] processpool.h:0378 info: send request to child 0
[ 09/13/18 21:52:24 ] mgr.cpp:0109 info: bind client sock 11 with server sock 9
```
我采用模拟HTTP报文发送给负载均衡服务器，服务器也确实有返回数据回来，我也用浏览器直接访问，但是会出现`403`编码，表示禁止访问，可能服务器出于安全的考虑，禁止这样的中转站的存在，所以不能让客户端访问（纯属猜测）

大家也可以试试其他的网站，每个网站的结果都有些不一样

## 各个文件的作用
1. main.cpp:
    1. 设置命令行参数的处理
    2. 读取配置文件
    3. 解析配置文件
    4. 开始线程池的循环

2. processpool: 线程池，是整个项目的发动机
3. fdwrapper：操作fd的包裹函数
4. log：日志
5. conn：客户端
6. mgr：处理网络连接和负载均衡的框架