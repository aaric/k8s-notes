# Kubernets Notes

> Learning with kubernetes.

## 1.Command

```bash
# kubectl概述
# https://kubernetes.io/zh/docs/reference/kubectl/overview

# 获取apiversion版本信息
sh> kubectl api-versions

# 获取资源的apiVersion版本信息
sh> kubectl explain pod
sh> kubectl explain deploy

# 获取完整的yaml信息
sh> kubectl get pod pod-demo -o yaml

# 查看pod的描述信息
sh> kubectl describe pod myapp-pod

# 查看pod日志
sh> kubectl logs myapp-pod -c myapp-container

# 编译已存在的pod的yaml信息
sh> kubectl edit pod myapp-pod
```

## 2.Yaml

### 2.1 Probe

> 探针是由kubelet对容器执行的定期诊断。要执行诊断，kubelet调用由容器实现的Handler。  
> 有三种类型的处理程序：ExecAction、TCPSocketAction、HTTPGetAction。

#### 2.1.1 Init

```bash
# su - admin
sh> tee myapp-pod.yml <<-'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
    - name: myapp-container
      image: busybox:1.28.4
      imagePullPolicy: IfNotPresent
      command: ["/bin/sh", "-c", "echo the app is running! && sleep 3600"]
    - name: init-myservice
      image: busybox:1.28.4
      imagePullPolicy: IfNotPresent
      command: ["/bin/sh", "-c", "until nslookup myservice; do echo waiting for myservice; sleep 2; done"]
    - name: init-mydb
      image: busybox:1.28.4
      imagePullPolicy: IfNotPresent
      command: ["/bin/sh", "-c", "until nslookup mydb; do echo waiting for mydb; sleep 2; done"]
EOF
sh> kubectl create -f myapp-pod.yml
sh> tee myapp-svc.yml <<-'EOF'
kind: Service
apiVersion: v1
metadata:
  name: myservice
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
---
kind: Service
apiVersion: v1
metadata:
  name: mydb
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9377
EOF
sh> kubectl create -f myapp-svc.yml
```

#### 2.1.2 Readiness

```bash
# su - admin
sh> tee readiness.yml <<-'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
    - name: readiness-container
      image: wangyanglinux/myapp:v1
      imagePullPolicy: IfNotPresent
      readinessProbe:
        httpGet:
          port: 80
          path: /index1.html
        initialDelaySeconds: 1
        periodSeconds: 3
EOF
sh> kubectl create -f readiness.yml
sh> kubectl describe pod myapp-pod
```

#### 2.1.3 Liveness

##### A. Exec

```bash
# su - admin
sh> tee liveness-exec.yml <<-'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
    - name: readiness-container
      image: wangyanglinux/myapp:v1
      imagePullPolicy: IfNotPresent
      command: ["/bin/sh","-c","date > /tmp/live; sleep 60; rm -rf /tmp/live"]
      livenessProbe:
        exec:
         command: ["/usr/bin/test", "-e", "/tmp/live"]
        initialDelaySeconds: 1
        periodSeconds: 3
EOF
sh> kubectl create -f liveness-exec.yml
sh> kubectl describe pod myapp-pod
sh> kubectl exec myapp-pod -it -- cat /tmp/live
```

##### B. HTTP

```bash
# su - admin
sh> tee liveness-http.yml <<-'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
    - name: readiness-container
      image: wangyanglinux/myapp:v1
      imagePullPolicy: IfNotPresent
      livenessProbe:
        httpGet:
          port: 80
          path: /index.html
        initialDelaySeconds: 1
        periodSeconds: 3
        timeoutSeconds: 10
EOF
sh> kubectl create -f liveness-http.yml
sh> kubectl describe pod myapp-pod
# cd /usr/share/nginx/html && mv index.html index1.html
sh> kubectl exec myapp-pod -it -- /bin/sh
```

##### B. TCP

```bash
# su - admin
sh> tee liveness-tcp.yml <<-'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
    - name: readiness-container
      image: wangyanglinux/myapp:v1
      imagePullPolicy: IfNotPresent
      livenessProbe:
        tcpSocket:
          port: 80
        initialDelaySeconds: 5
        timeoutSeconds: 1
EOF
sh> kubectl create -f liveness-tcp.yml
sh> kubectl describe pod myapp-pod
# vi /etc/nginx/conf.d/default.conf | 80 -> 8080
sh> kubectl exec myapp-pod -it -- /bin/sh
```

#### 2.1.4 Lifecycle

```bash
# su - admin
sh> tee lifecycle.yml <<-'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
    - name: readiness-container
      image: wangyanglinux/myapp:v1
      imagePullPolicy: IfNotPresent
      lifecycle:
        postStart:
          exec:
            command: ["/bin/sh", "-c", "echo post start > /tmp/pod.log"]
        preStop:
          exec:
            command: ["/bin/sh", "-c", "echo pre stop > /tmp/pod.log"]
EOF
sh> kubectl create -f lifecycle.yml
# tail -f /tmp/pod.log
sh> kubectl exec myapp-pod -it -- /bin/sh
```

### 2.2 Resource

> 工作负载型资源(workload)：Pod、ReplicaSet、Deployment、StatefulSet、DaemonSet、Job、CronJob。

#### 2.2.1 Pod

```bash
# su - admin
sh> tee pod-demo.yml <<-'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pod-demo
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-1
    image: wangyanglinux/myapp:v1
  - name: busybox-1
    image: busybox:latest
    command:
    - "/bin/sh"
    - "-c"
    - "sleep 3600"
EOF
sh> kubectl create -f pod-demo.yml
sh> kubectl logs pod-demo -c myapp-1
sh> kubectl exec pod-demo -c busybox-1 -it -- /bin/sh
```

#### 2.2.2 ReplicaSet

```bash
# su - admin
sh> tee replica-set-demo.yml <<-'EOF'
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-rs
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: wangyanglinux/myapp:v1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
EOF
sh> kubectl create -f replica-set-demo.yml
```

#### 2.2.3 Deployment

```bash
# su - admin
sh> tee deployment-demo.yml <<-'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: wangyanglinux/myapp:v1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
EOF
sh> kubectl apply -f deployment-demo.yml

# scal
sh> kubectl scale deployment myapp-deploy --replicas 3
# autoscale
sh> kubectl autoscale deployment myapp-deploy --min=5 --max=10 --cpu-percent=80
sh> kubectl delete hpa myapp-deploy
# set image
sh> kubectl set image deployment/myapp-deploy myapp=wangyanglinux/myapp:v2
# rollout
sh> kubectl rollout undo deployment/myapp-deploy
sh> kubectl rollout status deployment/myapp-deploy
sh> kubectl get rs
sh> kubectl describe deployments
sh> kubectl rollout history deployment/myapp-deploy
sh> kubectl rollout undo deployment/myapp-deploy --to-revision=3
sh> kubectl rollout pause deployment/myapp-deploy
```

#### 2.2.4 DaemonSet

```bash
# su - admin
sh> tee daemon-set-demo.yml <<-'EOF'
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: myapp-ds
spec:
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: wangyanglinux/myapp:v1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
EOF
sh> kubectl create -f daemon-set-demo.yml
```

#### 2.2.5 Job & CronJob

##### 2.2.5.1 JOb

```bash
# su - admin
sh> tee job-demo.yml <<-'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-job
spec:
  template:
    metadata:
      labels:
        app: perl
    spec:
      containers:
        - name: pi
          image: perl:5.32
          imagePullPolicy: IfNotPresent
          command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
EOF
sh> kubectl create -f job-demo.yml
sh> kubectl logs pi-job-j2dxd
```

##### 2.2.5.2 CronJob

```bash
# su - admin
sh> tee cron-job-demo.yml <<-'EOF'
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: print-cj
spec:
  schedule: "0/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: print
              image: busybox:1.28.4
              imagePullPolicy: IfNotPresent
              args:
                - /bin/sh
                - -c
                - /bin/date; echo hello from cron job
          restartPolicy: OnFailure
EOF
sh> kubectl create -f cron-job-demo.yml
sh> kubectl get cj
sh> kubectl logs print-cj-1603344900-f8m5q
```

#### 2.2.6 StatefullSet

```bash
# su - admin
sh> tee statefull-set-demo.yml <<-'EOF'
EOF
sh> kubectl create -f statefull-set-demo.yml
```

#### 2.2.7 HPA（HorizontalPodAutoScale）

```bash
# su - admin
sh> tee hpa-demo.yml <<-'EOF'
EOF
sh> kubectl create -f hpa-demo.yml
```

### 2.3 Service

#### 2.3.1 Service

> `.spec.type` *Valid options are ExternalName, ClusterIP, NodePort, and LoadBalancer.*

##### 2.3.1.1 ClusterIP - *Default*

```bash
# su - admin
sh> tee clusterip-default-demo.yml <<-'EOF'
apiVersion: v1
kind: Service
metadata:
  name: clusterip-svc
spec:
  selector:
    app: myapp
  #type: ClusterIP
  ports:
    - name: http
      port: 80
      targetPort: 80
EOF
sh> kubectl apply -f clusterip-default-demo.yml
```

##### 2.3.1.2 ClusterIP - *Headless Service*

```bash
# su - admin
sh> tee clusterip-headless-demo.yml <<-'EOF'
apiVersion: v1
kind: Service
metadata:
  name: clusterip-svc
spec:
  selector:
    app: myapp
  #type: ClusterIP
  clusterIP: "None"
  ports:
    - name: http
      port: 80
      targetPort: 80
EOF
sh> kubectl apply -f clusterip-headless-demo.yml
# yum -y install bind-utils
sh> dig -t A clusterip-svc.default.svc.cluster.local. @10.244.0.6
```

##### 2.3.1.3 NodePort

```bash
# su - admin
sh> tee nodeport-demo.yml <<-'EOF'
apiVersion: v1
kind: Service
metadata:
  name: nodeport-svc
spec:
  selector:
    app: myapp
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: 80
EOF
sh> kubectl apply -f nodeport-demo.yml
sh> iptables -t nat -nvL KUBE-NODEPORTS
```

##### 2.3.1.4 ExternalName

```bash
# su - admin
sh> tee externalname-demo.yml <<-'EOF'
apiVersion: v1
kind: Service
metadata:
  name: externalname-svc
spec:
  type: ExternalName
  externalName: www.baidu.com
EOF
sh> kubectl apply -f externalname-demo.yml
sh> dig -t A externalname-svc.default.svc.cluster.local. @10.244.0.6
```

#### 2.3.2 Ingress

> [Getting Started - `ingress-nginx`](https://kubernetes.github.io/ingress-nginx/)

```bash
# su - admin
（待完善）
```

### 2.4 Storage

> 特殊类型的存储卷：ConfigMap(当配置中心来使用的资源类型)、Secret(保存敏感数据)、DownwardAPI(把外部环境中的信息输出给容器)。

#### 2.4.1 ConfigMap

```bash
# su - admin
# --from-file
sh> tee appconfig.properties <<-'EOF'
app.name=Hello Project
app.version=1.0.0
EOF
sh> kubectl create configmap app-config-1 --from-file=./appconfig.properties
sh> kubectl get configmaps app-config-1 -o yaml
sh> kubectl delete configmap app-config-1

# --from-literal
sh> kubectl create configmap app-config-2 --from-literal=app.name=app2 \
  --from-literal=app.verson=2.0
sh> kubectl get configmaps app-config-2 -o yaml
sh> kubectl delete configmap app-config-2

# env
sh> tee configmap-demo.yml <<-'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-cm
data:
  app.name: Hello Project
  app.version: 1.0.0
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-env-cm
data:
  log_level: INFO
---
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
    - name: myapp
      image: wangyanglinux/myapp:v1
      imagePullPolicy: IfNotPresent
      command: ["/bin/sh", "-c", "env"]
      env:
        - name: APP_NAME
          valueFrom:
            configMapKeyRef:
               name: myapp-cm
               key: app.name
        - name: APP_VERSION
          valueFrom:
            configMapKeyRef:
               name: myapp-cm
               key: app.version
      envFrom:
        - configMapRef:
            name: myapp-env-cm
  restartPolicy: Never
EOF
sh> kubectl create -f configmap-demo.yml
sh> kubectl logs myapp-pod

# volume
sh> tee configmap-volume-demo.yml <<-'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-volume-cm
data:
  app.name: Hello Project
  app.version: 1.0.0
---
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
    - name: myapp
      image: wangyanglinux/myapp:v1
      imagePullPolicy: IfNotPresent
      command: ["/bin/sh", "-c", "cat /etc/config/app.name"]
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: myapp-volume-cm
  restartPolicy: Never
EOF
sh> kubectl create -f configmap-volume-demo.yml
sh> kubectl logs myapp-pod

# update config
sh> tee configmap-update-demo.yml <<-'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config-cm
data:
  log_level: INFO
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: wangyanglinux/myapp:v1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
          volumeMounts:
            - name: config-volume
              mountPath: /etc/config
      volumes:
        - name: config-volume
          configMap:
            name: myapp-config-cm
EOF
sh> kubectl apply -f configmap-update-demo.yml
sh> kubectl exec `kubectl get pod -l app=myapp -o=name | cut -d '/' -f2` \
  /bin/sh -- cat /etc/config/log_level
sh> kubectl edit configmap myapp-config-cm
```

#### 2.4.2 Secret

```bash
# su - admin
# serviceaccout
sh> kubectl run myapp-pod --image wangyanglinux/myapp:v1
sh> kubectl exec myapp-pod -- ls /run/secrets/kubernetes.io/serviceaccount

# opaque
sh> echo -n 'admin' | base64
YWRtaW4=
sh> tee secret-demo.yml <<-'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: YWRtaW4=
EOF
sh> kubectl create -f secret-demo.yml

# volume
sh> tee secret-volume-demo.yml <<-'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
    - name: myapp
      image: wangyanglinux/myapp:v1
      imagePullPolicy: IfNotPresent
      volumeMounts:
        - name: secret-volume
          mountPath: ''
          readOnly: true
  volumes:
    - name: secret-volume
      secret:
        secretName: mysecret
EOF
sh> kubectl create -f secret-volume-demo.yml

# env
sh> tee secret-env-demo.yml <<-'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: wangyanglinux/myapp:v1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
          env:
            - name: TEST_USER
              valueFrom:
                secretKeyRef:
                  name: mysecret
                  key: username
            - name: TEST_PWD
              valueFrom:
                secretKeyRef:
                  name: mysecret
                  key: password
EOF
sh> kubectl apply -f secret-env-demo.yml
sh> kubectl exec myapp-deploy-7cbc587967-hsxb6 -- env
```

#### 2.4.3 DownwardAPI

```yaml
```

### 2.5 Schedule

```yaml
```

### 2.6 Security

```yaml
```

### 2.7 Helm

```yaml
```
