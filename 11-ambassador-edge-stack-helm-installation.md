# Ambassador系列-11-Helm安装Ambassador Edge Stack 1.1.0

Ambassador已经发布1.1.0版本，其中最大的变化就是将原来的版本Ambassador和Ambassador Pro版本拆为三个版本：Ambassador API Gateway、Ambassador Edge Stack Community和Ambassador Edge Stack Enterprise。其中Ambassador API Gateway对应0.x版本的Ambassador，Ambassador Edge Stack对应o.x版本的Ambassador Pro，其中Ambassador Edge Stack Community是企业版本的试用版，Ambassador Edge Stack Enterprise是正式的企业版。

从Ambassador的文档和部署方式看，Ambassador的重点放在了Ambassador Edge Stack版本了，Ambassador的缺省部署版本改为Ambassador Edge Stack Community，主要是为了推广其企业版本。以下为各个版本的特性列表。

![Editions](images/editions.png)

下面介绍Ambassador Edge Stack Community的Helm部署。增加Helm仓库。

    helm repo add datawire https://www.getambassador.io
    
    helm search repo ambassador
    NAME                    CHART VERSION   APP VERSION     DESCRIPTION
    aliyuncs/ambassador     4.4.7           0.85.0          A Helm chart for Datawire Ambassador
    datawire/ambassador     6.1.1           1.1.0           A Helm chart for Datawire Ambassador
    stable/ambassador       5.3.0           0.86.1          A Helm chart for Datawire Ambassador

先创建ambassador命名空间。

    kubectl create namespace ambassador

安装Ambassador Edge Stack，将Admin和Proxy服务改为NodePort，使得Kubernetes集群外可以访问。忽略提示信息：manifest_sorter.go:175: info: skipping unknown hook: "crd-install"。

    helm install ambassador --namespace ambassador datawire/ambassador \
      --set adminService.type=NodePort \
      --set service.type=NodePort
    manifest_sorter.go:175: info: skipping unknown hook: "crd-install"
    manifest_sorter.go:175: info: skipping unknown hook: "crd-install"
    manifest_sorter.go:175: info: skipping unknown hook: "crd-install"
    manifest_sorter.go:175: info: skipping unknown hook: "crd-install"
    manifest_sorter.go:175: info: skipping unknown hook: "crd-install"
    manifest_sorter.go:175: info: skipping unknown hook: "crd-install"
    manifest_sorter.go:175: info: skipping unknown hook: "crd-install"
    manifest_sorter.go:175: info: skipping unknown hook: "crd-install"
    manifest_sorter.go:175: info: skipping unknown hook: "crd-install"
    manifest_sorter.go:175: info: skipping unknown hook: "crd-install"
    manifest_sorter.go:175: info: skipping unknown hook: "crd-install"
    manifest_sorter.go:175: info: skipping unknown hook: "crd-install"
    manifest_sorter.go:175: info: skipping unknown hook: "crd-install"
    manifest_sorter.go:175: info: skipping unknown hook: "crd-install"
    manifest_sorter.go:175: info: skipping unknown hook: "crd-install"
    NAME: ambassador
    LAST DEPLOYED: Sat Feb  8 06:01:35 2020
    NAMESPACE: ambassador
    STATUS: deployed
    REVISION: 1
    NOTES:
    -------------------------------------------------------------------------------
    Congratulations! You have successfully installed The Ambassador Edge Stack!
    -------------------------------------------------------------------------------
    NOTE: You are currently running The Ambassador Edge Stack in EVALUATION MODE.
    
    Request a free community license key at https://SERVICE_IP/edge_stack_admin/#dashboard
    to unlock all the features of The Ambassador Edge Stack and update the value of
    licenseKey.value in your values.yaml file.
    -------------------------------------------------------------------------------
    WARNING:
    
    With your installation of the Ambassador Edge Stack, you have created a:
    
    - AuthService named ambassador-auth
    
    - RateLimitService named ambassador-ratelimit
    
    in the ambassador namespace.
    
    Please ensure there is not another of these resources configured in your cluster.
    If there is, please either remove the old resource or run
    
    helm upgrade ambassador -n ambassador --set authService.create=false --set RateLimit.create=false
    
    For help, visit our Slack at https://d6e.co/slack or view the documentation online at https://www.getambassador.io.

查看创建的CRDs。

    kubectl get crds
    NAME                                           CREATED AT
    authservices.getambassador.io                  2020-02-07T22:01:30Z
    consulresolvers.getambassador.io               2020-02-07T22:01:30Z
    filterpolicies.getambassador.io                2020-02-07T22:01:31Z
    filters.getambassador.io                       2020-02-07T22:01:30Z
    hosts.getambassador.io                         2020-02-07T22:01:31Z
    kubernetesendpointresolvers.getambassador.io   2020-02-07T22:01:31Z
    kubernetesserviceresolvers.getambassador.io    2020-02-07T22:01:31Z
    logservices.getambassador.io                   2020-02-07T22:01:31Z
    mappings.getambassador.io                      2020-02-07T22:01:31Z
    modules.getambassador.io                       2020-02-07T22:01:31Z
    ratelimits.getambassador.io                    2020-02-07T22:01:31Z
    ratelimitservices.getambassador.io             2020-02-07T22:01:31Z
    tcpmappings.getambassador.io                   2020-02-07T22:01:32Z
    tlscontexts.getambassador.io                   2020-02-07T22:01:32Z
    tracingservices.getambassador.io               2020-02-07T22:01:32Z

查看Ambassador资源，可以看出增加了ambassador-redis，Ambassador依靠Redis来暂存的身份验证凭据和速率限制等信息，所以内置了Redis服务。也可以看出Ambassador镜像已经改为aes:1.1.0，AES为Ambassador Edge Stack的简写。

    kubectl get all -nambassador -o wide
    NAME                                    READY   STATUS    RESTARTS   AGE   IP           NODE        NOMINATED NODE   READINESS GATES
    pod/ambassador-86bbd7456d-2k8q2         1/1     Running   2          12m   10.244.1.2   k8s-node1   <none>           <none>
    pod/ambassador-86bbd7456d-4xj5n         1/1     Running   0          12m   10.244.1.4   k8s-node1   <none>           <none>
    pod/ambassador-86bbd7456d-sjds6         1/1     Running   2          12m   10.244.2.4   k8s-node2   <none>           <none>
    pod/ambassador-redis-8556cbb4c6-pj596   1/1     Running   0          12m   10.244.1.3   k8s-node1   <none>           <none>
    
    NAME                       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE   SELECTOR
    service/ambassador         NodePort    10.1.106.113   <none>        80:35643/TCP,443:17699/TCP   12m   app.kubernetes.io/instance=ambassador,app.kubernetes.io/name=ambassador
    service/ambassador-admin   NodePort    10.1.245.71    <none>        8877:30549/TCP               12m   app.kubernetes.io/instance=ambassador,app.kubernetes.io/name=ambassador
    service/ambassador-redis   ClusterIP   10.1.1.37      <none>        6379/TCP                     12m   app.kubernetes.io/instance=ambassador,app.kubernetes.io/name=ambassador-redis
    
    NAME                               READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES                       SELECTOR
    deployment.apps/ambassador         3/3     3            3           12m   ambassador   quay.io/datawire/aes:1.1.0   app.kubernetes.io/instance=ambassador,app.kubernetes.io/name=ambassador
    deployment.apps/ambassador-redis   1/1     1            1           12m   redis        redis:5.0.1                  app.kubernetes.io/instance=ambassador,app.kubernetes.io/name=ambassador-redis
    
    NAME                                          DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES                       SELECTOR
    replicaset.apps/ambassador-86bbd7456d         3         3         3       12m   ambassador   quay.io/datawire/aes:1.1.0   app.kubernetes.io/instance=ambassador,app.kubernetes.io/name=ambassador,pod-template-hash=86bbd7456d
    replicaset.apps/ambassador-redis-8556cbb4c6   1         1         1       12m   redis        redis:5.0.1                  app.kubernetes.io/instance=ambassador,app.kubernetes.io/name=ambassador-redis,pod-template-hash=8556cbb4c6

打开浏览器，访问proxy端口，http和https都会引导访问Edge Policy Console。

    http://192.168.1.55:35643/
    https://192.168.1.55:17699/

![Edge Policy Console](images/edge-policy-console.png)

点击上图的蓝色链接，提示要打开Edge Policy Console的话，需要安装edgectl，点击下载Linux edgectl，并上传到Master节点。

    https://192.168.1.55:17699/edge_stack/admin/#dashboard
    https://metriton.datawire.io/downloads/linux/edgectl

![Edge Policy Console](images/edge-policy-console-2.png)

将edgectl移到Shell PATH中，此处移到/usr/local/bin目录，并修改执行权限，并登陆Edge Policy Console。

    mv edgectl /usr/local/bin/
    chmod a+x /usr/local/bin/edgectl
    
    edgectl login --namespace=ambassador 192.168.1.55:17699
    Connecting to the Ambassador Edge Stack admin UI in this cluster...
    Unexpected error while trying to open your browser.
    Visit the following URL to access the Ambassador Edge Stack admin UI:
         https://192.168.1.55:17699/edge_stack/admin/#eyJhbGciOiJQUzUxMiIsInR5cCIXXXXXXXXX.eyJsb2dpbl90b2tlblXXXXXXXXXXXjoidjEiLCJleHAiOjE1ODExMTY2NjYsImlhdCI6MTU4MTExNDg2NiwibmJmIjXXXXXXXXXXXXXXXXX.WcvB8HZtq2MehbMxXYc3pXTvGMzN6r0crXXXXXXXXXXXXXXXXAZ3I5VmrCeNGzq1e9ROWNyqhQFkwHoHTbNF8__vOMJOqHGUrgZROTPa2QFdy9xhS9f40owhHRoppglj6oqZae1aW9IqMF-s8f-nkjwjOw34-EpjuZ4wEJYjkQUN2G6_YwDo8IKPeM1qCKo6NJwmXBY5qCa3E8cupy-BI1fIE36ENc_rKxeM6H-sU_GfLQ35CLGf-gAKXXXXXXQRvW4it9NB2_zeJ9pQje-2NnYRfKfh_GEVDpfPVPFJbJxrx0LLmImrT1x7qlIKh0RGAgXXXXXXXXXXXXXXXXXXXXXXXXXXX
    Error: browse: exec: "xdg-open": executable file not found in $PATH

打开浏览器，访问以上给出的链接，真正打开了Edge Policy Console。

    https://192.168.1.55:17699/edge_stack/admin/#eyJhbGciOiJQUzUxMiIsInR5cCIXXXXXXXXX.eyJsb2dpbl90b2tlblXXXXXXXXXXXjoidjEiLCJleHAiOjE1ODExMTY2NjYsImlhdCI6MTU4MTExNDg2NiwibmJmIjXXXXXXXXXXXXXXXXX.WcvB8HZtq2MehbMxXYc3pXTvGMzN6r0crXXXXXXXXXXXXXXXXAZ3I5VmrCeNGzq1e9ROWNyqhQFkwHoHTbNF8__vOMJOqHGUrgZROTPa2QFdy9xhS9f40owhHRoppglj6oqZae1aW9IqMF-s8f-nkjwjOw34-EpjuZ4wEJYjkQUN2G6_YwDo8IKPeM1qCKo6NJwmXBY5qCa3E8cupy-BI1fIE36ENc_rKxeM6H-sU_GfLQ35CLGf-gAKXXXXXXQRvW4it9NB2_zeJ9pQje-2NnYRfKfh_GEVDpfPVPFJbJxrx0LLmImrT1x7qlIKh0RGAgXXXXXXXXXXXXXXXXXXXXXXXXXXX

![Edge Policy Console](images/edge-policy-console-3.png)

从上图的License区域可以看出，当前是评估模式，点击“Get a free Community license.”可以获取Ambassador Edge Stack Community License。输入邮件地址，点击Sign up按钮，会提示给邮箱发送了Ambassador Edge Stack Community License。访问邮箱，获取注册码。然后在master节点注册。

    edgectl license -n ambassador eyJhbGciOiJQUzUxMiIsXXXXXXXXXXXXXX.eyJsaWNlbnNlX2tleV92ZXJzaW9uIjoidjIiLCJjdXN0b21lcl9pZCI6InR3aW5nYW9Ac2luYS5jbiIsImN1c3RvbWVyX2VtYWlsIjoidHdpbmdhb0BzaW5hLmNuIiwiZW5hYmxlZF9mZWF0dXJlcyI6WyIiLCJmaWx0ZXIiLCJyYXRlbGltaXQiLCJ0cmFmZmlXXXXXXXXXXXXXGFsIl0sImVuZm9yY2VkX2xpbWl0cyI6W3sibCI6ImRldnBvcnRhbC1zZXJ2aWNlcyIsInYiOjV9LHsibCI6InJhdGVsaW1pdC1zZXJ2aWNlIiwidiI6NX0seyJsIjoiYXV0aGZpbHRlci1zZXJ2aWNlIiwidiI6NX1dLCJtZXRhZGF0YSI6e30sImV4cCI6MTYxMjUzMDk4NCwiaXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX.TKBPqYGitumz5iSbFQq8EN9KN_BAqCJs9x03K6W3WBJxUx4fpXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXDvAUtL__7PYfTS-17Jq0ZJygOAC8hGtrOz8iCw--oFkAhpZ14mvc0-CpZEn0DgKAHel0WQY7nYGQ6aEh2GYQG80rf3KBSxZwbp-sawBANArwvCvWw1W_5tSpBy3FBG33J0IIb2rS9lAuFr0ZvVdocJr5vIKb1KQAH3Ww9sxLKfFdFOLN_5fUIsFiAOYiPuo0hpQp1BbIllxCYrKAMigXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    Warning: kubectl apply should be used on resource created by either kubectl create --save-config or kubectl apply
    secret/ambassador-edge-stack configured
    Please wait a minute or two for your license to be refreshed in your running Ambassador Edge Stack.

等一分钟左右，Edge Policy Console Dashboard页面的License提示会自动消失。

下面我们尝试创建一个Kubernetes的Service和Ambassador的Mapping。

    vi echo-service.yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        app: echo
      name: echo
    spec:
      ports:
      - name: http
        port: 8080
        protocol: TCP
        targetPort: 8080
      selector:
        app: echo
    ---
    apiVersion: apps/v1
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
      template:
        metadata:
          labels:
            app: echo
        spec:
          containers:
          - name: echo
            image: e2eteam/echoserver:2.2
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
    
    vi echo-mapping.yaml
    ---
    apiVersion: getambassador.io/v2
    kind: Mapping
    metadata:
      name: echo-mapping
    spec:
      prefix: /foo
      service: echo:8080
    
    kubectl apply -f echo-service.yaml
    kubectl apply -f echo-mapping.yaml

访问https,发现Ambassador代理了echo服务的访问。

    curl -i https://192.168.1.55:17699/foo -k
    HTTP/1.1 200 OK
    date: Fri, 07 Feb 2020 23:02:02 GMT
    content-type: text/plain
    server: envoy
    x-envoy-upstream-service-time: 5
    transfer-encoding: chunked
    
    
    Hostname: echo-75cf96d976-q8vhj
    
    Pod Information:
            node name:      k8s-node2
            pod name:       echo-75cf96d976-q8vhj
            pod namespace:  default
            pod IP: 10.244.2.5
    
    Server values:
            server_version=nginx: 1.14.2 - lua: 10015
    
    Request Information:
            client_address=10.244.2.4
            method=GET
            real path=/
            query=
            request_version=1.1
            request_scheme=http
            request_uri=http://192.168.1.55:8080/
    
    Request Headers:
            accept=*/*
            content-length=0
            host=192.168.1.55:17699
            user-agent=curl/7.29.0
            x-envoy-expected-rq-timeout-ms=3000
            x-envoy-internal=true
            x-envoy-original-path=/foo
            x-forwarded-for=10.244.0.0
            x-forwarded-proto=https
            x-request-id=49720520-2d85-4747-86e5-3206d460273d
    
    Request Body:
            -no body in request-

访问http，发现被https重定向了，应该缺省配置了https重定向，但我们的https缺省端口不是443，所以不能访问。

    curl -i http://192.168.1.55:35643/foo
    HTTP/1.1 301 Moved Permanently
    location: https://192.168.1.55:35643/foo
    date: Fri, 07 Feb 2020 23:03:33 GMT
    server: envoy
    content-length: 0

也可以访问admin端口。

    http://192.168.1.55:30549/ambassador/v0/diag/

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