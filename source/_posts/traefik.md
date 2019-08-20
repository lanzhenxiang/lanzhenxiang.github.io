---
title: traefik在K8s集群下，搭建HTTPS服务
date: 2019-08-20 13:23:09
tags:
  - k8s
  - traefik
  - https
---

新版本的 k8s 集群默认采用 `traefik` 代理原生 `ingress`，两者关于 `https` 的设置有比较大的区别。  
以下为`traefik` 设置 `https` 访问，供参考。

## 1. Traefik

### 1.1 TLS

```shell
# 针对 "traefik"，"cert" 名字必须是 "tls.crt"， "key" 名字必须是 "tls.key"，"traefik-ingress-controller-xxxxx" pod 默认读取对应名字
# "-subj" 是可选项
mkdir -p ~/addon/traefik/pki
cd ~/addon/traefik/pki
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=netonline.com"

# "traefik" 应用默认部署在 "kube-system" ，在对应 "namespace" 创建 "secret" 资源
kubectl create secret generic traefik-cert --from-file=/root/addon/traefik/pki/tls.crt --from-file=/root/addon/traefik/pki/tls.key -n kube-system
```

### 1.2 ConfigMap

```shell
# 以下配置适用于全部采用 "https" 的场景，"http" 访问会被重定向为 "https"
# "traefik.toml" 需要与 "traefik-ingress-controller-xxxxx" pod 中的启动参数的文件名一致
# "insecureSkipVerify = true" ，此配置指定了 "traefik" 在访问 "https" 后端时可以忽略TLS证书验证错误，从而使得 "https" 的后端，可以像http后端一样直接通过 "traefik" 透出，如kubernetes dashboard
# "insecureSkipVerify = true" 变更配置需要重启 pod 才会生效
cat ~/addon/traefik/traefik.toml
insecureSkipVerify = true
defaultEntryPoints = ["http","https"]
[entryPoints]
  [entryPoints.http]
  address = ":80"
    [entryPoints.http.redirect]
    entryPoint = "https"
  [entryPoints.https]
  address = ":443"
    [entryPoints.https.tls]
      [[entryPoints.https.tls.certificates]]
      # 默认路径，勿修改
      certFile = "/ssl/tls.crt"
      keyFile = "/ssl/tls.key"

# 生成 "configmap" 资源
kubectl create configmap traefik-conf --from-file=/root/addon/traefik/traefik.toml -n kube-system
```

### 1.3 编辑 traefik-ds.yaml

```yaml
# 挂载 "secret" 与 "configmap" 资源
# 添加 "https" 服务端口
# 添加 "traefik-ingress-controller-xxxxx" pod 启动参数
cat ~/addon/traefik/traefik-ds.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
---
kind: DaemonSet
apiVersion: extensions/v1beta1
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb
spec:
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      # 挂载 "secret" 与 "configmap" 资源
      volumes:
      - name: ssl
        secret:
          secretName: traefik-cert
      - name: config
        configMap:
          name: traefik-conf
      containers:
      - image: traefik:v1.7.12
        name: traefik-ingress-lb
        # 设置挂载点
        volumeMounts:
        - mountPath: "/ssl"
          name: "ssl"
        - mountPath: "/config"
          name: "config"
        ports:
        - name: http
          containerPort: 80
          hostPort: 80
        # 添加应用端口
        - name: https
          containerPort: 443
          hostPort: 443
        - name: admin
          containerPort: 8080
          hostPort: 8080
        securityContext:
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE
        args:
        # 添加启动参数 "--configfile=/config/traefik.toml"，注意路径与文件名与 "configmap" 的对应
        - --configfile=/config/traefik.toml
        - --api
        - --kubernetes
        - --logLevel=INFO
---
kind: Service
apiVersion: v1
metadata:
  name: traefik-ingress-service
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
    - protocol: TCP
      port: 80
      name: web
    # 添加服务端口
    - protocol: TCP
      port: 443
      name: https
    - protocol: TCP
      port: 8080
      name: admin

# 生成 "traefik" 应用
kubectl apply -f /root/addon/traefik/traefik-ds.yaml
```

## 2. Ingress

### 2.1 Ingress without TLS

```yaml
# 针对已经设置完全重定向的 "traefik" ，"ingress" 资源可直接不带 "tls" 属性
cat ~/addon/traefik/ui.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-web-ui
  namespace: kube-system
spec:
  rules:
  - host: traefik.netonline.com
    http:
      paths:
      - path: /
        backend:
          serviceName: traefik-web-ui
          servicePort: web
```

### 2.2 Ingress with TLS

```yaml
# 如果需要代理的应用不在 "kube-system" ，需要在对应 "namespace" 创建对应的 "secret"，方便 "tls:secretName" 属性调用读取
kubectl create secret generic traefik-cert --from-file=/root/addon/traefik/pki/tls.crt --from-file=/root/addon/traefik/pki/tls.key -n default

# 附带 "tls:secretName" 属性的 "ingress" 资源示例
cat ~/addon/traefik/ui.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-web-ui
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  # 针对已经设置完全重定向的 "traefik" ，"ingress" 资源可直接不带 "tls" 属性
  # tls:
  # - secretName: traefik-cert
  rules:
  - host: traefik.netonline.com
    http:
      paths:
      - backend:
          serviceName: traefik-web-ui
          servicePort: 80
```
