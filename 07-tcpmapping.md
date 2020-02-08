# Ambassador系列-07-TCP映射TCPMapping

Ambassador除了支持7层的HTTP，GRPC和Websocket，也可以通过TCPMapping支持4层的TCP连接。

我们先部署一MySQL上游服务，MySQL不对外暴露NodePort，让Ambassador代理对MySQL的访问。

先修改Ambassador的Deployment，将3306 容器端口放开。

    vi ambassador-rbac.yaml
    kind: Deployment
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
    
    kubectl apply -f ambassador-rbac.yaml

修改Ambassador的Service，将3306 端口以NodePort 33306端口放开。

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
        port: 8080
        protocol: TCP
        targetPort: 8080
        nodePort: 38080
      - name: mysql
        port: 3306
        protocol: TCP
        targetPort: 3306
        nodePort: 33306
      selector:
        service: ambassador
    
    kubectl apply -f ambassador-service.yaml

查看一下Ambassador的Service端口。

    kubectl get svc
    NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                         AGE
    ambassador-admin   NodePort    10.106.34.114    <none>        8877:38877/TCP                  9d
    ambssador          NodePort    10.98.129.0      <none>        8080:38080/TCP,3306:33306/TCP   9d
    kubernetes         ClusterIP   10.96.0.1        <none>        443/TCP                         32d

部署MySQL服务。

    vi mysql-service.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: mysql-service
      labels: 
        name: mysql-service
    spec:
      type: ClusterIP
      ports:
      - port: 3306
        name: mysql
        protocol: TCP
        targetPort: 3306
      selector:
        name: mysql-deploy
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: mysql-deploy
    spec:
      replicas: 1
      selector:
        matchLabels:
          name: mysql-deploy
      template:
        metadata:
          labels: 
            name: mysql-deploy
        spec:
          containers:
            - name: mysql
              image: mysql:8.0.18
              imagePullPolicy: IfNotPresent
              ports:
              - containerPort: 3306
              env:
              - name: MYSQL_ROOT_PASSWORD
                value: "1111"
    
    kubectl apply -f mysql-service.yaml

定义TCPMapping，将新添加的Ambassador端口镜像到mysql-service。

    vi mysql-tcpmapping.yaml
    ---
    apiVersion: getambassador.io/v1
    kind:  TCPMapping
    metadata:
      name:  mysql-tcpmapping
    spec:
      port: 3306
      service: mysql-service:3306
    
    kubectl apply -f mysql-tcpmapping.yaml

使用MySQL客户端连接，可以连通。

    mysql -h192.168.1.50 -uroot -p -P33306
    Enter password:
    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 9
    Server version: 8.0.18 MySQL Community Server - GPL
    
    Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.
    
    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.
    
    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
    
    mysql> exit

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