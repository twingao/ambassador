# Ambassador系列-08-TLS配置-HTTPS重定向和TLS终结

### HTTPS重定向

当前大多数网站都会自动将http请求重定向为https请求。上游服务如果需要支持重定向https需要配置变动或者编码。Ambassador支持在网关直接拦截http请求并重定向为https请求功能，上游服务不用做任何变动。

    Client                    Ambassador
    |                             |
    | http://<hostname>/api       |
    | --------------------------> |
    | 301: https://<hostname>/api |
    | <-------------------------- |
    | https://<hostname>/api      |
    | --------------------------> |
    |                             |


### TLS终结

客户端通过https访问上游服务，但上游服务不用启用https监听，Ambassador会监听https请求，并终结https，Ambassador将转发给上游服务的请求转为http。

客户端 -->> https --> Ambassador --> http -- 上游服务

需要Ambassador放开https监听端口，Deployment缺省已经监听了8443端口。

    vi ambassador-rbac.yaml
    ......
            ports:
            - name: http
              containerPort: 8080
            - name: https
              containerPort: 8443
            - name: admin
              containerPort: 8877
            - name: mysql
              containerPort: 3306
    ......
    
    kubectl apply -f ambassador-rbac.yaml

Ambassador服务应该将nodePort暴露为缺省端口http:80和https:443。

    vi ambassador-service.yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        service: ambassador
      name: ambssador
    spec:
      type: NodePort
      ports:
      - name: http
        port: 80
        protocol: TCP
        targetPort: 8080
        nodePort: 80
      - name: https
        port: 443
        protocol: TCP
        targetPort: 8443
        nodePort: 443
      - name: mysql
        port: 3306
        protocol: TCP
        targetPort: 3306
        nodePort: 33306
      selector:
        service: ambassador
    
    kubectl apply -f ambassador-service.yaml
    
    kubectl get svc
    NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                AGE
    ambassador-admin   NodePort    10.106.34.114    <none>        8877:38877/TCP                         9d
    ambssador          NodePort    10.101.169.174   <none>        80:80/TCP,443:443/TCP,3306:33306/TCP   91m
    echo               ClusterIP   10.99.237.186    <none>        8080/TCP,80/TCP                        9d
    echo-v1            ClusterIP   10.109.144.3     <none>        8080/TCP,80/TCP                        14h
    echo-v2            ClusterIP   10.99.106.214    <none>        8080/TCP,80/TCP                        14h
    httpbin            ClusterIP   10.105.136.255   <none>        80/TCP                                 17h
    kubernetes         ClusterIP   10.96.0.1        <none>        443/TCP                                32d
    mysql-service      ClusterIP   10.99.98.30      <none>        3306/TCP                               50m

配置tls上下文，先创建证书secret，将https证书和私钥存储到secret中。

    #创建一个私钥
    openssl genrsa -out key.pem 2048
    Generating RSA private key, 2048 bit long modulus
    ............+++
    .........................+++
    e is 65537 (0x10001)
    
    #创建一个自签名的证书
    openssl req -x509 -key key.pem -out cert.pem -days 365 -subj '/CN=ambassador-cert'
    
    ll *.pem
    -rw-r--r-- 1 root root 1111 12月  8 13:29 cert.pem
    -rw-r--r-- 1 root root 1679 12月  8 13:28 key.pem
    
    kubectl create secret tls ambassador-cert --cert=cert.pem --key=key.pem
    
    vi tls-context.yaml
    apiVersion: getambassador.io/v1
    kind: TLSContext
    metadata:
      name: tls
    spec:
      hosts: ["*"]
      secret: ambassador-cert
      redirect_cleartext_from: 8080
    
    kubectl apply -f tls-context.yaml

重新部署echo server服务和Mapping。

    vi echo-server-service.yaml
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        app: echo
      name: echo
    spec:
      ports:
      - port: 8080
        name: high
        protocol: TCP
        targetPort: 8080
      - port: 80
        name: low
        protocol: TCP
        targetPort: 8080
      selector:
        app: echo
    ---
    apiVersion: apps/v1beta1
    kind: Deployment
    metadata:
      labels:
        app: echo
      name: echo
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: echo
      strategy: {}
      template:
        metadata:
          creationTimestamp: null
          labels:
            app: echo
        spec:
          containers:
          - image: gcr.io/kubernetes-e2e-test-images/echoserver:2.2
            name: echo
            ports:
            - containerPort: 8080
            env:
              - name: NODE_NAME
                valueFrom:
                  fieldRef:
                    fieldPath: spec.nodeName
              - name: POD_NAME
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.name
              - name: POD_NAMESPACE
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.namespace
              - name: POD_IP
                valueFrom:
                  fieldRef:
                    fieldPath: status.podIP
            resources: {}
    
    vi echo-server-mapping.yaml
    ---
    apiVersion: getambassador.io/v1
    kind: Mapping
    metadata:
      name: echo-server-mapping
    spec:
      prefix: /foo
      service: echo:8080
    
    kubectl apply -f echo-server-service.yaml
    kubectl apply -f echo-server-mapping.yaml

可以看出发出http请求时，Ambassador返回301，强制重定向到https，curl重新发起https请求，最终echo server正常返回，此过程中echo server是不需要支持https的。

    curl -i -L -k http://192.168.1.50/foo
    HTTP/1.1 301 Moved Permanently
    location: https://192.168.1.50/foo
    date: Sun, 08 Dec 2019 09:22:21 GMT
    server: envoy
    content-length: 0
    
    HTTP/1.1 200 OK
    date: Sun, 08 Dec 2019 09:22:21 GMT
    content-type: text/plain
    server: envoy
    x-envoy-upstream-service-time: 12
    lua-scripts-enabled: Processed
    transfer-encoding: chunked
    
    
    Hostname: echo-5599595fd9-2vfnt
    
    Pod Information:
            node name:      k8s-node1
            pod name:       echo-5599595fd9-2vfnt
            pod namespace:  default
            pod IP: 10.244.1.26
    
    Server values:
            server_version=nginx: 1.14.2 - lua: 10015
    
    Request Information:
            client_address=10.244.2.27
            method=GET
            real path=/
            query=
            request_version=1.1
            request_scheme=http
            request_uri=http://192.168.1.50:8080/
    
    Request Headers:
            accept=*/*
            content-length=0
            host=192.168.1.50
            user-agent=curl/7.29.0
            x-envoy-expected-rq-timeout-ms=3000
            x-envoy-internal=true
            x-envoy-original-path=/foo
            x-forwarded-for=10.244.0.0
            x-forwarded-proto=https
            x-request-id=9d3b35e8-f42e-4dbb-8a53-b2a06f23041e
    
    Request Body:
            -no body in request-

为了后续的测试，删除tls-context。

    kubectl delete -f tls-context.yaml

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

[Ambassador系列-11-Helm安装Ambassador Edge Stack 1.1.0](11-ambassador-edge-stack-helm-installation.md)