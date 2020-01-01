# Ambassador系列-01-介绍、安装和使用

### 介绍

Ambassador是由Datawire公司开源的一个API网关项目，在Github上有2.3K star。

Ambassador当前（2019-12-11）的最新的版本是0.86.0，马上准备发布1.0.0版本，更新很频繁，而且主要在y版本更新，已经趋于成熟。

从0.50.0版本开始从apiVersion: getambassador.io/v0升级为apiVersion: getambassador.io/v1，表明API已经趋于稳定。

![Ambassador Architecture](images/ambassador-arch.png)

Ambassador是一个基于envoy proxy构建的，kubernetes原生的开源微服务网关。Ambassador具有控制平面和数据平面的。数据平面是envoy，Ambassador的控制平面负责监听k8中的service资源的变化，并将配置下发envoy，实际的流量转发通过envoy来完成。

具体流程如下：

1. 服务所有者在kubernetes manifests中定义配置(通过annotation或者CRD)。
2. 当manifest应用到集群时，kubernetes api会将更改通知Ambassador。
3. Ambassador解析更改并将配置转换为一种中间语义。envoy的配置由该IR生成。
4. 新的配置通过基于gRPC的聚合发现服务（ADS）API传递给envoy。
5. 流量通过重新配置的envoy，而不会断开任何连接。

### 特点

- 扩展和可用性

Ambassador将控制平面和envoy数据平面包装为一个容器，Ambassador没有自己的数据库，所有的数据都直接存储在Kubernetes的etc中。这样的实现使得Ambassador可像普通应用一样在Kubernetes中部署，可以直接采用Kubernetes的动态扩展的优点。

- 无状态架构

Ambassador是完全无状态的体系结构，每个Ambassador实例都独立于其它实例运行。Ambassador各个实例依赖Kubernetes来协调不同实例之间的配置。

### 安装

Ambassador有两种部署方式，yaml和helm部署，由于Helm v3版本安装ambassador有bug，无法安装，此次采用yaml安装方式。

在安装Ambassador之前先修改一下Kubernetes NodePort取值范围，改为1-39999。

    vi /etc/kubernetes/manifests/kube-apiserver.yaml
        - --service-node-port-range=1-39999
    systemctl daemon-reload
    systemctl restart kubelet

安装Ambassador。

    mkdir ambassador
    cd ambassador
    
    wget https://getambassador.io/yaml/ambassador/ambassador-rbac.yaml
    
    kubectl apply -f ambassador-rbac.yaml
    service/ambassador-admin created
    clusterrole.rbac.authorization.k8s.io/ambassador created
    serviceaccount/ambassador created
    clusterrolebinding.rbac.authorization.k8s.io/ambassador created
    customresourcedefinition.apiextensions.k8s.io/authservices.getambassador.io created
    customresourcedefinition.apiextensions.k8s.io/consulresolvers.getambassador.io created
    customresourcedefinition.apiextensions.k8s.io/kubernetesendpointresolvers.getambassador.io created
    customresourcedefinition.apiextensions.k8s.io/kubernetesserviceresolvers.getambassador.io created
    customresourcedefinition.apiextensions.k8s.io/mappings.getambassador.io created
    customresourcedefinition.apiextensions.k8s.io/modules.getambassador.io created
    customresourcedefinition.apiextensions.k8s.io/ratelimitservices.getambassador.io created
    customresourcedefinition.apiextensions.k8s.io/tcpmappings.getambassador.io created
    customresourcedefinition.apiextensions.k8s.io/tlscontexts.getambassador.io created
    customresourcedefinition.apiextensions.k8s.io/tracingservices.getambassador.io created
    deployment.apps/ambassador created

可以看到Ambassador已经安装，管理service已经安装好。

    kubectl get all
    NAME                             READY   STATUS    RESTARTS   AGE
    pod/ambassador-877b57b69-cvzbl   1/1     Running   0          4m34s
    pod/ambassador-877b57b69-rtgcq   1/1     Running   0          4m34s
    
    NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
    service/ambassador-admin   NodePort    10.106.34.114   <none>        8877:31207/TCP   4m36s
    service/kubernetes         ClusterIP   10.96.0.1       <none>        443/TCP          22d
    
    NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/ambassador   2/2     2            2           4m35s
    
    NAME                                   DESIRED   CURRENT   READY   AGE
    replicaset.apps/ambassador-877b57b69   2         2         2       4m35s

打开浏览器，访问以下地址，可以打开Ambassador的Dashboard。

    http://192.168.1.50:31207/ambassador/v0/diag

由于缺省没有创建Ambassador的Proxy Service，需要单独创建。注意采用NodePort方式。

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
      - port: 8080
        targetPort: 8080
        nodePort: 38080
      selector:
        service: ambassador
    
    kubectl apply -f ambassador-service.yaml
    
    # kubectl get all
    NAME                             READY   STATUS    RESTARTS   AGE
    pod/ambassador-877b57b69-cvzbl   1/1     Running   1          8d
    pod/ambassador-877b57b69-rtgcq   1/1     Running   1          8d
    
    NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
    service/ambassador-admin   NodePort    10.106.34.114   <none>        8877:31207/TCP    8d
    service/ambssador          NodePort    10.98.129.0     <none>        8080:38080/TCP    8d
    service/kubernetes         ClusterIP   10.96.0.1       <none>        443/TCP           31d
    
    NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/ambassador   2/2     2            2           8d
    
    NAME                                   DESIRED   CURRENT   READY   AGE
    replicaset.apps/ambassador-877b57b69   2         2         2       8d

### 使用

此时可以访问网关，发现没有路由。返回404错误。

    curl -i http://192.168.1.50:38080
    HTTP/1.1 404 Not Found
    date: Thu, 28 Nov 2019 13:49:21 GMT
    server: envoy
    content-length: 0

下面看一下Ambassador作为网关如何将请求路由分发到上游服务。先创建一个上游服务。

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
      replicas: 1
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
          - image: e2eteam/echoserver:2.2
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
    service/echo created
    deployment.apps/echo created

配置Ambassador的Mapping，其中包含了路由分发的规则。

    vi echo-server-mapping.yaml
    ---
    apiVersion: getambassador.io/v1
    kind: Mapping
    metadata:
      name: echo-server-mapping
    spec:
      prefix: /
      service: echo:8080
    
    kubectl apply -f echo-server-mapping.yaml
    mapping.getambassador.io/echo-server-mapping created

访问Ambassador服务，可以看到Ambassador网关服务将请求路由到echo服务上。

    curl -i http://192.168.1.50:38080
    HTTP/1.1 200 OK
    date: Thu, 28 Nov 2019 14:12:07 GMT
    content-type: text/plain
    server: envoy
    x-envoy-upstream-service-time: 4
    transfer-encoding: chunked
    
    Hostname: echo-5599595fd9-2vfnt
    
    Pod Information:
            node name:      k8s-node1
            pod name:       echo-5599595fd9-2vfnt
            pod namespace:  default
            pod IP: 10.244.1.3
    
    Server values:
            server_version=nginx: 1.14.2 - lua: 10015
    
    Request Information:
            client_address=10.244.1.2
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
            x-request-id=49b72d0e-4c68-4eaf-9150-dff9f7a26127
    
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