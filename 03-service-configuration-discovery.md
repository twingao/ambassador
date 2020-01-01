# Ambassador系列-03-服务配置和服务发现

### Ambassador 服务配置

Ambassador提供了三种服务配置方法。

- CRDs方式：Customer Resource Definitions
- 注解方式：Kubernetes Service Annotations
- Ingress方式：Kubernetes Ingress

### CRDs方式

[Ambassador系列-01-介绍、安装和使用](01-introduction.md)一节中使用的就是CRDs方式，路由规则都定义在Mapping CRD中。

    ---
    apiVersion: getambassador.io/v1
    kind: Mapping
    metadata:
      name: echo-server-mapping
    spec:
      prefix: /
      service: echo:8080

### 注解方式

在Service中增加Mapping注解，该Mapping只对该Service生效。prefix为路径前缀，service为服务名+端口，后面有详细解释。

    vi httpbin-service.yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: httpbin
      annotations:
        getambassador.io/config: |
          ---
          apiVersion: ambassador/v1
          kind: Mapping
          name: httpbin-mapping
          prefix: /http
          service: httpbin:80
      labels:
        app: httpbin
    spec:
      ports:
      - name: http
        port: 80
        targetPort: 80
      selector:
        app: httpbin
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: httpbin
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: httpbin
      template:
        metadata:
          labels:
            app: httpbin
        spec:
          containers:
          - image: docker.io/kennethreitz/httpbin
            name: httpbin
            ports:
            - containerPort: 80
    
    kubectl apply -f httpbin-service.yaml
    service/httpbin created
    deployment.apps/httpbin created
    
    kubectl get pod
    NAME                         READY   STATUS    RESTARTS   AGE
    ambassador-877b57b69-cvzbl   1/1     Running   1          8d
    ambassador-877b57b69-rtgcq   1/1     Running   1          8d
    echo-5599595fd9-2vfnt        1/1     Running   1          8d
    echo-5599595fd9-ffxpn        1/1     Running   1          8d
    httpbin-8c4b74ffb-f5j6b      1/1     Running   0          7m54s

验证一下。

    curl -i http://192.168.1.50:38080/http/status/200
    HTTP/1.1 200 OK
    server: envoy
    date: Sat, 07 Dec 2019 03:35:56 GMT
    content-type: text/html; charset=utf-8
    access-control-allow-origin: *
    access-control-allow-credentials: true
    content-length: 0
    x-envoy-upstream-service-time: 6
    lua-scripts-enabled: Processed

### Ingress方式

Ambassador支持Ingress Controller，在Ingress中的配置下发给Ambassador，最终应用到Envoy中。

    vi echo-server-ingress.yaml
    ---
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      annotations:
        kubernetes.io/ingress.class: ambassador
      name: echo-server-ingress
    spec:
      rules:
      - http:
          paths:
          - path: /bar/
            backend:
              serviceName: echo
              servicePort: 8080
    
    kubectl apply -f echo-server-ingress.yaml
    
    curl -i http://192.168.1.50:38080/bar
    HTTP/1.1 200 OK
    date: Sat, 07 Dec 2019 03:57:59 GMT
    content-type: text/plain
    server: envoy
    x-envoy-upstream-service-time: 2
    lua-scripts-enabled: Processed
    transfer-encoding: chunked
    
    
    Hostname: echo-5599595fd9-2vfnt
    
    Pod Information:
            node name:      k8s-node1
            pod name:       echo-5599595fd9-2vfnt
            pod namespace:  default
            pod IP: 10.244.1.4
    
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
            x-envoy-original-path=/bar
            x-forwarded-for=10.244.0.0
            x-forwarded-proto=http
            x-request-id=b7463d93-bf82-483c-8717-ad55c4d2088d
    
    Request Body:
            -no body in request-

### 服务发现

Ambassador支持三种服务发现机制。

- Kubernetes服务级别发现Kubernetes service-level discovery

默认情况下，Ambassador使用Kubernetes DNS和服务级别发现。根据Mapping的配置，Ambassador查询Service的DNS地址。流量将被路由到Service。然后，Kubernetes将负载均衡多个Pod之间的流量。

- Kubernetes端点级发现Kubernetes endpoint-level discovery

在负载均衡中使用，如回话亲和ring_hash负载均衡算法，Ambassador绕过服务级别发现，直接通过端点发现服务。

- Consul端点级发现

Ambassador可以和Consul集成，以进行端点级服务发现。Ambassador从Consul中获取端点信息。

### 服务定义

Ambassador服务定义为[scheme://]service[.namespace][:port]

- scheme：http/https；默认值为http。
- service：服务名称，Kubernetes、Consul的服务名称，或者外部服务地址。
- namespace：如果未指定，则默认为Ambassador的名称空间。
- port：服务对应的port，如果为未指定，则当http时默认为80，当https时默认为443。

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