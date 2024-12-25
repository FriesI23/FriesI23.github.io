---
#  friesi23.github.io (c) by FriesI23
#
#  friesi23.github.io is licensed under a
#  Creative Commons Attribution-ShareAlike 4.0 International License.
#
#  You should have received a copy of the license along with this
#  work. If not, see <http://creativecommons.org/licenses/by-sa/4.0/>.
title: 正确利用 Nginx 缓存加速 Github Page 访问
excerpt: |
  最近在尝试为博客进行缓存预热时, 发现脚本请求明明已经显示 "HIT", 同时服务器也设置了正确的 "proxy_cache_key",
  但是使用浏览器或其他设备访问还是可能无法命中缓存.
  本文将尝试分析产生这类问题的原因 (Vary: Accept-Encoding), 并给出对应的解决方案.
author: FriesI23
date: 2024-11-09 10:00:00 +0800
category:
tags:
  - github-page
  - nginx
  - nginx-cache
  - nginx-cache-miss
  - vary
  - accept-encoding
  - proxy
---

前面写了 ["使用 Nginx 代理 Github Page 并实现 HTTPS 访问"][blog-github-page-proxy-1] 与
["为 Nginx 反向代理配置缓存并在 Docker 中使用 ngx_cache_purge 模块"][blog-github-page-proxy-2]
两篇文章来介绍如何使用 Nginx 充当类似 CDN 的角色来加速 Github Page 上的博客.
不过最近在尝试预热缓存时, 发现 `curl` 请求明明已经显示 `HIT`, 但是使用浏览器访问还是可能无法命中缓存.
于是高强度上网冲浪了一波, 最终解决了这个问题. 下面大概分为几个部分来复盘问题产生的原因, 探索过程以及正确的解决方案.

## 1. 初步探索

经过在网络上搜索相关内容后, 发现已经有人提出了类似的问题:

> [NGINX cache (same URL) first returns MISS to all Chrome, Curl and Wget](https://serverfault.com/q/1003427).

该问题指出了缓存不命中的原因, 既 `Vary: Accept-Encoding` 导致 Nginx 为每一个请求中 `Accept-Encoding` 的值都创建了一个缓存副本.
举个例子:

1. 一个请求使用 `Accept-Encoding: gzip, br` 发起请求且响应中存在 `Vary: Accept-Encoding` 时
   (假设响应使用的压缩方式为 `Accpet-Encoding: br`),
   Nginx 会以 `Accept-Encoding: gzip, br` 与 `proxy_cache_key` 一起生成一个缓存键, 用于存放不同压缩变体的响应数据.
2. 此时如果另一个请求使用 `Accept-Encoding: br` 发起请求时, Nginx 会使用上一个请求的缓存么? 答案是不会, 原因是 `key` 并不一致!

在上面的例子中, 第一个请求建立的 `key` 为 [`Accept-Encoding: gzip, br`, `<proxy_cache_key>`] 组合的变体,
而第二个请求查询的 `key` 为 [`Accept-Encoding: br`, `<proxy_cache_key>`], 这显然无法命中缓存, 即使事实上是可以使用第一个的缓存值的.

Nginx 为什么要这样做呢, 个人猜测是为了保证设计上的正交性, 由于 `Accept-Encoding` 可以配置多个值并由服务器决定使用哪一种压缩形式返回,
因此根据请求的 `Accept-Encoding` 作为缓存键的一部分可以从设计上最大程度保证缓存的正确性.

不过 `Vary` 会导致缓存利用率低也是事实, 因为即使对于 `gzip, br` 与 `br, gzip`, Nginx 也会为其创建两个的缓存副本.
从宏观上看大量用户使用的浏览器大致都会请求 `gzip, deflate, br, zstd` 或是其按照顺序的子集 (比如 `gzip, br` 而不是 `br, gzip`),
不过这毕竟只是一种 "约定" 而不是一种强制 "规范", 因此也会导致下面的问题:

1. 对于低流量站点缓存效率极低, 很容易导致缓存大量 MISS, 最终会对后端产生很多不必要的请求导致消耗流量.
2. 对于大流量站点, 不确定的 `Accept-Encoding` 组合也会导致缓存效率低下, 各种相同内容的缓存在磁盘上存有多个副本.
3. 缓存快速膨胀也会导致:
   1. 要么 Nginx 频繁触发配置的缓存容量上限从而删除还在使用缓存, 导致缓存震荡.
   2. 要么必须为缓存块配置更大的容量.
4. 不确定的缓存键值也可能被恶意脚本利用并使服务器缓存震荡或者磁盘空间耗尽.

因此如果要正确设置缓存, 有必要解决这个问题.

## 2. (不正确的) 解决方案 1: 忽略 `Vary` 头

["NGINX cache (same URL) first returns MISS to all Chrome, Curl and Wget"](https://serverfault.com/q/1003427)
下提出了一种解决方法, 既忽略 `Vary` 头.

这种想法很直观, 根据 ["Nginx 文档 - proxy_cache_valid"][nginx-doc-proxy_cache_valid] 中的描述:

> Parameters of caching can also be set directly in the response header.
> This has higher priority than setting of caching time using the directive.
>
> - ...
> - If the header includes the “Vary” field with the special value “\*”, such a response will not be cached (1.7.7).
>   If the header includes the “Vary” field with another value,
>   such a response will be cached taking into account the corresponding request header fields (1.7.7).

既然 `Vary` 会影响缓存键, 那使用 `proxy_ignore_headers Vary` 忽略这个头就可以解决问题.

这种方法实际上并不安全, 因为忽略 `Vary` 实际上遮盖而不是解决了问题. 因为不同的 `Accept-Encoding: xx` 响应头对应的数据是不一样的
(数据可能是未压缩或者压缩, 压缩类型也需要从 `Accept-Encoding` 头中获得).
忽略 `Vary` 后会导致 Nginx 只使用 `<proxy_cache_key>` 缓存数据, 这显然是不对的.
e.g. 如果一个客户端不支持 `br`, 但 Nginx 却返回了 `br` 压缩的数据, 这样客户端显然无法正确解压数据.

因此不能单纯的忽略 `Vary` 头, 需要寻找其他方法.

## 3. (不太正确的) 解决方案 2: 使用 $encoding 区分缓存

如果不能忽略 `Vary` 头, 那根据压缩类型对 `key` 进行处理.

在 ["Is it safe to use proxy_ignore_headers Vary?"](https://serverfault.com/q/1005628) 与
["How to configure reverse proxy caching with Nginx - Normalize request attributes"][cache-nginx-5]
中都提到了一种解决方法, 既按照压缩类型区分缓存:

```nginx
map $http_accept_encoding $encoding {
    ~[\s:,]br(?:[\s,\;]|$)       2; # brotli
    ~[\s:,]gzip(?:[\s,\;]|$)     1; # gzip
    default                      0; # uncompressed
}

http|server|location {
    proxy_set_header Accept-Encoding $encoding;
    proxy_cache_key $scheme$proxy_host$encoding$uri$is_args$args;
}
```

那么这种配置可以解决问题么, 根据个人测试, 其实是不能的, 答案也很简单, Nginx 使用 [`Accept-Encoding: xx`, `<proxy_cache_key>`]
一起为缓存创建 `key`, 而:

> By default, the cache doesn't look at the value of Accept-Encoding.
> Because the backend returns an encoded response but proxy_cache_key doesn't include the encoding,
> a client that doesn't support Brotli may receive a previously Brotli-encoded response from the cache.
> You could insert the header Vary: Accept-Encoding in the response,
> which has the same effect as adding the value of Accept-Encoding to the cache key.
> Unfortunately, Nginx uses the original and immutable value for Accept-Encoding,
> not the normalized value set with proxy_set_header.

经过测试也确实如此, 缓存键创建使用原始不可变的 `Accept-Encoding`,
`proxy_set_header Accept-Encoding $encoding;` 并不能改变 Nginx 创建缓存键的方式.
而将 `$encoding` 加入 `proxy_cache_key` 也只是改变 [`Accept-Encoding: xx`, `<proxy_cache_key>`] 中后面一部分, 前面部分是不会变的.
最终还是会导致原始问题.

### 3.1. 怎么发现的问题

在使用 purge 请求缓存时, 频繁出现 404, 但是缓存明明没有刷新 (还是有 HIT),
因此测试了以下使用不同 `Accept-Encoding` 进行请求后使用相同 `Accept-Encoding` 和前面加入缓存时使用 `$encoding` 同时测试 purge,
发现相同 `Accept-Encoding` 可以清理, 而单独的 `$encoding` 并不行.
且 `$encoding` 相同而 `Accept-Encoding` 不同的缓存被删除时打印的路径并不相同.

这充分说明了 [`解决方案2`](#3-不太正确的-解决方案-2-使用-encoding-区分缓存) 并不能解决问题.

## 4. (不知道正不正确的) 解决方案 3: 联合使用上面两个方法

理论上在忽略 `Vary` 头的同时, 同时使用 $encoding 区分缓存, 应该可以达到效果, 不过个人看起来这种方案有一种 "面多加水, 水多加面" 的既视感,
因此并没有进行测试, 因此只是猜测可行, 具体还需要更多的测试与分析.

## 5. 解决方案 4: 使用 "代理链"

由上面几个 "不太成功" 的解决方案, 我们可以总结出如下几点:

1. `Accept-Encoding` 确实会独立于 `proxy_cache_key` 配置影响 `key` 的建立;
2. `Vary` 头最好不要简单的配置忽略;
3. `proxy_set_header Accept-Encoding` 可以在请求后端服务器时 "标准化" 压缩方式, 但是不会影响 `key` 的建立;

那么一个解决方法也呼之欲出: 我们可以使用两个 `server` 组成 "代理链", 前一个 `server` 对外暴露服务并标准化请求,
后一个 `server` 用于真正请求后端服务并缓存标准化后的请求.

```text
Client ---> Server_1(:443) --> Server_2(localhost) --> Github Page
```

### 5.1. Frontend: 对外提供服务的 Server

首先需要设置的 `server` 主要用于:

1. 对外暴露接口: `80`, `443`
2. 标准化 `Accept-Encoding`

```nginx
map $http_accept_encoding $encoding {
    ~[\s:,]?gzip(?:[\s,\;]|$) "gzip"; # gzip
    default ""; # uncompressed
}

server {
    listen 80;
    listen [::]:80;

    server_name example.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        rewrite ^(.*) https://$server_name$1 permanent;
        return 301 https://example.com;
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
    ssl_certificate /path/to/example.com/cert.pem;
    ssl_certificate_key /path/to/example.com/key.pem;

    access_log /var/log/nginx/example.com.frontend.access.log;
    error_log /var/log/nginx/example.com.frontend.error.log;

    proxy_redirect off;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header User-Agent $http_user_agent;
    proxy_set_header Accept-Encoding $encoding;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

其中要点在于 `proxy_set_header Accept-Encoding $encoding;` 与 `proxy_pass http://127.0.0.1:8080;`.

前者用于将 `Accept-Encoding` 标准化然后请求到缓存服务器 (Backend), 这样可以保证真正的 Backend 使用 `key` 组合只能为
[`Accept-Encoding: <gzip|br|...>`, `<proxy_cache_key>`]. 后者则将所有请求都转发给 Backend 服务器.

其他头都原样转发, 保证该服务器的透明性.

需要注意的是 map 是由上到下匹配的, 如果你的服务器配置了 `brotli` 且希望可以优先使用, 可以配置如下
(记得在对应的服务器中打开 Nginx 对 `brotli` 的支持: `brotli on; brotli_vary on;`):

```nginx
map $http_accept_encoding $encoding {
    ~[\s:,]?br(?:[\s,\;]|$) "br"; # brotli
    ~[\s:,]?gzip(?:[\s,\;]|$) "gzip"; # gzip
    default ""; # uncompressed
}
```

### 5.2. Backend: 提供缓存的真正转发请求的 Server

该服务器使用的其实就是剥离标准化相关与对外提供服务后的配置:

```nginx
limit_req_zone $anti_spider zone=anti_spider:120m rate=50r/m;
proxy_cache_path /var/cache/nginx/blog levels=1:2 keys_zone=blog_cache:50m max_size=500m inactive=10d use_temp_path=off;

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
    default $http_user_agent;
    "~*baiduspider" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.132 Safari/537.36";
}

map $http_user_agent $anti_spider {
    default "";
    "~*baiduspider" $http_user_agent;
}

map $http_accept_encoding $encoding {
    ~[\s:,]?gzip(?:[\s,\;]|$) "gzip"; # gzip
    default ""; # uncompressed
}

server {
    listen 127.0.0.1:8080;

    server_name example.com;

    gzip on;
    gzip_vary on;

    access_log /var/log/nginx/example.com.backend.access.log;
    error_log /var/log/nginx/example.com.backend.error.log;

    proxy_redirect off;

    proxy_set_header Host your.github.io;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header User-Agent $custom_user_agent;

    add_header X-Cache-Status $upstream_cache_status;

    proxy_ignore_headers X-Accel-Expires Cache-Control Expires;
    proxy_no_cache $cookie_sessionid;

    proxy_cache blog_cache;
    proxy_cache_lock on;
    proxy_cache_lock_age 2s;
    proxy_cache_lock_timeout 4s;
    proxy_cache_use_stale error timeout updating invalid_header http_500 http_502 http_503 http_504;
    proxy_cache_key $scheme$host$uri$is_args$args;
    proxy_cache_revalidate on;
    proxy_cache_background_update on;

    limit_req zone=anti_spider burst=2 nodelay;

    location ~* \.(jpg|jpeg|png|gif|css|js)$ {
        proxy_pass https://github-pages;

        gzip_static on;

        expires 120d;
        proxy_cache_valid any 120d;
    }

    location / {
        proxy_pass https://github-pages;

        gzip_static on;

        proxy_cache_valid 200 302 1h;
        proxy_cache_valid 206 304 302 30m;
        proxy_cache_valid 404 2h;
        proxy_cache_valid 500 502 504 10m;
        proxy_cache_valid any 5m;
    }

    location ~ /purge(?<purge_url>/.*) {
        allow 127.0.0.1;
        deny all;

        proxy_cache_purge blog_cache $scheme$host$purge_url$is_args$args;
        access_log /var/log/nginx/backend.purge.log;
    }
}
```

其中下面配置用于开启 Nginx 压缩支持, 注意压缩总是发生在缓存后, 因此如果服务器流量较大,
可以考虑在分离一个服务器用于单独压缩请求, 具体可以看 [Proxy chain][cache-nginx-54] 相关配置:

```nginx
http|server {
    gzip on;
    gzip_vary on;
}
```

`proxy_set_header Host your.github.io;` 用于指向需要代理的域名 (这里是 Github Page), 加链的话要保证最终 `Host` 设置正确.

下面的配置使用了第三方模块 [`ngx_cache_purge`](https://github.com/FRiCKLE/ngx_cache_purge),
如果不会添加的话看我的[上一篇博客][blog-github-page-proxy-2]:

```nginx
http|server|location ~ /purge(?<purge_url>/.*) {
    allow 127.0.0.1;
    deny all;

    proxy_cache_purge blog_cache $scheme$host$purge_url$is_args$args;
    access_log /var/log/nginx/backend.purge.log;
}
```

## 6. 其他缓存控制

对于请求头, 其中的 `X-Accel-Expires`, `Set-Cookie` 和 `Vary` 都会优先于 `proxy_cache_valid`,
而这也是引起产生 `MISS` 的根本原因. 前面已经介绍了针对 `Vary` 的解决方法, 下面会分别介绍另外两个的作用并给出解决方法.

### 6.1. 响应缓存时间控制相关: "X-Accel-Expires" / "Expires" / "Cache-Control"

这三个放在一起说. 其中 `X-Accel-Expires` 由客户端设置, 该设置会覆盖掉 Nginx 中 `proxy_cache_valid` 设置的值.
如果客户端没有设置 `X-Accel-Expires`, 则 Nginx 也会参考 `Expires` 与 `Cache-Control` 中设置的值.

如果我们希望客户端强制使用服务器缓存 (而不是控制 Nginx 缓存时间), 则需要在请求时忽略掉响应的头, 具体配置方法如下:

```nginx
http|server {
    proxy_ignore_headers X-Accel-Expires Cache-Control Expires;
}
```

配置后 Nginx 会强制忽略客户端设置缓存相关的头字段, 从而达到强制使用服务器缓存的目的.

### 6.2. Cookie 相关: "Set-Cookie"

`Set-Cookie` 会保存客户端当前会话对应的信息, 这当然是不能被缓存的. Nginx 文档中也是响应内容:

> Syntax: [proxy_cache_valid][nginx-prorxy_cache_valid] [code ...] time;
> ...
>
> - If the header includes the “Set-Cookie” field, such a response will not be cached.

对于登录用户创建的 Cookie, 不应进行缓存. 不过对于匿名用户, 则可以使用缓存.

```nginx
http|server {
    proxy_no_cache $cookie_sessionid;
}
```

## 7. 总结

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
# nginx.conf

load_module modules/ngx_http_cache_purge_module.so;

http {
    # ...
    include /etc/nginx/conf.d/*.conf;
}

```

```nginx
# conf.d/20-exmaple.com.conf

limit_req_zone $anti_spider zone=anti_spider:120m rate=50r/m;
proxy_cache_path /var/cache/nginx/blog levels=1:2 keys_zone=blog_cache:50m max_size=500m inactive=10d use_temp_path=off;

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
    default $http_user_agent;
    "~*baiduspider" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.132 Safari/537.36";
}

map $http_user_agent $anti_spider {
    default "";
    "~*baiduspider" $http_user_agent;
}

map $http_accept_encoding $encoding {
    ~[\s:,]?gzip(?:[\s,\;]|$) "gzip"; # gzip
    default ""; # uncompressed
}

server {
    listen 80;
    listen [::]:80;

    server_name example.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        rewrite ^(.*) https://$server_name$1 permanent;
        return 301 https://example.com;
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
    ssl_certificate /path/to/example.com/cert.pem;
    ssl_certificate_key /path/to/example.com/key.pem;

    access_log /var/log/nginx/example.com.frontend.access.log;
    error_log /var/log/nginx/example.com.frontend.error.log;

    proxy_redirect off;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header User-Agent $http_user_agent;
    proxy_set_header Accept-Encoding $encoding;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}

server {
    listen 127.0.0.1:8080;

    server_name example.com;

    gzip on;
    gzip_vary on;

    access_log /var/log/nginx/example.com.backend.access.log;
    error_log /var/log/nginx/example.com.backend.error.log;

    proxy_redirect off;

    proxy_set_header Host your.github.io;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header User-Agent $custom_user_agent;

    add_header X-Cache-Status $upstream_cache_status;

    proxy_ignore_headers X-Accel-Expires Cache-Control Expires;
    proxy_no_cache $cookie_sessionid;

    proxy_cache blog_cache;
    proxy_cache_lock on;
    proxy_cache_lock_age 2s;
    proxy_cache_lock_timeout 4s;
    proxy_cache_use_stale error timeout updating invalid_header http_500 http_502 http_503 http_504;
    proxy_cache_key $scheme$host$uri$is_args$args;
    proxy_cache_revalidate on;
    proxy_cache_background_update on;

    limit_req zone=anti_spider burst=2 nodelay;

    location ~* \.(jpg|jpeg|png|gif|css|js)$ {
        proxy_pass https://github-pages;

        gzip_static on;

        expires 120d;
        proxy_cache_valid any 120d;
    }

    location / {
        proxy_pass https://github-pages;

        gzip_static on;

        proxy_cache_valid 200 302 1h;
        proxy_cache_valid 206 304 302 30m;
        proxy_cache_valid 404 2h;
        proxy_cache_valid 500 502 504 10m;
        proxy_cache_valid any 5m;
    }

    location ~ /purge(?<purge_url>/.*) {
        allow 127.0.0.1;
        deny all;

        proxy_cache_purge blog_cache $scheme$host$purge_url$is_args$args;
        access_log /var/log/nginx/backend.purge.log;
    }
}
```

## 8. 参考资料

1. [How to configure reverse proxy caching with Nginx](https://dzx.fr/blog/how-to-configure-reverse-proxy-caching-with-nginx/)
2. [Nginx Docs - Module ngx_http_proxy_module](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)
3. [Nginx Docs - Module ngx_http_gzip_static_module](https://nginx.org/en/docs/http/ngx_http_gzip_static_module.html)
4. [NGINX cache (same URL) first returns MISS to all Chrome, Curl and Wget](https://serverfault.com/q/1003427)
5. [Is it safe to use proxy_ignore_headers Vary?](https://serverfault.com/q/1005628)

<!-- refs -->

[blog-github-page-proxy-1]: /post/202406/github-page-proxy
[blog-github-page-proxy-2]: /post/202410/nginx-cache
[nginx-doc-proxy_cache_valid]: https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache_valid
[cache-nginx-5]: https://dzx.fr/blog/how-to-configure-reverse-proxy-caching-with-nginx/#normalize-request-attributes
[cache-nginx-54]: https://dzx.fr/blog/how-to-configure-reverse-proxy-caching-with-nginx/#proxy-chain
[nginx-prorxy_cache_valid]: https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache_valid
