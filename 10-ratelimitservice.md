# Ambassador系列-10-RateLimitService限速服务

限速服务RateLimitService和AuthService一样也是一个外部服务，处理流程也和AuthService一样。

限速服务RateLimitService是通过在Mapping中添加label进行声明的。在请求时label会通过gRPC接口传递到限速服务，声明的方式封为v1和v0。具体的含义没有看的很懂，v1方式声明始终没有测试通过，v0方式的声明测试通过了，下面以v0为例进行说明。

Ambassador提供了一个限速服务的样例，该限速服务接收到"x-ambassador-test-allow: true"请求头时，不限速，返回HTTP状态码200。其它的请求头值"x-ambassador-test-allow: xxxxx"都限速，返回HTTP状态码429。

	#将example-rate-limit-service服务定义为RateLimitService
	vi example-ratelimitservice.yaml
	---
	apiVersion: getambassador.io/v1
	kind: RateLimitService
	metadata:
	  name: example-ratelimitservice
	spec:
	  service: "example-rate-limit:5000"
	---
	apiVersion: v1
	kind: Service
	metadata:
	  name: example-rate-limit-service
	spec:
	  type: ClusterIP
	  selector:
	    app: example-rate-limit
	  ports:
	  - port: 5000
	    name: http-example-rate-limit
	    targetPort: http-api
	---
	apiVersion: apps/v1
	kind: Deployment
	metadata:
	  name: example-rate-limit
	spec:
	  replicas: 1
	  strategy:
	    type: RollingUpdate
	  selector:
	    matchLabels:
	      app: example-rate-limit
	  template:
	    metadata:
	      labels:
	        app: example-rate-limit
	    spec:
	      containers:
	      - name: example-rate-limit
	        image: agervais/ambassador-ratelimit-service:1.0.0
	        imagePullPolicy: Always
	        ports:
	        - name: http-api
	          containerPort: 5000
	        resources:
	          limits:
	            cpu: "0.1"
	            memory: 100Mi

	kubectl apply -f example-ratelimitservice.yaml

创建httpbin服务，其中声明mapping，并添加限速label。

	vi httpbin-rate-limit-service.yaml
	---
	apiVersion: v1
	kind: Service
	metadata:
	  name: httpbin
	  labels:
	    app: httpbin
	  annotations:
	    getambassador.io/config: |
	      ---
	      apiVersion: ambassador/v0
	      kind: Mapping
	      name: httpbin-mapping
	      prefix: /http/
	      service: httpbin:80
	      rate_limits:
	        - descriptor: A test case
	          headers:
	            - "x-ambassador-test-allow"
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
	
	kubectl apply -f httpbin-rate-limit-service.yaml

测试一下效果。

	#携带请求头"x-ambassador-test-allow: true"，不限速。
	curl -i -H "x-ambassador-test-allow: true" http://192.168.1.55:38080/http/status/200
	HTTP/1.1 200 OK
	server: envoy
	date: Wed, 01 Jan 2020 01:27:07 GMT
	content-type: text/html; charset=utf-8
	access-control-allow-origin: *
	access-control-allow-credentials: true
	content-length: 0
	x-envoy-upstream-service-time: 11

	#携带请求头"x-ambassador-test-allow: 1111"，限速。
	curl -i -H "x-ambassador-test-allow: 1111" http://192.168.1.55:38080/http/status/200
	HTTP/1.1 429 Too Many Requests
	x-envoy-ratelimited: true
	date: Wed, 01 Jan 2020 00:54:38 GMT
	server: envoy
	content-length: 0

Ambassador Pro提供了一个高级限速服务，但是是收费的，如果业务需要限速服务，需要自己实现。

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