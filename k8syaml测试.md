---
ebook:
  title: 项目交付
  authors: jicheng.guo
  base-font-size: 6
---
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->
<!-- code_chunk_output -->

* [将应用在k8s部署测试](#将应用在k8s部署测试)
* [deployment yaml 文件如下:](#deployment-yaml-文件如下)
* [策略详解：](#策略详解)

<!-- /code_chunk_output -->
### 将应用在k8s部署测试

部署要求service （cluster ip ) ---- pod 之间的策略使用轮询方式。
部署要求 pod 最小为3个，支持高可用。

###  deployment yaml 文件如下:

---
apiVersion: v1
kind: Service
metadata:
  name: glrs-weixin
spec:
  selector:
    app: glrs-weixin
  type: NodePort
  ports:
  - name: http
    protocol: TCP
    nodePort: 32001
    port: 7001
    targetPort: 7001
  - name: https
    protocol: TCP
    nodePort: 32002
    port: 7002
    targetPort: 7002
  sessionAffinity: ClientIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: glrs-weixin
  labels:
    app: glrs-weixin
spec:
  replicas: 3
  selector:
    matchLabels:
      app: glrs-weixin
  template:
    metadata:
      labels:
        app: glrs-weixin
    spec:
      containers:
      - name: glrs-weixin
        image: 10.3.16.213/wechat/wxglweblogicaddapp:1
        env:
        - name: SVN_URL
          value: svn://10.4.12.48/davesvn/code/config/uat
        - name: SVN_USERNAME
          value: yanglu
        - name: SVN_PASSWD
          value: yanglu
        - name: Server_Role
          value: Admin
        - name: ADMIN_USERNAME
          value: weblogic
        - name: AdminPort
          value: "7001"
        - name: sslport
          value: "7002"
        - name: base_domain_default_password
          value: "999999999"
        - name: APP_NAME_1
          value: Gllic_wx
        - name: APP_PATH_1
          value: /applications/Gllic_wx
        ports:
        - containerPort: 7001
        - containerPort: 7002

### 策略详解：
K8S SERVICE 到pod 的负载支持如下几种方式：
