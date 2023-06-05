> Nginx/后端

> 最近在开发时突然发现自己的对Nginx的负载均衡配置这块知识不太清楚，之前了解过一点，但是没有总结和梳理过，借这篇文章来归纳总结一下。

# 看脑图

![Nginx的负载均衡配置](/assert/Nginx负载均衡配置.png)

# 负载均衡的概念

[负载均衡（Load Balance）](https://baike.baidu.com/item/%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1/932451)其意思就是分摊到多个操作单元上进行执行，例如Web服务器、FTP服务器、企业关键应用服务器和其它关键任务服务器等，从而共同完成工作任务。

在Web开发中，通俗来讲就是将用户的请求按照一定的策略**分配**到多台服务器上进行处理。

跨多个应用程序实例的负载平衡是优化资源利用率、最大化吞吐量、减少延迟和确保容错配置的常用技术。

# 将http请求代理到一组服务器上

下面这段配置中展示了如何将 HTTP 请求代理到后端服务器组。 该组由三台服务器组成，其中两台运行同一应用程序的实例，而第三台是备份服务器。 因为上游服务器中没有指定负载均衡算法，Nginx使用默认算法 Round Robin（轮询）：

```nginx
http {
    upstream backend {
        server backend1.example.com;
        server backend2.example.com;
        server 192.0.0.1 backup;
    }
    
    server {
        location / {
            proxy_pass http://backend; # 要将请求传递给服务器组，该组的名称在 proxy_pass 指令中进行指定
        }
    }
}
```

# Nginx的负载均衡策略

## 1. Round Robin

**轮询**：请求在服务器之间均匀分布，同时考虑**服务器权重**。 默认情况下使用此方法。

```nginx
upstream backend {
   # no load balancing method is specified for Round Robin
   server backend1.example.com;
   server backend2.example.com;
}
```

## 2. Least Connections

**最少连接**：将请求发送到活动连接数最少的服务器，这里也会考虑到**服务器权重**。

```nginx
upstream backend {
    least_conn;
    server backend1.example.com;
    server backend2.example.com;
}
```

## 3. IP Hash

**IP 哈希**：将请求发送到的服务器由客户端 IP 地址确定。 在这种情况下，使用 IPv4 地址的前三个八位字节或整个 IPv6 地址来计算哈希值。 该方法保证来自同一地址的请求到达同一服务器，除非它不可用。

```nginx
upstream backend {
    ip_hash;
    server backend1.example.com;
    server backend2.example.com;
}
```

## 4. Generic Hash

**通用hash**：请求发送到的服务器由用户定义的关键字确定，该关键字可以是文本字符串、变量或这两者的组合。

```nginx
upstream backend {
    hash $request_uri consistent;
    server backend1.example.com;
    server backend2.example.com;
}
```

`hash` 指令的可选 `consistent` 参数启用 ketama consistent-hash 负载平衡。 请求根据用户定义的散列键值均匀分布在所有上游服务器上。 如果将上游服务器添加到上游组或从上游组中删除，则仅会重新映射几个键，从而在负载平衡缓存服务器或其他累积状态的应用程序的情况下最大限度地减少缓存未命中。

## 5. Least Time（NGINX Plus才可用）

**最少时间**：对于每个请求，Nginx选择具有最低平均延迟和最少活动连接数的服务器。这里也会考虑到**服务器权重**。

其中最低平均延迟是根据 least_time 指令包含的以下参数计算的：

- `header` ：从服务器接收第一个字节的时间
- `last_byte` ：从服务器接收完整响应的时间
- `last_byte inflight` ：从服务器接收完整响应的时间，考虑到不完整的请求

```nginx
upstream backend {
    least_time header;
    server backend1.example.com;
    server backend2.example.com;
}
```

## 6. Random（NGINX Plus完整可用）

**随机**：每个请求都将传递到随机选择的服务器。 这里也会考虑到**服务器权重**。

如果指定了 two 参数，首先，NGINX 会根据服务器权重随机选择两个服务器，然后使用指定的方法选择其中一个服务器：

* `least_conn`：最少的活动连接数
* `least_time=header` (NGINX Plus)：从服务器接收响应头的最少平均时间（`$upstream_header_time`）
* `least_time=last_byte` (NGINX Plus)：从服务器接收完整响应的最短平均时间（`$upstream_response_time`）

```nginx
upstream backend {
    random two least_time=last_byte;
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com;
    server backend4.example.com;
}
```

# 权重

默认情况下，Nginx 使用 Round Robin 方法根据权重在组中的服务器之间分配请求。 服务器指令的权重参数设置服务器的权重； 默认值为 1：

```nginx
upstream backend {
    server backend1.example.com weight=5;
    server backend2.example.com;
    server 192.0.0.1 backup;
}
```

在示例中，backend1.example.com 的权重为 5； 其他两台服务器具有默认权重 (1)，但 IP 地址为 192.0.0.1 的服务器被标记为备用服务器并且不会接收请求，除非其他两台服务器均不可用。 使用这种权重配置，在每 6 个请求中，5 个发送到 backend1.example.com，1 个发送到 backend2.example.com。

# 慢启动（NGINX Plus才可用）

服务器慢启动功能可防止最近恢复的服务器被连接淹没，这可能会请求超时并导致服务器再次被标记为失败。

慢启动允许上游服务器在恢复或可用后逐渐将其权重从 0 恢复到其设定值。 这可以通过服务器指令的 `slow_start` 参数来完成：

```nginx
upstream backend {
    server backend1.example.com slow_start=30s;
    server backend2.example.com;
    server 192.0.0.1 backup;
}
```

时间值（此处为 30 秒）设置 NGINX Plus 将与服务器的连接数增加到最大值的时间。

请注意，如果组中只有一台服务器，则服务器指令的 `max_fails`、`fail_timeout` 和 `slow_start `参数将被忽略，服务器永远不会被视为不可用。

# uptream的参数配置

## 1. weight

这个在上面已经讲到了。

## 2. max_conns（NGINX Plus才可用）

使用 NGINX Plus，可以通过使用`max_conns`参数指定最大数量来限制与上游服务器的活动连接数量。

如果已达到`max_conns`限制，则将请求放入队列中以进行进一步处理，前提是还包括 queue 指令以设置队列中可以同时存在的最大请求数：

```nginx
upstream backend {
    server backend1.example.com max_conns=3;
    server backend2.example.com;
    queue 100 timeout=70;
}
```

如果队列被请求填满，或者在可选超时参数指定的超时期间无法选择上游服务器，则客户端会收到错误。

请注意，如果在其他工作进程中打开了空闲的保活连接，则应该忽略这个`max_conns`限制配置。 因为在内存与多个工作进程共享的配置中，与服务器的连接总数可能会超过`max_conns`值。

## max_fails

设置在 `fail_timeout` 参数设置的持续时间内应发生的与服务器通信的不成功尝试次数，从而判断服务器在 fail_timeout 参数设置的持续时间内不可用。 默认情况下，不成功尝试的次数设置为 1。0值为禁用尝试计数。

用通俗的话来说：表示失败了几次，就标记server已宕机，从而将其踢出上游服务。

## 4. fail_timeout

表示失败的重试时间。

默认情况下，该参数设置为 10 秒。

## 5. backup

将服务器标记为备份服务器。 当主服务器不可用时，自己才会加入到集群中，提供服务。

该参数不能与 hash、ip_hash 和Random负载均衡方法一起使用。

## 6. down

将服务器标记为永久不可用。

# 参考文章

* [HTTP Load Balancing](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/)
* [Module ngx_http_upstream_module](https://nginx.org/en/docs/http/ngx_http_upstream_module.html)
* [Nginx篇03-负载均衡简单配置和算法原理](https://tinychen.com/20200319-nginx-03-load-balancing/#4%E3%80%81%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1%E7%AD%96%E7%95%A5)
* [深入剖析Nginx负载均衡算法](https://www.taohui.tech/2021/02/08/nginx/%E6%B7%B1%E5%85%A5%E5%89%96%E6%9E%90Nginx%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1%E7%AE%97%E6%B3%95/)