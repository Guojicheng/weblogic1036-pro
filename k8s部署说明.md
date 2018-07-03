
---
ebook:
  title: 项目交付
  authors: jicheng.guo
  base-font-size: 6
---

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->
<!-- code_chunk_output -->

* [k8s yaml  文件如下:](#k8s-yaml-文件如下)

<!-- /code_chunk_output -->
###   k8s yaml  文件如下:
---
apiVersion: v1
kind: Service
metadata:
  name: glrs-weixin
spec:
  ports:
  - name: http
    nodePort: 31001
    port: 7001
    protocol: TCP
    targetPort: 7001
  - name: https
    nodePort: 31002
    port: 7002
    protocol: TCP
    targetPort: 7002
  selector:
    app: glrs-weixin
  sessionAffinity: None   ##sessionAffinity: ClientIP
  type: NodePort
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: glrs-weixin
  name: glrs-weixin
  namespace: default
  resourceVersion: "1759544"
  selfLink: /apis/extensions/v1beta1/namespaces/default/deployments/glrs-weixin
  uid: 4ae478e0-63fe-11e8-84e4-0cda411d818d
spec:
  progressDeadlineSeconds: 600
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: glrs-weixin
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: glrs-weixin
    spec:
      containers:
      - env:
        - name: SVN_URL
          value: svn://10.4.12.48/davesvn/code/config/uat
        - name: SVN_USERNAME
          value: yanglu
        - name: SVN_PASSWORD
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
        image: 10.3.16.213/wechat/weblogicaddapptest1
        imagePullPolicy: IfNotPresent
        name: wxglweblogicaddapp
        ports:
        - containerPort: 7001
          protocol: TCP
        - containerPort: 7002
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
