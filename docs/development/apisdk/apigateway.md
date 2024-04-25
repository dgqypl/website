---
sidebar_position: 2
---

# API 访问方式

Cloudpods 的API由各个后端服务分别提供，因此为了访问API，需要API的调用方必须能够直接访问各个服务。每个服务的APi可以通过两个URL地址访问。一个是EndpointType类型为internal的URL，该URL只能在Kubernetes集群内访问；另一个是EndpointType类型为public的URL，该URL为托管平台服务的宿主机的Node Port的访问URL。

调用API的服务有几种部署方案：

## 部署在Kubernetes集群内

这种情况，API调用方可以直接访问各个服务的service cluster IP，因此可以访问各个服务EndpointType为internal的地址。

## 和宿主机在同一内网安全域内

这种情况，API调用方可以直接访问宿主机的Node Port，因此可以访问各个服务EndpointType为public的地址。

## 和平台不在一个同一个网络安全域

这种情况，服务调用方不能直接访问集群的宿主机的Node Port，需要通过一个统一的API入口访问，这样便于做网络安全策略。

### 统一API入口

为此，API网关提供统一的API入口。各个后端服务API的入口URL统一由API网关转发，需要在访问路径上插入如下的路径：

```
/api/s/<service_type>
```

例如，访问region服务的网络（networks）列表服务的正常REST API如下，且需要直接访问region服务的internal或public的Endpoint。

```
GET /networks
```

统一通过API网关的REST API如下，其中 compute_v2 为region服务的服务类型。

```
GET /api/s/compute_v2/networks
```

### 开启统一API入口

该统一API入口默认关闭，需要在apigateway服务通过配置打开：

```
climc service-config-edit yunionapi
```

将如下配置项设置为True：

```
  enable_backend_service_proxy: true
```

设置后需重启apigateway服务。

### SDK和climc使用统一API入口

为了方便使用统一API入口，SDK和climc做了如下的变更：

[代码示例](https://github.com/yunionio/cloudpods/blob/be62b3e0495d687f0d7e45bc73df6a6abef5f8ac/pkg/mcclient/README.md?plain=1#L62)

* 认证服务入口需设置为：

```
https://<ip_or_domain_of_apigatway>/api/s/identity/v3
```

* EndpointType 需设置为 apigateway


### climc的配置

climc需要配置如下环境变量以使用统一的API入口访问后端：

```
export OS_AUTH_URL=https://<ip_or_domain_of_apigatway>/api/s/identity/v3
export OS_ENDPOINT_TYPE=apigateway
```

例如，访问web控制台的地址为：[https://192.168.220.71](https://192.168.220.71)， 则统一API的AUTH_URL地址为：[https://192.168.220.71/api/s/identity/v3](https://192.168.220.71/api/s/identity/v3)。
