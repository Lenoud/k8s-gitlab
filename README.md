## 4.gitlab平台的搭建
<a name="QXAvi"></a>
### **4.1部署GitLab**
GitLab主要涉及到3个应用：Redis、Postgresql和Gitlab 核心程序。
<a name="gx7zW"></a>
#### （1）基础准备
所有节点解压软件包并导入镜像：<br /># tar -zxvf CICD-Runner.tar.gz<br /># docker load -i cicd-runner/images/image.tar
<a name="siDOi"></a>
#### （2）部署GitLab服务
master	节点配置好nfs服务器做持久化存储pvc 
```powershell
[root@k8s-master-node1 manifests]# cat /etc/exports
/root/nfs-data/gitlab-conf *(insecure,rw,sync,no_root_squash)
/root/nfs-data/gitlab-log *(insecure,rw,sync,no_root_squash)
/root/nfs-data/gitlab-data *(insecure,rw,sync,no_root_squash)
[root@k8s-master-node1 nfs-data]# exportfs -r  && exportfs
/root/nfs-data/gitlab-conf
                <world>
/root/nfs-data/gitlab-log
                <world>
/root/nfs-data/gitlab-data
                <world>
```

新建命名空间kube-ops，在该命名空间下部署GitLab，YAML文件如下：<br />[root@k8s-master-node1 ~]# cd cicd-runner/manifests/<br />[root@k8s-master-node1 manifests]# vi gitlab-deployment.yaml
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kube-ops
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitlab
  namespace: kube-ops
  labels:
    name: gitlab
spec:
  selector:
    matchLabels:
      name: gitlab
  template:
    metadata:
      name: gitlab
      labels:
        name: gitlab
    spec:
      containers:
      - name: gitlab
        image: yidaoyun/gitlab-ce:v1.0
        imagePullPolicy: IfNotPresent
        env:
        - name: GITLAB_ROOT_PASSWORD
          value: lb781023
        - name: GITLAB_ROOT_EMAIL
          value: 351719672@qq.com
        - name: GITLAB_HOST
          value: 192.168.100.101  #masterIP
        - name: GITLAB_PORT
          value: "80"
        - name: GITLAB_SSH_PORT
          value: "22"
        ports:
        - name: http
          containerPort: 80
        - name: ssh
          containerPort: 22
        volumeMounts:
        - mountPath: /var/opt/gitlab
          name: gitlab-data
        - mountPath: /var/log/gitlab
          name: gitlab-log
        - mountPath: /etc/gitlab
          name: gitlab-conf
      volumes:
      - name: gitlab-data
        persistentVolumeClaim:
          claimName: gitlab-data
      - name: gitlab-conf
        persistentVolumeClaim:
          claimName: gitlab-conf
      - name: gitlab-log
        persistentVolumeClaim:
          claimName: gitlab-log
---
apiVersion: v1
kind: Service
metadata:
  name: gitlab
  namespace: kube-ops
  labels:
    name: gitlab
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: http
      nodePort: 30880
    - name: ssh
      port: 22
      targetPort: ssh
      nodePort: 32222
  selector:
    name: gitlab
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: gitlab-pv
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 15Gi
  nfs:
    path: /root/nfs-data/gitlab-data
    server: 192.168.100.101
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  volumeMode: Filesystem
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: gitlab-conf
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 2Gi
  nfs:
    path: /root/nfs-data/gitlab-conf
    server: 192.168.100.101
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  volumeMode: Filesystem
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: gitlab-log
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 1Gi
  nfs:
    path: /root/nfs-data/gitlab-log
    server: 192.168.100.101
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  volumeMode: Filesystem
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitlab-data
  namespace: kube-ops
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 15G
  storageClassName: nfs
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitlab-conf
  namespace: kube-ops
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2G
  storageClassName: nfs
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitlab-log
  namespace: kube-ops
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1G
  storageClassName: nfs

```
**创建资源：**
```powershell
[root@k8s-master-node1 manifests]# kubectl apply -f gitlab-deployment.yaml
```
**查看Pod：**<br />[root@k8s-master-node1 manifests]# kubectl -n kube-ops get pods<br />NAME                      READY   STATUS    RESTARTS   AGE<br />gitlab-8656b798ff-z54sm          1/1       Running    0            3m23s<br />![image.png](https://cdn.nlark.com/yuque/0/2023/png/35492615/1688641599689-5eae3349-5497-4aaa-a9d3-8fb678c63534.png#averageHue=%2323201f&clientId=u31008168-8d8e-4&from=paste&height=182&id=u74fdf057&originHeight=200&originWidth=1062&originalType=binary&ratio=1.100000023841858&rotation=0&showTitle=false&size=33955&status=done&style=none&taskId=u3d005a17-8bc8-4fa0-96f0-e7e6a811091&title=&width=965.4545245288822)
<a name="htMyi"></a>
#### （3）自定义hosts
查看GitLab Pod的IP地址：<br />[root@k8s-master-node1 manifests]# kubectl -n kube-ops get pods -owide<br />NAME    READY    STATUS    RESTARTS    AGE    IP    NODE    NOMINATED NODE    READINESS GATES<br />gitlab-8656b798ff-z54sm       1/1    Running    0    10m    10.244.1.87    k8s-worker-node1    <none>    <none><br />在集群中自定义hosts添加gitlab Pod的解析：
```powershell
[root@k8s-master-node1 manifests]# kubectl edit configmap coredns -n kube-system
```
........<br />apiVersion: v1<br />data:<br />  Corefile: |<br />    .:53 {<br />        errors<br />        health {<br />           lameduck 5s<br />        }<br />        ready<br />        kubernetes cluster.local in-addr.arpa ip6.arpa {<br />           pods insecure<br />           fallthrough in-addr.arpa ip6.arpa<br />           ttl 30<br />        }<br />## 添加以下字段<br />        hosts {<br />            10.244.1.87 gitlab-8656b798ff-z54sm   <br />            fallthrough<br />        }<br />        prometheus :9153<br />        forward . /etc/resolv.conf {<br />           max_concurrent 1000<br />        }<br />        cache 30<br />        loop<br />        reload<br />        loadbalance<br />    }<br />........<br />[root@k8s-master-node1 manifests]# kubectl -n kube-system rollout restart deployment coredns<br />deployment.apps/coredns restarted<br />![image.png](https://cdn.nlark.com/yuque/0/2023/png/35492615/1688641556466-b4339b9c-5aa4-4240-9bd5-fa84b4791ccd.png#averageHue=%231f1e1d&clientId=u31008168-8d8e-4&from=paste&height=553&id=u77da543d&originHeight=608&originWidth=823&originalType=binary&ratio=1.100000023841858&rotation=0&showTitle=false&size=35587&status=done&style=none&taskId=uee21dbb2-5aac-4d13-9bb4-ebdbbe2f0fc&title=&width=748.1818019654144)
<a name="DPf0c"></a>
#### （4）访问GitLab
查看Service：<br />[root@k8s-master-node1 manifests]# kubectl -n kube-ops get svc<br />NAME     TYPE     CLUSTER-IP   EXTERNAL-IP  PORT(S)            AGE<br />gitlab      NodePort   10.96.0.38    <none>    80:30880/TCP,22:32222/TCP   2m2s<br />通过http://master_ip:30880访问GitLab，如图所示：<br />![image.png](https://cdn.nlark.com/yuque/0/2023/png/35492615/1688642260313-b9be8388-78a8-4f46-b584-87324e11274c.png#averageHue=%23fdfdfc&clientId=ud397a9f3-0d23-4&from=paste&height=591&id=u186b2251&originHeight=650&originWidth=1193&originalType=binary&ratio=1.100000023841858&rotation=0&showTitle=false&size=39525&status=done&style=none&taskId=uee309c97-8841-4d24-8747-753a48895e2&title=&width=1084.5454310385653)<br />登录后：<br />![image.png](https://cdn.nlark.com/yuque/0/2023/png/35492615/1688642988504-30cba939-d659-4c91-bd6f-ac98d1ccb718.png#averageHue=%23fdfdfd&clientId=u44ae5c08-4b4c-4&from=paste&height=641&id=u5f827242&originHeight=705&originWidth=1870&originalType=binary&ratio=1.100000023841858&rotation=0&showTitle=false&size=56782&status=done&style=none&taskId=ub297e70f-f15f-4744-8ad6-2d8ff576ff4&title=&width=1699.9999631534931)
<a name="wiqYx"></a>
## 5.gitlab平台的配置
<a name="EnSQ5"></a>
### 5.1 创建一个新的公开项目
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35492615/1688643414773-e2c3d99a-5836-420b-aa32-28b38dcc4cbb.png#averageHue=%23fcf7f0&clientId=u44ae5c08-4b4c-4&from=paste&height=827&id=u32cf275a&originHeight=910&originWidth=1863&originalType=binary&ratio=1.100000023841858&rotation=0&showTitle=false&size=95305&status=done&style=none&taskId=u96affb4f-8b56-418b-b7c3-2a486a17586&title=&width=1693.636326927785)
<a name="a93kn"></a>
### 5.2 **部署Gitlab CI Runner**
<a name="uuqLd"></a>
#### （1）获取 Gitlab CI Register Token
登录GitLab管理界面(http://master_ip:30880/admin)，然后点击左侧菜单栏中的Runners，如图所示：<br />![image.png](https://cdn.nlark.com/yuque/0/2023/png/35492615/1688643686026-41e7190d-c8c6-492b-a395-c38733dd6b8a.png#averageHue=%23f5f4f2&clientId=u44ae5c08-4b4c-4&from=paste&height=690&id=ubc5e1d03&originHeight=759&originWidth=1100&originalType=binary&ratio=1.100000023841858&rotation=0&showTitle=false&size=92171&status=done&style=none&taskId=ud4407356-8387-4ce6-a422-e9262428861&title=&width=999.9999783255842)<br />可以看到该页面中有两个重要参数：Runner URL和Register Token，后期部署Runner时会用到这两个参数。
<a name="SwbB3"></a>
#### （2）编写Gitlab CI Runner资源清单文件
**首先，通过ConfigMap资源来传递Runner镜像所需的环境变量，**<br />[root@k8s-master-node1 manifests]# vi runner-configmap.yaml
```yaml
apiVersion: v1
data:
  REGISTER_NON_INTERACTIVE: "true" #修改
  REGISTER_LOCKED: "false"
  METRICS_SERVER: "0.0.0.0:9100"
  CI_SERVER_URL: "http://192.168.100.101:30880/"  #添加masterIP+port
  RUNNER_REQUEST_CONCURRENCY: "4" #修改
  RUNNER_EXECUTOR: "kubernetes"
  KUBERNETES_NAMESPACE: "kube-ops" #修改
  KUBERNETES_PRIVILEGED: "true"
  KUBERNETES_CPU_LIMIT: "1"
  KUBERNETES_CPU_REQUEST: "500m"
  KUBERNETES_MEMORY_LIMIT: "1Gi"
  KUBERNETES_SERVICE_CPU_LIMIT: "1"
  KUBERNETES_SERVICE_MEMORY_LIMIT: "1Gi"
  KUBERNETES_HELPER_CPU_LIMIT: "500m"
  KUBERNETES_HELPER_MEMORY_LIMIT: "100Mi"
  KUBERNETES_PULL_POLICY: "if-not-present"
  KUBERNETES_TERMINATIONGRACEPERIODSECONDS: "10"
  KUBERNETES_POLL_INTERVAL: "5"
  KUBERNETES_POLL_TIMEOUT: "360"
kind: ConfigMap
metadata:
  labels:
    app: gitlab-ci-runner
  name: gitlab-ci-runner-cm
  namespace: kube-ops
```
**编写一个用于注册、运行和取消注册Gitlab CI Runner的小脚本。只有当Pod正常通过 Kubernetes（TERM信号）终止时，才会触发转轮取消注册。如果强制终止Pod（SIGKILL信号），Runner将不会注销自身。必须手动完成对这种被杀死的Runner的清理，配置清单文件如下：**<br />[root@k8s-master-node1 manifests]# vi runner-scripts-configmap.yaml
```yaml
apiVersion: v1
data:
  run.sh: |
    #!/bin/bash
    unregister() {
        kill %1
        echo "Unregistering runner ${RUNNER_NAME} ..."
        /usr/bin/gitlab-ci-multi-runner unregister -t "$(/usr/bin/gitlab-ci-multi-runner list 2>&1 | tail -n1 | awk '{print $4}' | cut -d'=' -f2)" -n ${RUNNER_NAME}
        exit $?
    }
    trap 'unregister' EXIT HUP INT QUIT PIPE TERM
    echo "Registering runner ${RUNNER_NAME} ..."
    /usr/bin/gitlab-ci-multi-runner register -r ${GITLAB_CI_TOKEN}
    sed -i 's/^concurrent.*/concurrent = '"${RUNNER_REQUEST_CONCURRENCY}"'/' /home/gitlab-runner/.gitlab-runner/config.toml
    echo "Starting runner ${RUNNER_NAME} ..."
    /usr/bin/gitlab-ci-multi-runner run -n ${RUNNER_NAME} &
    wait
kind: ConfigMap
metadata:
  labels:
    app: gitlab-ci-runner
  name: gitlab-ci-runner-scripts
  namespace: kube-ops
```
**可以看到需要一个GITLAB_CI_TOKEN，然后使用Gitlab CI runner token来创建一个 Kubernetes secret对象。将token进行base64编码：**<br />[root@k8s-master-node1 manifests]# echo 7KFagx5yf_ksN1s7zFsa | base64 -w0<br />N0tGYWd4NXlmX2tzTjFzN3pGc2EK<br />**然后使用上面的token创建一个Secret对象：**<br />[root@k8s-master-node1 manifests]# vi runner-token-secret.yaml
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-ci-token
  namespace: kube-ops
  labels:
    app: gitlab-ci-runner
data:
  GITLAB_CI_TOKEN: N0tGYWd4NXlmX2tzTjFzN3pGc2EK #将token进行base64编码后的字符串
```
**接下来编写用于运行Runner的控制器对象，这里使用Statefulset资源对象。首先，在开始运行的时候，尝试取消注册所有的同名Runner，然后再尝试重新注册并开始运行。另外通过使用envFrom来指定Secrets和ConfigMaps来用作环境变量，对应的资源清单文件如下：**<br />[root@k8s-master-node1 manifests]# vi runner-statefulset.yaml
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: gitlab-ci-runner
  namespace: kube-ops
  labels:
    app: gitlab-ci-runner
spec:
  updateStrategy:
    type: RollingUpdate
  replicas: 2
  serviceName: gitlab-ci-runner
  selector:
    matchLabels:
      app: gitlab-ci-runner
  template:
    metadata:
      labels:
        app: gitlab-ci-runner
    spec:
      volumes:
      - name: gitlab-ci-runner-scripts
        projected:
          sources:
          - configMap:
              name: gitlab-ci-runner-scripts
              items:
              - key: run.sh
                path: run.sh
                mode: 0755
      serviceAccountName: gitlab-ci
      securityContext:
        runAsNonRoot: true
        runAsUser: 999
        supplementalGroups: [999]
      containers:
      - image: yidaoyun/gitlab-runner:v1.0
        name: gitlab-ci-runner
        command:
        - /scripts/run.sh
        envFrom:
        - configMapRef:
            name: gitlab-ci-runner-cm
        - secretRef:
            name: gitlab-ci-token
        env:
        - name: RUNNER_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        ports:
        - containerPort: 9100
          name: http-metrics
          protocol: TCP
        volumeMounts:
        - name: gitlab-ci-runner-scripts
          mountPath: "/scripts"
          readOnly: true
      restartPolicy: Always
```
**上面使用了一个名为gitlab-ci的serviceAccount，新建一个rbac资源清单文件：**<br />[root@k8s-master-node1 manifests]# vi runner-rbac.yaml
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab-ci
  namespace: kube-ops
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: gitlab-ci
  namespace: kube-ops
rules:
  - apiGroups: [""]
    resources: ["*"]
    verbs: ["*"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: gitlab-ci
  namespace: kube-ops
subjects:
  - kind: ServiceAccount
    name: gitlab-ci
    namespace: kube-ops
roleRef:
  kind: Role
  name: gitlab-ci
  apiGroup: rbac.authorization.k8s.io
```
<a name="K6m7h"></a>
#### （3）部署GitLab CI Runner
**创建Runner资源对象：**<br />[root@k8s-master-node1 manifests]# kubectl apply -f runner-configmap.yaml -f runner-scripts-configmap.yaml -f runner-token-secret.yaml -f runner-statefulset.yaml -f runner-rbac.yaml<br />configmap/gitlab-ci-runner-cm created<br />configmap/gitlab-ci-runner-scripts created<br />secret/gitlab-ci-token created<br />statefulset.apps/gitlab-ci-runner created<br />serviceaccount/gitlab-ci created<br />role.rbac.authorization.k8s.io/gitlab-ci created<br />rolebinding.rbac.authorization.k8s.io/gitlab-ci created<br />**查看Pod：**<br />[root@k8s-master-node1 manifests]# kubectl -n kube-ops get pods<br />NAME                 READY   STATUS    RESTARTS   AGE<br />gitlab-8656b798ff-z54sm      1/1       Running     0           32m<br />gitlab-ci-runner-0         1/1       Running     0           32s<br />gitlab-ci-runner-1         1/1       Running     0           85s<br />**回到admin界面查看：**<br />![image.png](https://cdn.nlark.com/yuque/0/2023/png/35492615/1688705558211-9570b12b-d901-4793-98ca-c47bd8616998.png#averageHue=%23f9f7f7&clientId=uf07f5f4d-b952-4&from=paste&height=811&id=ua22c7d63&originHeight=892&originWidth=1772&originalType=binary&ratio=1.100000023841858&rotation=0&showTitle=false&size=104169&status=done&style=none&taskId=ua46424f0-f168-41c3-8d38-bf403770f90&title=&width=1610.9090559935773)<br />可以看到Runner已经注册成功。
<a name="opQmO"></a>
### 5.3 配置GitLab
<a name="iQ4Gg"></a>
#### （1）开启Container Registry
进入我们的项目，依次点击“设置”→“CI/CD”，如图所示：<br />![image.png](https://cdn.nlark.com/yuque/0/2023/png/35492615/1688705890631-6e3e70bd-2454-4e59-b02d-b0f7b2db7e32.png#averageHue=%23fcfcfb&clientId=uf07f5f4d-b952-4&from=paste&height=802&id=u389c259e&originHeight=882&originWidth=1850&originalType=binary&ratio=1.100000023841858&rotation=0&showTitle=false&size=87113&status=done&style=none&taskId=u7c93e3b8-d3a9-4900-8d2b-b7823aafc14&title=&width=1681.8181453657553)<br />展开“变量”栏目，配置镜像仓库相关的参数。  <br />**key : value的形式**
:::info
REGISTRY (192.168.100.101)  #harbor仓库IP<br />REGISTRY_IMAGE（springcloud）<br />REGISTRY_USER（admin） #harbor仓库账号<br />REGISTRY_PASSWORD（Harbor12345）#harbor仓库密码<br />REGISTRY_PROJECT（library）<br />HOST（192.168.100.101） #mater节点IP
:::
<a name="tphZE"></a>
#### （2）添加Kubernetes集群
Kubectl在使用的时候默认会读取当前用户目录下面的~/.kube/config文件来链接集群，所以需要在Gitlab上配置集成Kubernetes集群。<br />在GitLab Admin界面下，点击“设置”，展开“外发请求”，勾选“允许钩子和服务访问本地网络”，如图所示：<br />![image.png](https://cdn.nlark.com/yuque/0/2023/png/35492615/1688709105268-c747ce5d-e25b-4ba8-806b-9592c51a013a.png#averageHue=%23fdfcfc&clientId=ue2b98359-62e2-4&from=paste&height=704&id=ub5311f9d&originHeight=880&originWidth=1850&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=76593&status=done&style=none&taskId=u88f36178-bb5b-411b-9a0d-1aa8cac58e9&title=&width=1480)<br />进入项目，依次点击左侧菜单栏“运维”→“Kubernetes”，点击“添加Kubernetes集群”，选择“Add existing cluster”：<br />![image.png](https://cdn.nlark.com/yuque/0/2023/png/35492615/1688709230351-49c02222-876d-4e73-a417-1f038d6b3cfa.png#averageHue=%23fcfbfa&clientId=ue2b98359-62e2-4&from=paste&height=651&id=u531eeea3&originHeight=814&originWidth=1850&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=84261&status=done&style=none&taskId=u06a23a6f-04c0-48e7-a976-3d410df5a6f&title=&width=1480)
:::info
Kubernetes 集群名称可以随便填写，如k8s-workers。<br />API地址是集群的apiserver的地址，可以通过命令kubectl cluster-info获取：<br />[root@k8s-master-node1 manifests]# kubectl cluster-info<br />Kubernetes control plane is running at [https://apiserver.cluster.local:6443](https://apiserver.cluster.local:6443) #https://masterIP:6443<br />CoreDNS is running at [https://apiserver.cluster.local:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy](https://apiserver.cluster.local:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy)<br />To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.<br />这个项目准备部署在一个名为gitlab的namespace下面，所以首先创建namespace：<br />[root@k8s-master-node1 manifests]# kubectl create ns gitlab<br />namespace/gitlab created<br />CA Certificate即CA证书，可以直接新建一个ServiceAccount，绑定cluster-admin的权限：
:::
[root@k8s-master-node1 manifests]# vi gitlab-sa.yaml
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab
  namespace: gitlab
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: gitlab
  namespace: gitlab
subjects:
  - kind: ServiceAccount
    name: gitlab
    namespace: gitlab
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
```
然后创建上面的ServiceAccount对象：<br />[root@k8s-master-node1 gitlab-runner]# kubectl apply -f gitlab-sa.yaml<br />serviceaccount/gitlab created<br />clusterrolebinding.rbac.authorization.k8s.io/gitlab created<br />通过上面创建的ServiceAccount获取CA证书和Token：<br />[root@k8s-master-node1 manifests]# kubectl get serviceaccount gitlab -n gitlab -o json
:::info
找到token名称<br />    "secrets": [<br />        {<br />            "name": "gitlab-token-8mx4g"<br />        }<br />    ]<br />}
:::
然后根据上面的Secret找到CA证书和token：<br />[root@k8s-master-node1 manifests]# kubectl get secret gitlab-token-8mx4g  -n gitlab -o json
:::info
    "data": {<br />        "ca.crt": "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM2VENDQWRHZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQ0FYRFRJek1EY3dOakV3TWpBME9Wb1lEekl4TWpNd05qRXlNVEF5TURRNVdqQVZNUk13RVFZRApWUVFERXdwcmRXSmxjbTVsZEdWek1JSUJJakFOQmdrcWhraUc5dzBCQVFFRkFBT0NBUThBTUlJQkNnS0NBUUVBCjAyMEFXWmRVb1BGZXRkQk9QWWJQYjlCY1Z4WW9jcnNtSXprcFRmSTN0TmNWeTNHdnhvdkVDNDVIaDY4YWx4cisKaTYwbmJZRHBmSFVUREl0a1pzTnRkRGNOK2pQc3ZyVE1oTnpyb1BDYjVpQXFZeXNrNW0rcnk2RDdlQXppRmI2TwprYVRoNHRmOG9HUEZNYVlDY1BDV2lNc1p1VFR1Q1pWWEkvcmNWTnlqbEhTT1BCbW1FUHFQZkMxLysrZDRITUNtClliclEzTnhqcy84OFBWdHpZRG1uSkFnMi9mcUVTbjNIUzVjdXNhb3k3b2d0dlRiOG9EOUZNOTZoV3haWjdCUmYKcExxSStWRHZZdDYwY0ZDM1pXd1crcFJ2VGh4SVBOVGttekUzcDdweGI5OGtqNGdRWE9XR1ExcERRNFFlTFpNSApSTjZmTWU5TG84WUQ4ZlBkdjVKczJRSURBUUFCbzBJd1FEQU9CZ05WSFE4QkFmOEVCQU1DQXFRd0R3WURWUjBUCkFRSC9CQVV3QXdFQi96QWRCZ05WSFE0RUZnUVVNRDZIcUhTS2pXNWlvMlpNZE1hV1dzSDBsZzR3RFFZSktvWkkKaHZjTkFRRUxCUUFEZ2dFQkFNV2taQ2M3VTVIVlQ4Qk01dUpoOWxNbGN3WHRrRTBFY2ZRV1BadWp3cHFiSWZKNwpRdVI2WEg0c1o4RGJVOTJNNm9qQ1dPenJvNzdPcnVmYWFKa2pDdEVBT1V0VTRkZnZwOHdhSDFhaG56RkxTSFJJCkhMVk1iMEtBRzVRaitNZ2JpemxudWZTRWk0STI5eXRQbGVMdENrR29WWDgxV25MVG5nUGsvZEdvaEU4YTZBZFQKWWxlbVNzWi9VaU0vYlMvcDlGM3Q2anduN0wrNXJFdkVkb2djUkszVTVUWDdubzgwS3hURjBQUnZkS0ptb3B3ego4Q01NY3lBS1dxRlFzekVPK0JrblJGQ1MzVFl6TzF3Y3UrUm1HL2lwbFBJME1acUZORUorY0tlcUFwTkFNd0RGCmFwMmpJUDZNWi9oY3VmTXJ0T2RaNDhzYUc5eDdGckdKOUtSeWt3Zz0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=",<br />        "namespace": "Z2l0bGFi",<br />        "token": "ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNkltVkVWME5MVUU5V1IzQTFkMkZuTUhJeVEySndSVGMzWVhodFpra3dZbkExZGxnM09VMU1iVjlGUldNaWZRLmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUpuYVhSc1lXSWlMQ0pyZFdKbGNtNWxkR1Z6TG1sdkwzTmxjblpwWTJWaFkyTnZkVzUwTDNObFkzSmxkQzV1WVcxbElqb2laMmwwYkdGaUxYUnZhMlZ1TFRodGVEUm5JaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5elpYSjJhV05sTFdGalkyOTFiblF1Ym1GdFpTSTZJbWRwZEd4aFlpSXNJbXQxWW1WeWJtVjBaWE11YVc4dmMyVnlkbWxqWldGalkyOTFiblF2YzJWeWRtbGpaUzFoWTJOdmRXNTBMblZwWkNJNkltTXpZVGxrTURnNUxUUmhOV1F0TkRrNE5TMDRNakJoTFdOa05EQTVNR0kyWVRKak9DSXNJbk4xWWlJNkluTjVjM1JsYlRwelpYSjJhV05sWVdOamIzVnVkRHBuYVhSc1lXSTZaMmwwYkdGaUluMC5QdGZ3ZmJDVEt0dVRPQ1RRYU1kbEhrcWxDekNnWkVWNHRqMHlMQTF3bkFIZEx1YUtkMGlxNk5Yb0FhQWROZ3VJR0Q4N2h1LXpvVkhYcTNZSUppaGJQdWZlWExtajZYdmtvYVBQOTBiWVhRT1FQUnQtMXRXdTZRaS1sYkpodkE3ZEJqeUQ5d3A4Wkwza1hfZE85OG5rNHhqTElYSTFHZ05wWUxBQ2xRc25lVW5nalNxdEo2aXVhcGtZalFwYTJ4SUdVdnYxWDltX3ViSGEwTXZoRUVUMFM0NGdpSFhEcEZlT3h6SER2YkZsNkpfZllZYm1GaHhTWW50WWNUSjdJRF9yVGZTWE5YdVZRc0wzTmcxTlZqV25rOS0xN1Z2TXRjUE1oMlIwdGc0U2hxSGE2MjVFMmhwbzVERWxPZTZ0SS1XeXdvN1JqWjA5dk5JeGpDY0h5UWV2Ync="<br />    },
:::
对获取的值转换格式  echo " " | base64 -d <br />![image.png](https://cdn.nlark.com/yuque/0/2023/png/35492615/1688710191992-c1ae38d7-c0b9-4627-88ba-c190c2d3ae67.png#averageHue=%23302b27&clientId=u33469a75-1266-4&from=paste&height=525&id=u27aacfca&originHeight=578&originWidth=1645&originalType=binary&ratio=1.100000023841858&rotation=0&showTitle=false&size=181027&status=done&style=none&taskId=u6505eea8-895d-4615-8f4d-b3b892b6870&title=&width=1495.4545130414417)<br />![image.png](https://cdn.nlark.com/yuque/0/2023/png/35492615/1688710275220-b6cd66f8-ec33-4ffe-b5af-bad39afdfa19.png#averageHue=%23fbfaf9&clientId=u33469a75-1266-4&from=paste&height=741&id=u0a2d9784&originHeight=815&originWidth=1758&originalType=binary&ratio=1.100000023841858&rotation=0&showTitle=false&size=109145&status=done&style=none&taskId=ub97e98c5-52e8-4a20-9ca0-9b8d1025d3f&title=&width=1598.181783542161)
<a name="kC0uE"></a>
