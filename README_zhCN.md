## 简介

一个黑石ingress资源对应一个黑石外网lb（lb类型为普通型，计费模式为按流量计费，所属vpc为集群所在vpc）。
黑石lb命名规范：命名空间-ingress名称--16位uuid
tblbc通过黑石ingress的annotation创建不同用途的黑石lb。
<br>

相关annotation     | 介绍
-------- | ---
kubernetes.io/ingress.class: "tke-bm" | 黑石ingress需要指定为tke-bm
ingress.tke.bm.kubernetes.io/cert-id: "Id4idi9c"    | cert-id为合法的黑石lb服务端证书id，会创建https-443的监听器
kubernetes.io/ingress.allow-http: "false"     | 是否创建http-80的监听器（是否支持http的访问）

## 部署到tke黑石集群
#### 创建tblbc权限、角色资源
- kubectl create -f deploy/rbac.yaml
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tblbc
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/
kind: ClusterRole
metadata:
  name: system:controller:tblbc
rules:
- apiGroups: [""]
  resources: ["secrets", "endpoints", "services", "pods", "nodes", "namespaces", "configmaps", "events"]
  verbs: ["get", "list", "watch", "update", "create", "patch"]
- apiGroups: ["extensions"]
  resources: ["ingresses"]
  verbs: ["get", "list", "watch", "update"]
- apiGroups: ["extensions"]
  resources: ["ingresses/status"]
  verbs: ["update"]
---
apiVersion: rbac.authorization.k8s.io/
kind: ClusterRoleBinding
metadata:
  name: system:controller:tblbc
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:controller:tblbc
subjects:
- kind: ServiceAccount
  name: tblbc
  namespace: kube-system
```
#### 创建tblbc deployment
- kubectl create -f  deploy/tblbc.yaml
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: l7-lb-controller
  namespace: kube-system
  labels:
    k8s-app: tke-bm-lb-controller
    kubernetes.io/name: "TBLBC"
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: tke-bm-lb-controller
  template:
    metadata:
      labels:
        k8s-app: tke-bm-lb-controller
        name: tke-bm-lb-controller
    spec:
      serviceAccountName: tblbc
      terminationGracePeriodSeconds: 180
      containers:
      - image: ccr.ccs.tencentyun.com/library/ingress-tke-bm-tblbc-amd64:v0.1
        imagePullPolicy: Always
        name: l7-lb-controller
        volumeMounts:
        - mountPath: /etc/tblbc/tblbc.conf
          name: tblbc-config-volume
          readOnly: true
        - mountPath: /etc/cpminfo
          name: host-info-volume
          readOnly: true
        - mountPath: /opt/ccs_agent/temptoken
          name: temp-token-volume
          readOnly: true
        resources:
          # Request is set to accommodate this pod alongside the other
          # master components on a single core master.
          # TODO: Make resource requirements depend on the size of the cluster
          requests:
            cpu: 10m
            memory: 50Mi
        command:
        # TODO: split this out into args when we no longer need to pipe stdout to a file #6428
        - sh
        - -c
        - 'exec /tblbc --sync-period=180s --running-in-cluster=true --config-file-path=/etc/tblbc/tblbc.conf --running-in-cluster=true -v=3 --logtostderr=true 2>&1'
      volumes:
      - name: tblbc-config-volume
        hostPath:
          path: /etc/kubernetes/qcloud.conf
          type: File
      - name: host-info-volume
        hostPath:
          path: /etc/cpminfo
          type: File
      - name: temp-token-volume
        hostPath:
          path: /opt/ccs_agent/temptoken
          type: FileOrCreate
```
#### 输出验证
```
kubectl get deploy -n kube-system l7-lb-controller
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
l7-lb-controller   1         1         1            1           2h
```
## 使用示例
#### 创建http service
- kubectl create -f example/http-svc.yaml
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        imagePullPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: http-svc
  labels:
    app: nginx
spec:
  type: NodePort
  ports:
  - port: 80  # Port doesn't matter as nodeport is used for Ingress
    targetPort: 80
    protocol: TCP
    name: my-http-port
  selector:
    app: nginx
```
#### service信息（nodePort: 31476）
```
kubectl get svc http-svc
NAME       TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
http-svc   NodePort   10.188.42.170   <none>        80:31476/TCP   5h
```
### 具有默认后端的http ingress（tblbc只关注annotation具有kubernetes.io/ingress.class: "tke-bm" 的ingress）
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test
  annotations:
    kubernetes.io/ingress.class: "tke-bm"
spec:
  backend:
    # This assumes http-svc exists and routes to healthy endpoints.
    serviceName: http-svc
    servicePort: 80

```
### 具有默认后端的https ingress（ingress.tke.bm.kubernetes.io/cert-id: "Id4idi9c" cert-id为黑石证书管理平台服务端证书id，设置该annotation会创建https的监听器）
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: https
  annotations:
    kubernetes.io/ingress.class: "tke-bm"
    ingress.tke.bm.kubernetes.io/cert-id: "Id4idi9c"
spec:
  backend:
    # This assumes http-svc exists and routes to healthy endpoints.
    serviceName: http-svc
    servicePort: 80
```
### 多个host规则的http ingress
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: multi-hosts
  annotations:
    kubernetes.io/ingress.class: "tke-bm"
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /foo
        backend:
          serviceName: http-svc
          servicePort: 80
      - path: /bar
        backend:
          serviceName: http-svc
          servicePort: 80
  - host: www.example.com
    http:
      paths:
      - path: /foo
        backend:
          serviceName: http-svc
          servicePort: 80
      - path: /bar
        backend:
          serviceName: http-svc
          servicePort: 80
```
### 不支持http ingress（kubernetes.io/ingress.allow-http: "false"指示该ingress不支持http）
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: only-https
  annotations:
    kubernetes.io/ingress.class: "tke-bm"
    kubernetes.io/ingress.allow-http: "false"
    ingress.tke.bm.kubernetes.io/cert-id: "Id4idi9c"
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /foo
        backend:
          serviceName: http-svc
          servicePort: 80
      - path: /bar
        backend:
          serviceName: http-svc
          servicePort: 80
  - host: www.example.com
    http:
      paths:
      - path: /foo
        backend:
          serviceName: http-svc
          servicePort: 80
      - path: /bar
        backend:
          serviceName: http-svc
          servicePort: 80
```
#### 输出验证
```
kubectl get ingress
NAME          HOSTS                         ADDRESS       PORTS     AGE
https         *                             118.89.5.54   80        4h
multi-hosts   foo.bar.com,www.example.com   118.89.5.51   80        4h
only-https    foo.bar.com,www.example.com   118.89.5.53   80        1d
test          *                             118.89.5.52   80        4h
```

## 注意事项

- tblbc会根据lb的名称查询是否是黑石ingress对应的lb，尽量不要在黑石lb控制台对lb进行操作
