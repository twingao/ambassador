# Ambassador系列-04-服务配置Mapping

Ambassador设计旨在让Kubernetes服务的开发者可以轻松灵活地配置流量如何路由到该服务，其核心是Mapping资源，支持7层的HTTP，GRPC和Websocket，也可以通过TCPMapping支持4层的TCP连接。Ambassador必须定义一个或多个Mapping才能访问上游服务。

Mapping通过不同的配置选项实现不同的路由规则，下面进行说明。

### 增加Request Headers

Ambassador可以给上游服务的http请求中添加请求头header。

    vi echo-server-mapping.yaml
    ---
    apiVersion: getambassador.io/v1
    kind: Mapping
    metadata:
      name: echo-server-mapping
    spec:
      prefix: /foo
      add_request_headers:
        x-test-proto: "%PROTOCOL%"
        x-test-ip: "%DOWNSTREAM_REMOTE_ADDRESS_WITHOUT_PORT%"
        x-test-static: This is a test header
        x-test-static-2:
          value: This the test header #same as above  x-test-static header
      service: echo:8080
    
    kubectl apply -f echo-server-mapping.yaml
    
    curl -i http://192.168.1.50:38080/foo
    HTTP/1.1 200 OK
    date: Sat, 07 Dec 2019 07:25:09 GMT
    content-type: text/plain
    server: envoy
    x-envoy-upstream-service-time: 1
    lua-scripts-enabled: Processed
    transfer-encoding: chunked
    
    
    Hostname: echo-5599595fd9-2vfnt
    
    Pod Information:
            node name:      k8s-node1
            pod name:       echo-5599595fd9-2vfnt
            pod namespace:  default
            pod IP: 10.244.1.7
    
    Server values:
            server_version=nginx: 1.14.2 - lua: 10015
    
    Request Information:
            client_address=10.244.1.9
            method=GET
            real path=/
            query=
            request_version=1.1
            request_scheme=http
            request_uri=http://192.168.1.50:8080/
    
    Request Headers:
            accept=*/*
            content-length=0
            host=192.168.1.50:38080
            user-agent=curl/7.29.0
            x-envoy-expected-rq-timeout-ms=3000
            x-envoy-internal=true
            x-envoy-original-path=/foo
            x-forwarded-for=10.244.0.0
            x-forwarded-proto=http
            x-request-id=6075a013-5d50-48ec-9f56-4f2d0dca7bdc
            x-test-ip=10.244.0.0
            x-test-proto=HTTP/1.1
            x-test-static=This is a test header
            x-test-static-2=This the test header
    
    Request Body:
            -no body in request-

### 增加Response Headers

    vi echo-server-mapping.yaml
    ---
    apiVersion: getambassador.io/v1
    kind: Mapping
    metadata:
      name: echo-server-mapping
    spec:
      prefix: /foo
      add_response_headers:
        x-test-static: This is a test header
      service: echo:8080
    
    kubectl apply -f echo-server-mapping.yaml
    
    curl -i http://192.168.1.50:38080/foo
    HTTP/1.1 200 OK
    date: Sat, 07 Dec 2019 07:28:36 GMT
    content-type: text/plain
    server: envoy
    x-envoy-upstream-service-time: 1
    x-test-static: This is a test header
    lua-scripts-enabled: Processed
    transfer-encoding: chunked
    
    
    Hostname: echo-5599595fd9-2vfnt
    
    Pod Information:
            node name:      k8s-node1
            pod name:       echo-5599595fd9-2vfnt
            pod namespace:  default
            pod IP: 10.244.1.7
    
    Server values:
            server_version=nginx: 1.14.2 - lua: 10015
    
    Request Information:
            client_address=10.244.1.9
            method=GET
            real path=/
            query=
            request_version=1.1
            request_scheme=http
            request_uri=http://192.168.1.50:8080/
    
    Request Headers:
            accept=*/*
            content-length=0
            host=192.168.1.50:38080
            user-agent=curl/7.29.0
            x-envoy-expected-rq-timeout-ms=3000
            x-envoy-internal=true
            x-envoy-original-path=/foo
            x-forwarded-for=10.244.0.0
            x-forwarded-proto=http
            x-request-id=45500d8e-5e76-49b8-9c75-6726f6cb47d6
    
    Request Body:
            -no body in request-

### 删除Request Headers

    ---
    apiVersion: getambassador.io/v1
    kind: Mapping
    metadata:
      name: echo-server-mapping
    spec:
      prefix: /foo
      remove_request_headers:
      - authorization
      service: echo:8080

### 删除Response Headers

    ---
    apiVersion: getambassador.io/v1
    kind: Mapping
    metadata:
      name: echo-server-mapping
    spec:
      prefix: /foo
      remove_response_headers:
      - x-envoy-upstream-service-time
      service: echo:8080

### 使用host或者host_regex路由

Ambassador支持按照host或者host正则表达式进行路由分发。

    vi echo-server-mapping.yaml
    ---
    apiVersion: getambassador.io/v1
    kind: Mapping
    metadata:
      name: echo-server-mapping
    spec:
      prefix: /foo
      host: example-echo.com
      service: echo:8080
    
    kubectl apply -f echo-server-mapping.yaml
    
    curl -i -H "Host: example-echo.com" http://192.168.1.50:38080/foo
    HTTP/1.1 200 OK
    date: Sat, 07 Dec 2019 07:41:21 GMT
    content-type: text/plain
    server: envoy
    x-envoy-upstream-service-time: 5
    lua-scripts-enabled: Processed
    transfer-encoding: chunked
    
    
    Hostname: echo-5599595fd9-2vfnt
    
    Pod Information:
            node name:      k8s-node1
            pod name:       echo-5599595fd9-2vfnt
            pod namespace:  default
            pod IP: 10.244.1.7
    
    Server values:
            server_version=nginx: 1.14.2 - lua: 10015
    
    Request Information:
            client_address=10.244.2.7
            method=GET
            real path=/
            query=
            request_version=1.1
            request_scheme=http
            request_uri=http://example-echo.com:8080/
    
    Request Headers:
            accept=*/*
            content-length=0
            host=example-echo.com
            user-agent=curl/7.29.0
            x-envoy-expected-rq-timeout-ms=3000
            x-envoy-internal=true
            x-envoy-original-path=/foo
            x-forwarded-for=10.244.0.0
            x-forwarded-proto=http
            x-request-id=dd4601b5-5d6f-4b15-bb9b-36cd69aac8b1
    
    Request Body:
            -no body in request-

### host改写host_rewrite

有些上游服务区分主机host，Ambassador分发到上游服务时可以改写host请求头。

    vi echo-server-mapping.yaml
    ---
    apiVersion: getambassador.io/v1
    kind: Mapping
    metadata:
      name: echo-server-mapping
    spec:
      prefix: /foo
      host_rewrite: example-echo.com
      service: echo:8080
    
    kubectl apply -f echo-server-mapping.yaml
    
    curl -i http://192.168.1.50:38080/foo
    HTTP/1.1 200 OK
    date: Sat, 07 Dec 2019 07:54:48 GMT
    content-type: text/plain
    server: envoy
    x-envoy-upstream-service-time: 11
    lua-scripts-enabled: Processed
    transfer-encoding: chunked
    
    
    Hostname: echo-5599595fd9-ffxpn
    
    Pod Information:
            node name:      k8s-node2
            pod name:       echo-5599595fd9-ffxpn
            pod namespace:  default
            pod IP: 10.244.2.6
    
    Server values:
            server_version=nginx: 1.14.2 - lua: 10015
    
    Request Information:
            client_address=10.244.2.7
            method=GET
            real path=/
            query=
            request_version=1.1
            request_scheme=http
            request_uri=http://example-echo.com:8080/
    
    Request Headers:
            accept=*/*
            content-length=0
            host=example-echo.com
            user-agent=curl/7.29.0
            x-envoy-expected-rq-timeout-ms=3000
            x-envoy-internal=true
            x-envoy-original-path=/foo
            x-forwarded-for=10.244.0.0
            x-forwarded-proto=http
            x-request-id=aa8a6982-bf11-4650-b6f3-17d830a402fd
    
    Request Body:
            -no body in request-

### 使用路径前缀prefix或者prefix_regex路由

    ---
    apiVersion: getambassador.io/v1
    kind: Mapping
    metadata:
      name: echo-server-mapping
    spec:
      prefix: /
      service: echo:8080

### 重定向

主机重定向Host Redirect。请求http://$AMBASSADOR_URL/redirect时会返回301，重定向到[http://www.baidu.com/redirect](http://www.baidu.com/redirect)。

    vi redirect-mapping.yaml
    ---
    apiVersion: getambassador.io/v1
    kind: Mapping
    metadata:
      name: redirect-mapping
    spec:
      prefix: /redirect
      service: www.baidu.com
      host_redirect: true
    
    kubectl apply -f redirect-mapping.yaml
    
    curl -i -L http://192.168.1.50:38080/redirect
    HTTP/1.1 301 Moved Permanently
    location: http://www.baidu.com/redirect
    lua-scripts-enabled: Processed
    date: Sat, 07 Dec 2019 11:41:01 GMT
    server: envoy
    content-length: 0
    
    HTTP/1.1 302 Found
    Cache-Control: max-age=86400
    Connection: Keep-Alive
    Content-Length: 222
    Content-Type: text/html; charset=iso-8859-1
    Date: Sat, 07 Dec 2019 11:41:01 GMT
    Expires: Sun, 08 Dec 2019 11:41:01 GMT
    Location: https://www.baidu.com/search/error.html
    Server: Apache
    
    HTTP/1.1 200 OK
    Accept-Ranges: bytes
    Cache-Control: max-age=86400
    Connection: Keep-Alive
    Content-Length: 15852
    Content-Type: text/html
    Date: Sat, 07 Dec 2019 11:41:02 GMT
    Etag: "3dec-57b3a9a43af80"
    Expires: Sun, 08 Dec 2019 11:41:02 GMT
    Last-Modified: Thu, 22 Nov 2018 06:01:50 GMT
    P3p: CP=" OTI DSP COR IVA OUR IND COM "
    Server: Apache
    Set-Cookie: BAIDUID=8291E8A91272E7E222C7C17DF16BFE66:FG=1; expires=Sun, 06-Dec-20 11:41:02 GMT; max-age=31536000; path=/; domain=.baidu.com; version=1
    Vary: Accept-Encoding,User-Agent
    
    <!DOCTYPE html>
    <!--STATUS OK-->
    <html>
    <head>
        <meta http-equiv="X-UA-Compatible" content="IE=edge,chrome=1">
        <meta http-equiv="content-type" content="text/html;charset=utf-8">
        <meta content="always" name="referrer">
        <script src="https://ss1.bdstatic.com/5eN1bjq8AAUYm2zgoY3K/r/www/nocache/imgdata/seErrorRec.js"></script>
        <title>页面不存在_百度搜索</title>
    ......

主机和路径重定向Host Redirect+Path Redirect，请求http://$AMBASSADOR_URL/redirect时会返回301，重定向到[http://www.baidu.com/ip](http://www.baidu.com/ip)。

    vi redirect-mapping.yaml
    ---
    apiVersion: getambassador.io/v1
    kind: Mapping
    metadata:
      name: redirect-mapping
    spec:
      prefix: /redirect
      service: www.baidu.com
      host_redirect: true
      path_redirect: /ip
    
    kubectl apply -f redirect-mapping.yaml
    
    curl -i -L http://192.168.1.50:38080/redirect
    HTTP/1.1 301 Moved Permanently
    location: http://www.baidu.com/ip
    lua-scripts-enabled: Processed
    date: Sat, 07 Dec 2019 11:51:35 GMT
    server: envoy
    content-length: 0
    
    HTTP/1.1 302 Found
    Cache-Control: max-age=86400
    Connection: Keep-Alive
    Content-Length: 222
    Content-Type: text/html; charset=iso-8859-1
    Date: Sat, 07 Dec 2019 11:51:36 GMT
    Expires: Sun, 08 Dec 2019 11:51:36 GMT
    Location: https://www.baidu.com/search/error.html
    Server: Apache
    
    HTTP/1.1 200 OK
    Accept-Ranges: bytes
    Cache-Control: max-age=86400
    Connection: Keep-Alive
    Content-Length: 15852
    Content-Type: text/html
    Date: Sat, 07 Dec 2019 11:51:36 GMT
    Etag: "3dec-57b3a9a43af80"
    Expires: Sun, 08 Dec 2019 11:51:36 GMT
    Last-Modified: Thu, 22 Nov 2018 06:01:50 GMT
    P3p: CP=" OTI DSP COR IVA OUR IND COM "
    Server: Apache
    Set-Cookie: BAIDUID=E2C2E199B52AE9299C1446F31FC46288:FG=1; expires=Sun, 06-Dec-20 11:51:36 GMT; max-age=31536000; path=/; domain=.baidu.com; version=1
    Vary: Accept-Encoding,User-Agent
    
    <!DOCTYPE html>
    <!--STATUS OK-->
    <html>
    <head>
        <meta http-equiv="X-UA-Compatible" content="IE=edge,chrome=1">
        <meta http-equiv="content-type" content="text/html;charset=utf-8">
        <meta content="always" name="referrer">
        <script src="https://ss1.bdstatic.com/5eN1bjq8AAUYm2zgoY3K/r/www/nocache/imgdata/seErrorRec.js"></script>
        <title>页面不存在_百度搜索</title>
    ......

### 重写Rewrites

不指定rewrite，默认方式，访问http://$AMBASSADOR_URL/prefix/status/200时，重写为[http://httpbin/status/200](http://httpbin/status/200)。

    ---
    apiVersion: getambassador.io/v1
    kind: Mapping
    metadata:
      name: httpbin2-mapping
    spec:
      prefix: /prefix
      service: httpbin

指定rewrite，访问http://$AMBASSADOR_URL/prefix/status/200时，重写为[http://httpbin/status/200](http://httpbin/status/200)。

    vi httpbin2-mapping.yaml
    ---
    apiVersion: getambassador.io/v1
    kind: Mapping
    metadata:
      name: httpbin2-mapping
    spec:
      prefix: /prefix
      service: httpbin
    
    kubectl apply -f httpbin2-mapping.yaml
    
    curl -i http://192.168.1.50:38080/httpstatus/200
    HTTP/1.1 200 OK
    server: envoy
    date: Sat, 07 Dec 2019 12:06:57 GMT
    content-type: text/html; charset=utf-8
    access-control-allow-origin: *
    access-control-allow-credentials: true
    content-length: 0
    x-envoy-upstream-service-time: 6
    lua-scripts-enabled: Processed

rewrite为空字符串，访问http://$AMBASSADOR_URL/status/200时，重写为[http://httpbin/status/200](http://httpbin/status/200)。

    vi httpbin2-mapping.yaml
    ---
    apiVersion: getambassador.io/v1
    kind: Mapping
    metadata:
      name: httpbin2-mapping
    spec:
      prefix: /status
      service: httpbin
      rewrite: ""
    
    kubectl apply -f httpbin2-mapping.yaml
    
    curl -i http://192.168.1.50:38080/status/200
    HTTP/1.1 200 OK
    server: envoy
    date: Sat, 07 Dec 2019 12:10:09 GMT
    content-type: text/html; charset=utf-8
    access-control-allow-origin: *
    access-control-allow-credentials: true
    content-length: 0
    x-envoy-upstream-service-time: 11
    lua-scripts-enabled: Processed

### 自动重试Automatic Retries

    retry_on：(必填)指定重试失败请求的条件，支持的值列表为以下值之一：
        5xx
        gateway-error
        connect-failure
        retriable-4xx
        refused-stream
        retriable-status-codes
    num_retries：(缺省值1) 重试次数
    per_try_timeout：(缺省值为全局的请求超时时间)每次重试的超时时间。

## Ambassador系列文章

[Ambassador系列-01-介绍、安装和使用](01-installation-introduction.md)

[Ambassador系列-02-Module模块](02-module.md)

[Ambassador系列-03-服务配置和服务发现](03-service-configuration-discovery.md)

[Ambassador系列-04-服务配置Mapping](04-service-mapping.md)

[Ambassador系列-05-负载均衡](05-load-balance.md) 

[Ambassador系列-06-金丝雀发布、断路器、CORS和流量镜像](06-other-feature.md)

[Ambassador系列-07-TCP映射TCPMapping](07-tcpmapping.md)

[Ambassador系列-08-TLS配置-HTTPS重定向和TLS终结](08-tlscontext.md)

[Ambassador系列-09-AuthService认证服务](09-authservice.md)

[Ambassador系列-10-RateLimitService限速服务](10-ratelimitservice.md)