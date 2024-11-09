---
#  friesi23.github.io (c) by FriesI23
#
#  friesi23.github.io is licensed under a
#  Creative Commons Attribution-ShareAlike 4.0 International License.
#
#  You should have received a copy of the license along with this
#  work. If not, see <http://creativecommons.org/licenses/by-sa/4.0/>.
classes: wide
title: 正确利用 Nginx 缓存加速 Github Page 访问
excerpt: |
  最近在尝试为博客进行缓存预热时, 发现脚本请求明明已经显示 "HIT", 同时服务器也设置了正确的 "proxy_cache_key",
  但是使用浏览器或其他设备访问还是可能无法命中缓存 ("MISS").
  本文将尝试分析产生这类问题的原因并给出正确的配置方法, 最终达到大幅提高缓存命中率的目的.
author: FriesI23
date: 2024-11-09 10:00:00 +0800
category:
tags:
  - github-page
  - nginx
  - nginx-cache
  - nginx-cache-miss
  - proxy
---

前面写了 ["使用 Nginx 代理 Github Page 并实现 HTTPS 访问"][blog-github-page-proxy-1] 与
["为 Nginx 反向代理配置缓存并在 Docker 中使用 ngx_cache_purge 模块"][blog-github-page-proxy-2]
两篇文章来介绍如何使用 Nginx 充当类似 CDN 的角色来加速 Github Page 上的博客.
不过最近在尝试预热缓存时, 发现 `curl` 请求明明已经显示 `HIT`, 但是使用浏览器访问还是可能无法命中缓存,
因此又高强度阅读了一波 Nginx 相关的文档并最终找到问题所在: `Vary`.

下面会先简单回顾一下缓存的基本配置方法, 然后着重阐述配置的一些坑和需要 "标准化" 的地方.

## 1. 简单配置

下面的配置可以为对应服务开启最基本的缓存服务, 这里就不一一介绍作用了, 有需要可以看前面的文章或者直接看 Nginx 官方教程.

> `<...>` 为需要自行替换的部分.

```nginx
proxy_cache_path /var/cache/nginx/my_cache levels=1:2 keys_zone=my_cache:100m max_size=5g inactive=30d use_temp_path=off;

upstream github-pages {
    server 185.199.108.153:443;
    server 185.199.109.153:443;
    server 185.199.110.153:443;
    server 185.199.111.153:443;
    server [2606:50c0:8000::153]:443;
    server [2606:50c0:8001::153]:443;
    server [2606:50c0:8002::153]:443;
    server [2606:50c0:8003::153]:443;
}

server {
    listen 443 default_server ssl;
    listen [::]:443 ssl;

    server_name <example.com>;

    proxy_redirect off;

    proxy_set_header Host <your.real.github.page.host>;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    add_header X-Cache-Status $upstream_cache_status;

    proxy_cache my_cache;

    location / {
        proxy_pass https://github-pages;
    }
}
```

我们也可以加上一些基本的进阶配置来控制缓存行为:

```nginx
server {
    proxy_cache_lock on;
    proxy_cache_lock_age 1s;
    proxy_cache_lock_timeout 5s;
    proxy_cache_use_stale error timeout updating invalid_header http_500 http_502 http_503 http_504;
    proxy_cache_key $scheme://$proxy_host$uri$is_args$args;
    proxy_cache_revalidate on;
    proxy_cache_background_update on;

    location / {
        proxy_pass https://github-pages;

        proxy_cache_valid 200 302 1h;
        proxy_cache_valid 500 502 504 1m;
        proxy_cache_valid any 10m;
    }
}
```

## 2. 静态资源

对于一些静态资源, 也包括 Javascript 脚本, 他们其实并不会频繁变更, 尤其是 js 脚本使用 `xxx-x.y.z.js` 这种形式发布时, 其内容大概率是确定的.
此时可以针对这些资源单独配置缓存时长:

```nginx
server {
    location ~* .(jpg|jpeg|png|gif|css|js)$ {
        proxy_pass https://github-pages;

        expires 30d;
        proxy_cache_valid any 10d;
    }
}
```

## 3. 缓存控制

对于请求头, 其中的 `X-Accel-Expires`, `Set-Cookie` 和 `Vary` 都会优先于 `proxy_cache_valid`.
而这也是引起产生 `MISS` 的根本原因, 下面会分别介绍这些头的作用并给出解决方法.

### 3.1. 响应缓存时间控制相关: "X-Accel-Expires" / "Expires" / "Cache-Control"

这三个放在一起说. 其中 `X-Accel-Expires` 由客户端设置, 该设置会覆盖掉 Nginx 中 `proxy_cache_valid` 设置的值.
如果客户端没有设置 `X-Accel-Expires`, 则 Nginx 也会参考 `Expires` 与 `Cache-Control` 中设置的值.

如果我们希望客户端强制使用服务器缓存 (而不是控制 Nginx 缓存时间), 则需要在请求时忽略掉响应的头, 具体配置方法如下:

```nginx
server {
    proxy_ignore_headers X-Accel-Expires Cache-Control Expires;
}
```

配置后 Nginx 会强制忽略客户端设置缓存相关的头字段, 从而达到强制使用服务器缓存的目的.

### 3.2. Cookie 相关: "Set-Cookie"

`Set-Cookie` 会保存客户端当前会话对应的信息, 这当然是不能被缓存的. Nginx 文档中也是响应内容:

> Syntax: [proxy_cache_valid][nginx-prorxy_cache_valid] [code ...] time;
> ...
>
> - If the header includes the “Set-Cookie” field, such a response will not be cached.

对于登录用户创建的 Cookie, 不应进行缓存. 不过对于匿名用户, 则可以使用缓存.

```nginx
server {
    proxy_no_cache $cookie_sessionid;
}
```

### 3.3. Vary 相关: "Vary"

`Vary` 主要用于指示缓存服务器如何对缓存内容进行区分, 这也是使得我们缓存利用率低甚至失效的罪魁祸首.
已 `Accept-Encoding` 为例, 假设此时 Nginx 还没有缓存:

1. Client 请求中包含 `Accept-Encoding: gzip, deflate, br, zstd`, 表示客户端支持以下几种压缩方式并期待服务器返回其中一种.
2. Nginx 收到请求后, 将其转发给真正的后端.
3. 后端应答后, Nginx 收到 Response, 假设内容为: `Vary: Accept-Encoding, Content-Encoding: br`.
   此时 Nginx 将请求转发回客户端, 同时对响应内容进行缓存.

那么上面缓存的 `key` 是什么呢, 是 `proxy_cache_key` 中配置的 `key` 么? 并不是.

> Syntax: [proxy_cache_valid][nginx-prorxy_cache_valid] [code ...] time;
> ...
>
> - If the header includes the “Vary” field with the special value “\*”,
>   such a response will not be cached (1.7.7).
>   If the header includes the “Vary” field with another value,
>   such a response will be cached taking into account the corresponding request header fields (1.7.7).

事实上 Nginx 会根据 `Accept-Encoding` 的值进行缓存 (Nginx 是根据 Reqeust 而不是 Response 构建 key 的).
这会导致实际的 `key` 是 `Accept-Encoding` 的值与 `proxy_cache_valid` 的值拼接而成.

这会导致什么问题呢, 如果后续有了如下流程:

1. Client 请求中包含 `Accept-Encoding: gzip, br`, 表示客户端支持以下几种压缩方式并期待服务器返回其中一种.
2. Nginx 收到请求...

此时 Nginx 并不会使用缓存, 因为没有找到对应的 `key`, 因此还是会发生如下流程:

1. Nginx 收到请求后, 将其转发给真正的后端.
2. 后端应答后, Nginx 收到 Response, 假设内容为: `Vary: Accept-Encoding, Content-Encoding: br`.
   此时 Nginx 将请求转发回客户端, 同时对响应内容进行缓存.

发现问题了么, `Accept-Encoding` 内的内容是可以排列组合的, 这会导致生成:

```text
SUM = C(4, 1) × 1! + C(4, 2) × 2! + C(4, 3) × 3! + C(4, 4) × 4! = 64
```

种组合的 `key`, 这显然是不能接受的, 因为如果返回的是 `br`, 那缓存的实际就是一份数据, 那就应该 HIT 而不是 MISS.

那应该怎样解决呢, 我们可以 "标准化" `Vary` 支持的相关头. 下面会介绍配置方法, 以 [`Accept-Encoding`](#4-标准化) 为例.

> 不要将 Vary 加入到 `proxy_ignore_headers`, 可以看这里的相关讨论:
> [ServerFault - Is it safe to use proxy_ignore_headers Vary?][is-it-safe-to-use-proxy-ignore-headers-vary]

## 4. 标准化

### 4.1. 启用压缩

针对静态资源, 可以启用压缩, 针对静态资源 NGINX 会首先检查请求的文件是否已经有一个对应的已压缩文件, 有的话会直接返回, 而不是实时压缩.
使用以下配置可以启用压缩 (以 gzip 为例, 这个 Nginx 直接支持的压缩方式).

```nginx
server {
    gzip on;
    gzip_vary on;

    location ~* .(jpg|jpeg|png|gif|css|js)$ {
        gzip_static on;
    }
}
```

不过需要注意的是 Nginx 的缓存行为发生在压缩前. 这会导致如果启用缓存且后端服务器响应不支持对应压缩方式时,
每次请求命中缓存后内容都会重新被实时压缩. 当然 Github Page 大概肯定是支持 gzip 的, 所以可以放心打开.

### 4.2. 标准化 "Accept-Encoding" 请求头

前面我们已经知道导致缓存频繁 MISS 的罪魁祸首就是 `Accept-Encoding` 这类请求头导致 `key` 的排列组合,
因此解决思路就是让 `Accept-Encoding` 中的每一种压缩类型都对应一个 `key`,
也就是在 Nginx 请求到后端时保证 `Accept-Encoding` 中只有一种压缩类型 (选择压缩方式由 Nginx 代理).

```nginx
map $http_accept_encoding $encoding {
    ~[\s:,]?gzip(?:[\s,\;]|$) "gzip"; # gzip
    default ""; # uncompressed
}

server {
    proxy_set_header Accept-Encoding $encoding;

    proxy_cache_key $scheme://$proxy_host$encoding$uri$is_args$args;
}
```

我们由 Nginx 重写 `Accept-Encoding` 头的值, 使其请求后端时只包含一种压缩方式, 同时在 `key` 中加入这个值, 便可以让缓存与压缩方式对应.
此时请求流程如下:

1. Client 请求中包含 `Accept-Encoding: gzip, deflate, br, zstd`, 表示客户端支持以下几种压缩方式并期待服务器返回其中一种.
2. Nginx 收到请求后, 将头重写为 `Accept-Encoding: gzip`, 将其转发给真正的后端.
3. 后端应答后, Nginx 收到 Response, 假设内容为: `Vary: Accept-Encoding, Content-Encoding: gzip`.
   此时 Nginx 将请求转发回客户端, 同时对响应内容进行缓存 (`key`: [`gzip`, `<$proxy_cache_key>`]).

需要注意的是 map 是由上到下匹配的, 如果你的服务器配置了 `brotli` 且希望可以优先使用, 可以配置如下
(记得打开 Nginx 对 `brotli` 的支持: `brotli on; brotli_vary on;`):

```nginx
map $http_accept_encoding $encoding {
    ~[\s:,]?br(?:[\s,\;]|$) "br"; # brotli
    ~[\s:,]?gzip(?:[\s,\;]|$) "gzip"; # gzip
    default ""; # uncompressed
}
```

另外在修改 `proxy_cache_key` 时, 将 `$encoding` 添加到 `$uri` 或任何用户控制字段之前, 最大程度防止以下问题:

```text
# Request 1
http://example/resource1      Accept-Encoding: gzip
key: http://example/resource1gzip
# Request 2
http://example/resource1gzip
key: http://example/resource1gzip
```

## 5. 总结

经过上述配置, 基本可以解决各种缓存 MISS 的问题, 由于每一种压缩对应一个 `key`, 我们便可以写个脚本来正确定期预热缓存了.
这里给一个简单的预热方法:

```shell
curl http://example.com/sitemap.xml | grep -Eo "(http|https)://[a-zA-Z0-9./?=_%:-]*" | grep -v "pdf" | sort | while read line; do
curl -A 'Cache Warmer' -s -L $line > /dev/null 2>&1
curl -A 'Cache Warmer' -H "Accept-Encoding: br" -s -L $line > /dev/null 2>&1
curl -A 'Cache Warmer' -H "Accept-Encoding: gzip" -s -L $line > /dev/null 2>&1
echo $line
done
```

然后是完整的 Nginx 配置, 和以前一样替换/随机化比较隐私的相关配置.下面配置仅供参考,
具体请根据自己服务器情况 (CPU/内存/存储/带宽等) 与站点访问情况进行个性化配置:

```nginx
limit_req_zone $anti_spider zone=blog:60m rate=200r/m;
proxy_cache_path /var/cache/nginx/blog levels=1:2 keys_zone=blog:100m max_size=500m inactive=20d use_temp_path=off;

upstream github-pages {
    server 185.199.108.153:443;
    server 185.199.109.153:443;
    server 185.199.110.153:443;
    server 185.199.111.153:443;
    server [2606:50c0:8000::153]:443;
    server [2606:50c0:8001::153]:443;
    server [2606:50c0:8002::153]:443;
    server [2606:50c0:8003::153]:443;
}

map $http_user_agent $custom_user_agent {
    default "-";
    "~*baiduspider" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/80.0.3987.132 Safari/537.36";
}

map $http_user_agent $anti_spider {
    default "";
    "~*baiduspider" "-";
}

map $http_accept_encoding $encoding {
    ~[\s:,]?gzip(?:[\s,\;]|$) "gzip";
    default ""; # uncompressed
}

log_format cache_st '$remote_addr - $upstream_cache_status [$time_local] '
'"$request" $status $body_bytes_sent '
'"$http_referer" "$http_user_agent" '
'"$http_accept_encoding" "$sent_http_vary" "$encoding"';

server {
    listen 80;
    listen [::]:80;

    server_name example.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://example.com$request_uri;
    }
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name example.com;

    gzip on;
    gzip_vary on;

    ssl_prefer_server_ciphers on;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_certificate /path/to/fullchain.pem;
    ssl_certificate_key /path/to/privkey.pem;

    access_log /path/to/access.log;
    access_log /path/to/cache.log cache_st;
    error_log /path/to/error.log;

    proxy_redirect off;

    proxy_set_header Host "real.example.com";
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header User-Agent $custom_user_agent;
    proxy_set_header Accept-Encoding $encoding;

    add_header X-Cache-Status $upstream_cache_status;

    proxy_ignore_headers X-Accel-Expires Cache-Control Expires;
    proxy_no_cache $cookie_sessionid;

    proxy_cache blog;
    proxy_cache_lock on;
    proxy_cache_lock_age 2s;
    proxy_cache_lock_timeout 4s;
    proxy_cache_use_stale error timeout updating invalid_header http_500 http_502 http_503 http_504;
    proxy_cache_key $scheme://$proxy_host$encoding$uri$is_args$args;
    proxy_cache_revalidate on;
    proxy_cache_background_update on;

    limit_req zone=blog burst=5 nodelay;

    location ~ /purge(/.*) {
        allow 127.0.0.1;
        deny all;
        proxy_cache_purge blog $scheme://$proxy_host$encoding$1$is_args$args;
        access_log /path/to/purge.log;
    }

    location ~* \.(jpg|jpeg|png|gif|css|js)$ {
        proxy_pass https://github-pages;

        gzip_static on;

        expires 100d;
        proxy_cache_valid any 100d;
    }

    location / {
        proxy_pass https://github-pages;

        gzip_static on;

        proxy_cache_valid 200 302 7d;
        proxy_cache_valid 206 304 302 1d;
        proxy_cache_valid 404 ;
        proxy_cache_valid 500 502 504 10m;
        proxy_cache_valid any 5m;
    }
}
```

## 6. 参考资料

1. [How to configure reverse proxy caching with Nginx](https://dzx.fr/blog/how-to-configure-reverse-proxy-caching-with-nginx/)
2. [Nginx Docs - Module ngx_http_proxy_module](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)
3. [Nginx Docs - Module ngx_http_gzip_static_module](https://nginx.org/en/docs/http/ngx_http_gzip_static_module.html)
4. [NGINX cache (same URL) first returns MISS to all Chrome, Curl and Wget](https://serverfault.com/q/1003427)
5. [Is it safe to use proxy_ignore_headers Vary?][is-it-safe-to-use-proxy-ignore-headers-vary]

<!-- refs -->

[blog-github-page-proxy-1]: /post/202406/github-page-proxy
[blog-github-page-proxy-2]: /post/202410/nginx-cache
[nginx-prorxy_cache_valid]: https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache_valid
[is-it-safe-to-use-proxy-ignore-headers-vary]: https://serverfault.com/q/1005628
