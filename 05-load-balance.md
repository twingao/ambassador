# Ambassador系列-05-负载均衡

Ambassador支持多种负载均衡算法。

- round_robin，轮询算法，依次把客户端的请求分发到不同的上游服务实例。
- least_request，最少请求算法，请求会被转发到请求数最少的上游服务实例。
- ring_hash，会话亲和算法，可以根据hash、http头，Cookie或者实际的源IP地址，始终转发同一个上游服务实例。

负载均衡配置即可全局配置，也可以基于Service配置。

重新部署echo服务，将echo的副本数设为2。

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
    
    kubectl apply -f echo-server-service.yaml
    
    kubectl get pod
    NAME                         READY   STATUS    RESTARTS   AGE
    ambassador-877b57b69-cvzbl   1/1     Running   2          8d
    ambassador-877b57b69-rtgcq   1/1     Running   2          8d
    echo-5599595fd9-2vfnt        1/1     Running   2          8d
    echo-5599595fd9-ffxpn        1/1     Running   2          8d

### round_robin，轮询算法

负载均衡使用到Kubernetes endpoint级服务发现，所以需要实现创建KubernetesEndpointResolver。可以看出是轮询访问。

    vi echo-server-mapping.yaml
    ---
    apiVersion: getambassador.io/v1
    kind: KubernetesEndpointResolver
    metadata:
      name: echo-endpoint-resolver
    ---
    apiVersion: getambassador.io/v1
    kind: Mapping
    metadata:
      name: echo-server-mapping
    spec:
      prefix: /foo
      service: echo:8080
      resolver: echo-endpoint-resolver
      load_balancer:
        policy: round_robin
    
    kubectl apply -f echo-server-mapping.yaml
    
    curl -i http://192.168.1.50:38080/foo
    HTTP/1.1 200 OK
    date: Sat, 07 Dec 2019 13:23:11 GMT
    content-type: text/plain
    server: envoy
    x-envoy-upstream-service-time: 6
    lua-scripts-enabled: Processed
    transfer-encoding: chunked
    
    
    Hostname: echo-5599595fd9-2vfnt
    
    Pod Information:
            node name:      k8s-node1
            pod name:       echo-5599595fd9-2vfnt
            pod namespace:  default
            pod IP: 10.244.1.7
    ......
    
    
    curl -i http://192.168.1.50:38080/foo
    HTTP/1.1 200 OK
    date: Sat, 07 Dec 2019 13:23:28 GMT
    content-type: text/plain
    server: envoy
    x-envoy-upstream-service-time: 5
    lua-scripts-enabled: Processed
    transfer-encoding: chunked
    
    
    Hostname: echo-5599595fd9-ffxpn
    
    Pod Information:
            node name:      k8s-node2
            pod name:       echo-5599595fd9-ffxpn
            pod namespace:  default
            pod IP: 10.244.2.6
    
    curl -i http://192.168.1.50:38080/foo -s | grep "pod IP:"
            pod IP: 10.244.1.7
    curl -i http://192.168.1.50:38080/foo -s | grep "pod IP:"
            pod IP: 10.244.2.6
    curl -i http://192.168.1.50:38080/foo -s | grep "pod IP:"
            pod IP: 10.244.1.7
    curl -i http://192.168.1.50:38080/foo -s | grep "pod IP:"
            pod IP: 10.244.1.7
    curl -i http://192.168.1.50:38080/foo -s | grep "pod IP:"
            pod IP: 10.244.2.6
    curl -i http://192.168.1.50:38080/foo -s | grep "pod IP:"
            pod IP: 10.244.2.6
    curl -i http://192.168.1.50:38080/foo -s | grep "pod IP:"
            pod IP: 10.244.1.7
    curl -i http://192.168.1.50:38080/foo -s | grep "pod IP:"
            pod IP: 10.244.1.7

### least_request，最少请求算法

    ---
    apiVersion: getambassador.io/v1
    kind: KubernetesEndpointResolver
    metadata:
      name: echo-endpoint-resolver
    ---
    apiVersion: getambassador.io/v1
    kind: Mapping
    metadata:
      name: echo-server-mapping
    spec:
      prefix: /foo
      service: echo:8080
      resolver: echo-endpoint-resolver
      load_balancer:
        policy: least_request

### ring_hash，会话亲和算法

根据源IP地址会话亲和，由于源IP地址相同，所以始终转发到同一个echo实例。

    vi echo-server-mapping.yaml
    ---
    apiVersion: getambassador.io/v1
    kind: KubernetesEndpointResolver
    metadata:
      name: echo-endpoint-resolver
    ---
    apiVersion: getambassador.io/v1
    kind: Mapping
    metadata:
      name: echo-server-mapping
    spec:
      prefix: /foo
      service: echo:8080
      resolver: echo-endpoint-resolver
      load_balancer:
        policy: ring_hash
        source_ip: true
    
    kubectl apply -f echo-server-mapping.yaml
    
    curl -i http://192.168.1.50:38080/foo -s | grep "pod IP:"
            pod IP: 10.244.2.6
    curl -i http://192.168.1.50:38080/foo -s | grep "pod IP:"
            pod IP: 10.244.2.6
    curl -i http://192.168.1.50:38080/foo -s | grep "pod IP:"
            pod IP: 10.244.2.6
    curl -i http://192.168.1.50:38080/foo -s | grep "pod IP:"
            pod IP: 10.244.2.6
    curl -i http://192.168.1.50:38080/foo -s | grep "pod IP:"
            pod IP: 10.244.2.6
    curl -i http://192.168.1.50:38080/foo -s | grep "pod IP:"
            pod IP: 10.244.2.6
    curl -i http://192.168.1.50:38080/foo -s | grep "pod IP:"
            pod IP: 10.244.2.6
    curl -i http://192.168.1.50:38080/foo -s | grep "pod IP:"
            pod IP: 10.244.2.6
    curl -i http://192.168.1.50:38080/foo -s | grep "pod IP:"
            pod IP: 10.244.2.6
    curl -i http://192.168.1.50:38080/foo -s | grep "pod IP:"
            pod IP: 10.244.2.6
    curl -i http://192.168.1.50:38080/foo -s | grep "pod IP:"
            pod IP: 10.244.2.6
    curl -i http://192.168.1.50:38080/foo -s | grep "pod IP:"
            pod IP: 10.244.2.6

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