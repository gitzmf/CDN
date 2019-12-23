## 六、多个后端主机实现调度功能

#### 1、动静分离示例：

```
backend default {
    .host = "172.16.0.9";
    .port = "80";
}
backend appsrv {
    .host = "172.16.0.10";
    .port = "80";
}
sub vcl_recv {
    if (req.url ~ "(?i)\.php$") {
        set req.backend_hint = appsrv;
    } else {
        set req.backend_hint = default;
    }
}
```

#### 2、轮询调度

```
import directors;
backend srv1 {
    .host = "192.168.0.9";
    .port = "80";
}
backend srv2 {
    .host = "192.168.0.10";
    .port = "80";
}
sub vcl_init {
    new websrvs = directors.round_robin();  #round_robin()调度算法，不支持加权
    websrvs.add_backend(srv1);
    websrvs.add_backend(srv2);
}
sub vcl_recv {
    set req.backend_hint = websrvs.backend();
}
```

#### 3、基于cookie的session sticky（session粘滞性）

```
sub vcl_init {
    new h = directors.hash();
    h.add_backend(one, 1);
    h.add_backend(two, 1);
}
sub vcl_recv {
    set req.backend_hint = h.backend(req.http.cookie);
}
```

#### 4、随机调度，支持权重

```
sub vcl_init {
    new websrvs = directors.random();
    websrvs.add_backend(srv1, 1);
    websrvs.add_backend(srv2, 2);
}
```

#### 5、后端健康检查

> .probe：定义健康状态检测方法；
> .url：检测时要请求的URL，默认为”/";
> .request：发出的具体请求；
> .request =
> "GET /.healthtest.html HTTP/1.1"
> "Host: www.dongfei.tech"
> "Connection: close"
> .window：基于最近的多少次检查来判断其健康状态；
> .threshold：最近.window中定义的这么次检查中至有.threshhold定义的次数是成功的；
> .interval：检测频度；
> .timeout：超时时长；
> .expected_response：期望的响应码，默认为200；

```
import directors;
probe http_chk {
        .url = "/index.html";
        .interval = 2s;
        .timeout = 2s;
        .window = 10;  #最近10次检查
        .threshold = 7;  #有7次成功则为健康主机
}
backend srv1 {
        .host = "192.168.0.9";
        .port = "80";
        .probe = http_chk;
}
backend srv2 {
        .host = "192.168.0.10";
        .port = "80";
        .probe = http_chk;
}
sub vcl_init {
        new websrvs = directors.random();
        websrvs.add_backend(srv1, 1);
        websrvs.add_backend(srv2, 2);
}
sub vcl_recv {
        set req.backend_hint = websrvs.backend();
}
```

```
# 查看后端主机健康状态信息  
varnish> backend.list      
Backend name                   Refs   Admin      Probe
srv1(192.168.0.9,,80)          3      probe      Healthy 10/10
srv2(192.168.0.10,,80)         3      probe      Healthy 10/10
varnish> backend.set_health srv1 sick|healthy|auto  #手动标记主机状态 down|up|probe
```

设置后端的主机属性：

```
backend BE_NAME {
    ...
    .connect_timeout = 0.5s;  #连接超时时间
    .first_byte_timeout = 20s;  #第一个字节20s不响应则为超时
    .between_bytes_timeout = 5s;  #第一个字节和第二个字节间隔超时时间
    .max_connections = 50;  #最大连接数
}
```

## 七、varnish的运行时参数

> 最大并发连接数 = thread_pools * thread_pool_max

- thread_pools：工作线程数，最好小于或等于CPU核心数量
- thread_pool_max：每线程池的最大线程数
- thread_pool_min：最大空闲线程数
- thread_pool_timeout：空闲超过多长时间被清除
- thread_pool_add_delay：生成线程之前等待的时间
- thread_pool_destroy_delay：清除超出最大空闲线程数的线程之前等待的时间

## 八、Varnish日志管理

virnish的日志默认存储在80M的内存空间中，如果日志记录超出了则覆盖前边的日志，服务器重启后丢失；需要更改配置使其永久保存到磁盘

```
# varnishstat -1 -f MAIN  #指定查看MAIN段的信息
# varnishstat -1 -f MAIN.cache_hit -f MAIN.cache_miss  #显示指定参数的当前统计数据
MAIN.cache_hit              47         0.00 Cache hits
MAIN.cache_miss             89         0.01 Cache misses
# varnishtop -1 -i ReqHeader  #显示指定的排序信息
   165.00 ReqHeader Accept: */*
   165.00 ReqHeader Host: 192.168.0.8
   165.00 ReqHeader User-Agent: curl/7.29.0
   165.00 ReqHeader X-Forwarded-For: 192.168.0.7
```

将日志永久保存到：`/var/log/varnish/varnish.log`

```
# systemctl start varnishlog.service
varnishlog
```

以Apache/NCSA日志格式显示

```
# varnishncsa
192.168.0.7 - - [14/Jul/2018:12:34:23 +0800] "GET http://192.168.0.8/javascripts/test1.html HTTP/1.1" 200 11 "-" "curl/7.29.0"
```

补充：NCSA格式是WEB日志格式的一种