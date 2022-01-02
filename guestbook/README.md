# guestbook

[使用 Redis 部署 PHP 留言板应用程序](https://kubernetes.io/zh/docs/tutorials/stateless-application/guestbook/)

## 过程

### 集群检查

在部署应用前有必要对集群环境进行检查，确认集群环境是否正常、集群的版本信息，以及集群中的节点是否正常运行。

#### 查看上下文

```bash
> kubectl config get-contexts                  
CURRENT   NAME               CLUSTER            AUTHINFO                               NAMESPACE
          docker-desktop     docker-desktop     docker-desktop                         
*         minikube           minikube           minikube                               default
```

#### 切换上下文

```bash
> kubectl config use-context minikube
CURRENT   NAME               CLUSTER            AUTHINFO                               NAMESPACE
          docker-desktop     docker-desktop     docker-desktop                         
*         minikube           minikube           minikube                               default
```

#### 查看 k8s 版本信息

```bash
> kubectl version
Client Version: version.Info{Major:"1", Minor:"23", GitVersion:"v1.23.1", GitCommit:"86ec240af8cbd1b60bcc4c03c20da9b98005b92e", GitTreeState:"clean", BuildDate:"2021-12-16T11:33:37Z", GoVersion:"go1.17.5", Compiler:"gc", Platform:"darwin/arm64"}
Server Version: version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.4", GitCommit:"b695d79d4f967c403a96986f1750a35eb75e75f1", GitTreeState:"clean", BuildDate:"2021-11-17T15:42:41Z", GoVersion:"go1.16.10", Compiler:"gc", Platform:"linux/arm64"}
```

#### 查看集群状态

如果集群不在运行状态，可以检查一下 `kubelet` 的状态，因为 `kube-apiserver` 是由 `kubelet` 管理的。

```bash
> kubectl cluster-info
Kubernetes control plane is running at https://kubernetes.docker.internal:6443
CoreDNS is running at https://kubernetes.docker.internal:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

#### 查看集群节点，只有单个节点

```bash
> kubectl get no -o wide
NAME             STATUS   ROLES                  AGE     VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE         KERNEL-VERSION     CONTAINER-RUNTIME
docker-desktop   Ready    control-plane,master   3d21h   v1.22.4   192.168.65.4   <none>        Docker Desktop   5.10.76-linuxkit   docker://20.10.11
```

#### 查看 node 说明

```bash
> kubectl explain no
KIND:     Node
VERSION:  v1

DESCRIPTION:
     Node is a worker node in Kubernetes. Each node will have a unique
     identifier in the cache (i.e. in etcd).

FIELDS:
   apiVersion   <string>
     APIVersion defines the versioned schema of this representation of an
     object. Servers should convert recognized schemas to the latest internal
     value, and may reject unrecognized values. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources

   kind <string>
     Kind is a string value representing the REST resource this object
     represents. Servers may infer this from the endpoint the client submits
     requests to. Cannot be updated. In CamelCase. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds

   metadata     <Object>
     Standard object's metadata. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata

   spec <Object>
     Spec defines the behavior of a node.
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status

   status       <Object>
     Most recently observed status of the node. Populated by the system.
     Read-only. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status
```

### 部署后端 Redis

#### 创建主 Redis Deployment [create]

```bash
> kubectl create -f redis-leader-deployment.yaml
deployment.apps/redis-leader created
```

#### 查看 Deployment 状态

```bash
> kubectl get deployments -l app=redis -l role=leader
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
redis-leader   1/1     1            1           63m
```

#### 查看 Pod 状态 [yaml]

```bash
> kubectl get po -l app=redis -l role=leader -o yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2022-01-02T12:43:25Z"
  generateName: redis-leader-5d66d78fcb-
  labels:
    app: redis
    pod-template-hash: 5d66d78fcb
    role: leader
    tier: backend
  name: redis-leader-5d66d78fcb-l2drh
  namespace: default
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: redis-leader-5d66d78fcb
    uid: 942d6581-6ab6-4f5a-b849-1f80ab2e7e21
  resourceVersion: "161467"
  uid: a5fc9aa2-9534-43af-b09b-7240457cda1a
spec:
  containers:
  - image: registry.cn-shenzhen.aliyuncs.com/kubeops/redis:6.0.5
    imagePullPolicy: IfNotPresent
    name: leader
    ports:
    - containerPort: 6379
      protocol: TCP
    resources:
      requests:
        cpu: 100m
        memory: 100Mi
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-kjzqs
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: docker-desktop
  preemptionPolicy: PreemptLowerPriority
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - name: kube-api-access-kjzqs
    projected:
      defaultMode: 420
      sources:
      - serviceAccountToken:
          expirationSeconds: 3607
          path: token
      - configMap:
          items:
          - key: ca.crt
            path: ca.crt
          name: kube-root-ca.crt
      - downwardAPI:
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
            path: namespace
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2022-01-02T12:43:25Z"
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2022-01-02T12:43:26Z"
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2022-01-02T12:43:26Z"
    status: "True"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: "2022-01-02T12:43:25Z"
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: docker://30b948eff36f3bbc4489e4b338e7e95f1150a4c5281e4d33031d9f28027f2819
    image: redis:6.0.5
    imageID: docker-pullable://redis@sha256:800f2587bf3376cb01e6307afe599ddce9439deafbd4fb8562829da96085c9c5
    lastState: {}
    name: leader
    ready: true
    restartCount: 0
    started: true
    state:
      running:
        startedAt: "2022-01-02T12:43:25Z"
  hostIP: 192.168.65.4
  phase: Running
  podIP: 10.1.0.51
  podIPs:
  - ip: 10.1.0.51
  qosClass: Burstable
  startTime: "2022-01-02T12:43:25Z"
```

#### 排查 Pod 启动失败问题

如果 Pod 的 `STATUS` 长时间没有变为 `Running`，可以通过 `kubectl describe pod <POD_NAME>` 查看 Pod 的详细信息。

```bash
> kubectl describe po redis-leader-fb76b4755-nw4x4
Name:         redis-leader-fb76b4755-nw4x4
Namespace:    default
Priority:     0
Node:         docker-desktop/192.168.65.4
Start Time:   Sun, 02 Jan 2022 20:10:26 +0800
Labels:       app=redis
              pod-template-hash=fb76b4755
              role=leader
              tier=backend
Annotations:  <none>
Status:       Running
IP:           10.1.0.46
IPs:
  IP:           10.1.0.46
Controlled By:  ReplicaSet/redis-leader-fb76b4755
Containers:
  leader:
    Container ID:   docker://8df17907fec2b38966ec46c9d71bfc89fe77519952df327449d54150c4e1384b
    Image:          docker.io/redis:6.0.5
    Image ID:       docker-pullable://redis@sha256:800f2587bf3376cb01e6307afe599ddce9439deafbd4fb8562829da96085c9c5
    Port:           6379/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Sun, 02 Jan 2022 20:10:27 +0800
    Ready:          True
    Restart Count:  0
    Requests:
      cpu:        100m
      memory:     100Mi
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-ggts8 (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  kube-api-access-ggts8:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Burstable
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  4m9s  default-scheduler  Successfully assigned default/redis-leader-fb76b4755-nw4x4 to docker-desktop
  Normal  Pulled     4m8s  kubelet            Container image "docker.io/redis:6.0.5" already present on machine
  Normal  Created    4m8s  kubelet            Created container leader
  Normal  Started    4m8s  kubelet            Started container leader
```

#### 创建主 Redis service

```bash
> kubectl create -f redis-leader-service.yaml
service/redis-leader created
```

#### 查看主 Redis service 状态 [wide]

```bash
> kubectl get svc -l app=redis -l role=leader -o wide
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE   SELECTOR
redis-leader   ClusterIP   10.105.105.38   <none>        6379/TCP   23m   app=redis,role=leader,tier=backend
```

#### 创建两个从 Redis 副本

```bash
> kubectl create -f redis-follower-deployment.yaml
deployment.apps/redis-follower created
```

#### 验证两个从 Redis 副本在运行

```bash
> kubectl get po -l app=redis -l role=follower
NAME                              READY   STATUS    RESTARTS   AGE
redis-follower-74cc7db576-fstd2   1/1     Running   0          106s
redis-follower-74cc7db576-kwbsc   1/1     Running   0          106s
```

#### 创建从 Redis service

```bash
> kubectl create -f redis-follower-service.yaml
service/redis-follower created
```

#### 查看从 Redis service 状态 [json]

```bash
> kubectl get svc -l app=redis -l role=follower -o json
{
    "apiVersion": "v1",
    "items": [
        {
            "apiVersion": "v1",
            "kind": "Service",
            "metadata": {
                "creationTimestamp": "2022-01-02T12:46:02Z",
                "labels": {
                    "app": "redis",
                    "role": "follower",
                    "tier": "backend"
                },
                "name": "redis-follower",
                "namespace": "default",
                "resourceVersion": "161635",
                "uid": "92c2298a-cc31-4c3b-b1fb-bdbf25f80975"
            },
            "spec": {
                "clusterIP": "10.105.1.89",
                "clusterIPs": [
                    "10.105.1.89"
                ],
                "internalTrafficPolicy": "Cluster",
                "ipFamilies": [
                    "IPv4"
                ],
                "ipFamilyPolicy": "SingleStack",
                "ports": [
                    {
                        "port": 6379,
                        "protocol": "TCP",
                        "targetPort": 6379
                    }
                ],
                "selector": {
                    "app": "redis",
                    "role": "follower",
                    "tier": "backend"
                },
                "sessionAffinity": "None",
                "type": "ClusterIP"
            },
            "status": {
                "loadBalancer": {}
            }
        }
    ],
    "kind": "List",
    "metadata": {
        "resourceVersion": "",
        "selfLink": ""
    }
}
```

### 部署留言板前端

#### 创建前端 Deployment [apply]

```bash
> kubectl apply -f frontend-deployment.yaml
```

#### 验证三个前端副本正在运行

```bash
> kubectl get po -l app=guestbook -l tier=frontend -o wide
NAME                        READY   STATUS    RESTARTS   AGE     IP          NODE             NOMINATED NODE   READINESS GATES
frontend-57756596cb-m2jj7   1/1     Running   0          6m51s   10.1.0.54   docker-desktop   <none>           <none>
frontend-57756596cb-skz9h   1/1     Running   0          6m51s   10.1.0.53   docker-desktop   <none>           <none>
frontend-57756596cb-xqc6n   1/1     Running   0          6m51s   10.1.0.55   docker-desktop   <none>           <none>
```

#### 创建前端服务

```bash
> kubectl apply -f frontend-service.yaml
service/frontend created
```

#### 验证前端服务正在运行

```bash
> kubectl get svc -l tier=frontend
NAME       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
frontend   ClusterIP   10.105.140.183   <none>        80/TCP    18m
```

#### 通过 `kubectl port-forward` 查看前端服务

运行以下命令将本机的 8080 端口转发到服务的 80 端口。

```bash
> kubectl port-forward svc/frontend 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

#### 扩展 Web 前端

```bash
> kubectl scale deployment frontend --replicas=5
```

#### 验证正在运行的前端 Pod 的数量

```bash
> kubectl get po -l app=guestbook -l tier=frontend
NAME                        READY   STATUS    RESTARTS   AGE
frontend-57756596cb-g76s5   1/1     Running   0          46s
frontend-57756596cb-h8xhm   1/1     Running   0          46s
frontend-57756596cb-m2jj7   1/1     Running   0          38m
frontend-57756596cb-skz9h   1/1     Running   0          38m
frontend-57756596cb-xqc6n   1/1     Running   0          38m
```

#### 查看 Redis Pod 日志

```bash
> kubectl logs redis-leader-5d66d78fcb-pvd84
1:C 02 Jan 2022 12:44:53.370 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:C 02 Jan 2022 12:44:53.370 # Redis version=6.0.5, bits=64, commit=00000000, modified=0, pid=1, just started
1:C 02 Jan 2022 12:44:53.370 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
1:M 02 Jan 2022 12:44:53.370 * Running mode=standalone, port=6379.
1:M 02 Jan 2022 12:44:53.370 # Server initialized
1:M 02 Jan 2022 12:44:53.370 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
1:M 02 Jan 2022 12:44:53.371 * Ready to accept connections
1:M 02 Jan 2022 12:44:53.975 * Replica 10.1.0.50:6379 asks for synchronization
1:M 02 Jan 2022 12:44:53.975 * Partial resynchronization not accepted: Replication ID mismatch (Replica asked for '5f3f86fcc80e66d778048b0d0876f094154ebb65', my replication IDs are 'ef5856ce660b088d5e8f1a75a50d07d624d18326' and '0000000000000000000000000000000000000000')
1:M 02 Jan 2022 12:44:53.975 * Replication backlog created, my new replication IDs are '6e7879f5f854ed376103ea1eb10c5579d21f2d64' and '0000000000000000000000000000000000000000'
1:M 02 Jan 2022 12:44:53.975 * Starting BGSAVE for SYNC with target: disk
1:M 02 Jan 2022 12:44:53.975 * Background saving started by pid 21
1:M 02 Jan 2022 12:44:53.976 * Replica 10.1.0.49:6379 asks for synchronization
1:M 02 Jan 2022 12:44:53.977 * Partial resynchronization not accepted: Replication ID mismatch (Replica asked for '5f3f86fcc80e66d778048b0d0876f094154ebb65', my replication IDs are '6e7879f5f854ed376103ea1eb10c5579d21f2d64' and '0000000000000000000000000000000000000000')
1:M 02 Jan 2022 12:44:53.977 * Waiting for end of BGSAVE for SYNC
21:C 02 Jan 2022 12:44:53.978 * DB saved on disk
21:C 02 Jan 2022 12:44:53.979 * RDB: 0 MB of memory used by copy-on-write
1:M 02 Jan 2022 12:44:54.073 * Background saving terminated with success
1:M 02 Jan 2022 12:44:54.073 * Synchronization with replica 10.1.0.50:6379 succeeded
1:M 02 Jan 2022 12:44:54.073 * Synchronization with replica 10.1.0.49:6379 succeeded
```

#### 进入 Redis 容器查看数据

```bash
> kubectl exec -it redis-leader-5d66d78fcb-pvd84 -- redis-cli
127.0.0.1:6379> KEYS *
1) "guestbook"
127.0.0.1:6379> GET guestbook
",test,hello"
```

### 后续维护

#### 给所有 Redis Pod 添加 status 标签

```bash
> kubectl label pods -l app=redis status=healthy
pod/redis-follower-74cc7db576-fstd2 labeled
pod/redis-follower-74cc7db576-kwbsc labeled
pod/redis-leader-5d66d78fcb-pvd84 labeled
```

#### 查看 Redis Pod 的标签 [--show-labels]

```bash
> kubectl get po -l app=redis --show-labels                    
NAME                              READY   STATUS    RESTARTS   AGE   LABELS
redis-follower-74cc7db576-fstd2   1/1     Running   0          96m   app=redis,pod-template-hash=74cc7db576,role=follower,status=healthy,tier=backend
redis-follower-74cc7db576-kwbsc   1/1     Running   0          96m   app=redis,pod-template-hash=74cc7db576,role=follower,status=healthy,tier=backend
redis-leader-5d66d78fcb-pvd84     1/1     Running   0          85m   app=redis,pod-template-hash=5d66d78fcb,role=leader,status=healthy,tier=backend
```

#### 更新主 Redis Pod 的标签

```bash
> kubectl label pods -l app=redis -l role=leader status=unhealthy --overwrite
pod/redis-leader-5d66d78fcb-pvd84
```

#### 查看主 Redis 资源使用情况

```bash
> kubectl top po -l app=redis -l role=leader
NAME                            CPU(cores)   MEMORY(bytes)   
redis-leader-5d66d78fcb-9bm92   2m           3Mi
```

#### 使用 edit 更新从 Redis Pod 的副本数量

```bash
> kubectl edit deploy/redis-follower
deployment.apps/redis-follower edited

# 查看 Pod 副本数量，从 2 变成 3
> kubectl get pods -l app=redis -l role=follower
NAME                              READY   STATUS    RESTARTS   AGE
redis-follower-74cc7db576-cvf2q   1/1     Running   0          9m29s
redis-follower-74cc7db576-mz6h8   1/1     Running   0          25s
redis-follower-74cc7db576-twq9x   1/1     Running   0          9m29s
```

### 解决镜像拉取问题

由于 guestbook 前端镜像是在谷歌的 gcr.io 上，所以如果没有代理是无拉取的，这里选择在香港服务器上将镜像同步到阿里云容器镜像服务上。

#### 拉取镜像

```bash
> docker pull gcr.io/google_samples/gb-frontend:v5
v5: Pulling from google_samples/gb-frontend
72a69066d2fe: Pull complete 
fbf13c1e88c3: Pull complete 
cddf91161400: Pull complete 
2c396aa97b98: Pull complete 
1d9707294ce1: Pull complete 
443be0efd1a3: Pull complete 
f40e54f5a6bb: Pull complete 
449e25c19260: Pull complete 
4116245e7948: Pull complete 
063a257bdaed: Pull complete 
ba6b06b0aa4f: Pull complete 
331cb0169fcf: Pull complete 
7889266700ad: Pull complete 
2648f8d3ecd7: Pull complete 
2b19c0592f6e: Pull complete 
c3b640245fb3: Pull complete 
4a8a6bc16a1b: Pull complete 
Digest: sha256:1ffc7816e028b2e2f2b592594383a0139b9f570ff5fcc5fdfd81806aa8d403bf
Status: Downloaded newer image for gcr.io/google_samples/gb-frontend:v5
gcr.io/google_samples/gb-frontend:v5
```

#### 重命名镜像

```bash
> docker tag gcr.io/google_samples/gb-frontend:v5
```

#### 登录阿里云容器镜像服务

```bash
> docker login registry.cn-shenzhen.aliyuncs.com -ukubeops
Password: 
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

#### 推送镜像到阿里云容器镜像服务

```bash
> docker push registry.cn-shenzhen.aliyuncs.com/kubeops/gb-frontend:v5
The push refers to repository [registry.cn-shenzhen.aliyuncs.com/kubeops/gb-frontend]
6fd1212aa9ea: Pushed 
76f17e2309c9: Pushed 
4b6e88f9653e: Pushed 
2aecaf70d382: Pushed 
80cd58a1bab0: Pushed 
2e559a423c71: Pushed 
ede4b550d621: Pushed 
2dba6c06bdde: Pushed 
ecd6a695b6ea: Pushed 
b01bf213b941: Pushed 
0ccbc08ded6d: Pushed 
449cff66aba3: Pushed 
643f1d079de7: Pushed 
0104a3ee0257: Pushed 
fd3ee495df7c: Pushed 
00f117848faf: Pushed 
ad6b69b54919: Pushed 
v5: digest: sha256:1ffc7816e028b2e2f2b592594383a0139b9f570ff5fcc5fdfd81806aa8d403bf size: 3876
```

#### 更新镜像

```bash
> kubectl set image deploy/redis-follower follower=registry.cn-shenzhen.aliyuncs.com/kubeops/gb-redis-follower:v2
deployment.apps/redis-follower image updated
```

### 使用 Kustomize 部署

#### 添加 kustomization.yaml 配置文件

```yaml
resources:
  - redis-leader-service.yaml
  - redis-leader-deployment.yaml
  - redis-follower-service.yaml
  - redis-follower-deployment.yaml
  - frontend-service.yaml
  - frontend-deployment.yaml
```

#### 删除旧的部署

之前的所有资源这个时候也可以一次性删除。

```bash
> kubectl delete -k .
service "frontend" deleted
service "redis-follower" deleted
service "redis-leader" deleted
deployment.apps "frontend" deleted
deployment.apps "redis-follower" deleted
deployment.apps "redis-leader" deleted
```

#### 一键部署

```bash
> kubectl apply -k .
service/frontend created
service/redis-follower created
service/redis-leader created
deployment.apps/frontend created
deployment.apps/redis-follower created
deployment.apps/redis-leader created
```

### 回滚

回滚是很常见的操作，这里我们使用 `kubectl rollout undo` 来实现。

#### 查看 Deployment 版本

```bash
> kubectl rollout history deploy redis-follower
deployment.apps/redis-follower
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>
```

#### 回滚到上一个版本

```bash
> kubectl rollout undo deploy redis-follower
```

#### 回滚到指定版本

```bash
> kubectl rollout undo deploy redis-follower --to-revision=1
```

### Pod 水平自动伸缩

相较 `kubectl scale`，这个能力更加强大，他会根据当前资源的使用情况，自动进行伸缩。

#### 创建 Horizontal Pod Autoscaler

`--cpu-percent=80` 指的是 HPA 将（通过 Deployment）增加或者减少 Pod 副本的数量以保持所有 Pod 的平均 CPU 利用率在 80% 左右。

```bash
> kubectl autoscale deploy frontend --min=5 --max=10 --cpu-percent=80
horizontalpodautoscaler.autoscaling/frontend autoscaled
```

### 查看 Autoscaler 的状态

```bash
> kubectl get hpa
NAME       REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
frontend   Deployment/frontend   1%/80%    5         10        5          3m32s
````
