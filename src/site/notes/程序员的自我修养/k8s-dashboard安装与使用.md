---
{"dg-publish":true,"permalink":"//k8s-dashboard/","title":"k8s-dashboard安装与使用","tags":["Kubernetes"]}
---


# k8s-dashboard安装与使用

For English users： you can refer to my answers at [stackoverflow](https://stackoverflow.com/a/52176544/7230212)

本文适用于kubernetes 1.9-1.11版本，其它版本未验证。

## 安装

1. https模式  
      

    [官方部署文档](https://github.com/kubernetes/dashboard/wiki/Installation#recommended-setup)，注意**必须提供证书**！

    
    ```Bash
    kubectl apply -f https://github.com/kubernetes/dashboard/blob/master/src/deploy/recommended/kubernetes-dashboard.yaml
    ```
    
2. http模式
    
    ```Plain
    kubectl apply -f https://github.com/kubernetes/dashboard/blob/master/src/deploy/alternative/kubernetes-dashboard.yaml
    ```
    

以上两种模式，配置文件最后可以添加type: NodePort 实现外网的访问

## 外网访问

1. apiserver  
    apiserver是在pod中启动的，访问端口是6443，但是加了认证机制  
    
2. kube-proxy  
    kube proxy 命令会启动一个代理，供访问apiserver  
    
    ```Plain
    kubectl proxy --address='0.0.0.0' --accept-hosts='^*
    

    默认启动8001端口（端口可以指定），这时可以从外网访问了  

      

    `http(s)://MasterIP:8001/api/v1/namespaces/kube-system/services/http(s):kubernetes-dashboard:/proxy/` (根据上一步安装模式决定是否通过https)

    
3. node port  
    如果上一步添加了NodePort，直接使用  
    `nodeIp:port`访问

## 登陆

1. https模式  
    访问方式参见  
    [https://github.com/kubernetes/dashboard/wiki/Accessing-Dashboard---1.7.X-and-above](https://github.com/kubernetes/dashboard/wiki/Accessing-Dashboard---1.7.X-and-above) 页面全部是通过https访问的方式，如果没有指定证书，登陆按钮是无效的（即使提供了token）  
    参照第3点生成token登陆  
    
2. http模式  
    参见  
    [https://github.com/kubernetes/dashboard/wiki/Accessing-Dashboard---1.6.X-and-below](https://github.com/kubernetes/dashboard/wiki/Accessing-Dashboard---1.6.X-and-below)  
    http部署的，即使版本高于1.6，还是要使用1.6及以下的方式访问  
    参照第3点生成token，由于Kube-proxy和apiserver的访问方式添加的头会被丢弃，所以http方式只能通过  
    `nodeIp:port`的方式访问, 且该模式不会有登陆页面，只能通过对每一个请求添加头：`Authorization: Bearer <token>`进行访问
3. 生成token  
    参考  
    [https://github.com/kubernetes/dashboard/wiki/Creating-sample-user](https://github.com/kubernetes/dashboard/wiki/Creating-sample-user)

    ```
    

    默认启动8001端口（端口可以指定），这时可以从外网访问了  

      

    `http(s)://MasterIP:8001/api/v1/namespaces/kube-system/services/http(s):kubernetes-dashboard:/proxy/` (根据上一步安装模式决定是否通过https)

    
3. node port  
    如果上一步添加了NodePort，直接使用  
    `nodeIp:port`访问

## 登陆

1. https模式  
    访问方式参见  
    [https://github.com/kubernetes/dashboard/wiki/Accessing-Dashboard---1.7.X-and-above](https://github.com/kubernetes/dashboard/wiki/Accessing-Dashboard---1.7.X-and-above) 页面全部是通过https访问的方式，如果没有指定证书，登陆按钮是无效的（即使提供了token）  
    参照第3点生成token登陆  
    
2. http模式  
    参见  
    [https://github.com/kubernetes/dashboard/wiki/Accessing-Dashboard---1.6.X-and-below](https://github.com/kubernetes/dashboard/wiki/Accessing-Dashboard---1.6.X-and-below)  
    http部署的，即使版本高于1.6，还是要使用1.6及以下的方式访问  
    参照第3点生成token，由于Kube-proxy和apiserver的访问方式添加的头会被丢弃，所以http方式只能通过  
    `nodeIp:port`的方式访问, 且该模式不会有登陆页面，只能通过对每一个请求添加头：`Authorization: Bearer <token>`进行访问
3. 生成token  
    参考  
    [https://github.com/kubernetes/dashboard/wiki/Creating-sample-user](https://github.com/kubernetes/dashboard/wiki/Creating-sample-user)
