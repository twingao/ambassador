# Ambassador系列-02-Module模块

Ambassador的全局配置，可以通过Module模块配置Ambassador的一些全局的配置参数。当前只有两种Module模块：

- ambassador：配置全局的系统参数。

- tls：配置tls参数

Module可以以CRDs方式定义。

    ---
    apiVersion: getambassador.io/v1
    kind: Module
    metadata:
      name: ambassador
    spec:
      config:
        enable_grpc_web: true
    ---
    apiVersion: getambassador.io/v1
    kind: Module
    metadata:
      name: tls
    spec:
      config:
        server:
          enabled: true
          secret: ambassador-certs
          redirect_cleartext_from: 8080

也可以以注解的方式定义，可以定义在任何Service中，但通常定义在Ambassador Service中，定义在Service中，也是全局生效。

    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: ambassador
      annotations:
        getambassador.io/config: |
          ---
          apiVersion: getambassador.io/v1
          kind: Module
          name: ambassador
          config:
            enable_grpc_web: True
          ---
          apiVersion: getambassador.io/v1
          kind: Module
          name: tls
          config:
            server:
              enabled: true
              secret: ambassador-certs
              redirect_cleartext_from: 8080
    spec:
      type: LoadBalancer
      externalTrafficPolicy: Local
      ports:
       - name: http
         port: 80
         targetPort: 8080
       - name: https
         port: 443
         targetPort: 8443
      selector:
        service: ambassador

实验环境接上节[Ambassador系列-01-介绍、安装和使用](01-introduction.md)。

下面举一例子，Ambassador支持在每个请求上运行内联Lua脚本的功能。 例如添加自定义的报文头。

    vi ambassador-module.yaml
    ---
    apiVersion: getambassador.io/v1
    kind: Module
    metadata:
      name: ambassador
    spec:
      config:
        lua_scripts: |
          function envoy_on_response(response_handle)
            response_handle:headers():add("Lua-Scripts-Enabled", "Processed")
          end
    
    kubectl apply -f ambassador-module.yaml
    module.getambassador.io/ambassador created
    
    kubectl get module
    NAME         AGE
    ambassador   53s

可以看到增加了一个响应头"lua-scripts-enabled: Processed"。

    curl -i http://192.168.1.50:38080
    HTTP/1.1 200 OK
    date: Sat, 07 Dec 2019 02:52:59 GMT
    content-type: text/plain
    server: envoy
    x-envoy-upstream-service-time: 7
    lua-scripts-enabled: Processed
    transfer-encoding: chunked
    
    
    Hostname: echo-5599595fd9-ffxpn
    
    Pod Information:
            node name:      k8s-node2
            pod name:       echo-5599595fd9-ffxpn
            pod namespace:  default
            pod IP: 10.244.2.4
    
    Server values:
            server_version=nginx: 1.14.2 - lua: 10015
    
    Request Information:
            client_address=10.244.1.5
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
            x-envoy-original-path=/
            x-forwarded-for=10.244.0.0
            x-forwarded-proto=http
            x-request-id=c294858d-bdfe-4c66-b5e0-ad48db281dfe
    
    Request Body:
            -no body in request-

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