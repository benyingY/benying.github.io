#### **Kubernetes**: Up and Running, 2nd Edition 读书笔记

​	DaemonSet 用来在集群的每个节点上运行一个Pod，作为代理或者常驻程序。比如日志代理，监控代理。你也可以使用labels让DaemonSet Pods跑在特定的node上，比如你可以在暴露出去的node上跑专门的入侵检测软件。书上还举例了一种情况，云厂商在升级和扩容集群的时候，往往会删除再重建虚拟机。如果你在节点上需要一些个性化的基础软件，这个时候就会出问题，但是你可以用DaemonSet来解决。



**1 DaemonSet调度**

​	DaemonSet决定了DaemonSet Pod运行在哪个node上，具体的实现是在创建DaemonSet Pod时，会指定nodeName字段。所以DaemonSet Pod无需Kubernetes scheduler去调度。如果有一个新node加入集群，DaemonSet会在该node上新建一个DaemonSet Pod。





**2 新建DaemonSet**

```yaml
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: fluentd
  labels:
    app: fluentd
spec:
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd:v0.14.10
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

DaemonSet的yaml文件，DaemonSet Pod会发布到每个node，除非使用了node selector。



```shell
$ kubectl apply -f fluentd.yaml
daemonset "fluentd" created
```

把DaemonSet提交到集群



```shell
$ kubectl describe daemonset fluentd
Name:           fluentd
Image(s):       fluent/fluentd:v0.14.10
Selector:       app=fluentd
Node-Selector:  <none>
Labels:         app=fluentd
Desired Number of Nodes Scheduled: 3
Current Number of Nodes Scheduled: 3
Number of Nodes Misscheduled: 0
Pods Status:    3 Running / 0 Waiting / 0 Succeeded / 0 Failed
```

查看DaemonSet，可以看到已经有3个DaemonSet Pod在运行



```shell
$ kubectl get pods -o wide
NAME            AGE    NODE
fluentd-1q6c6   13m    k0-default-pool-35609c18-z7tb
fluentd-mwi7h   13m    k0-default-pool-35609c18-ydae
fluentd-zr6l7   13m    k0-default-pool-35609c18-pol3
```

查看Pod与node的信息





**3 限制DaemonSet Pod到特定node**

```shell
$ kubectl label nodes k0-default-pool-35609c18-z7tb ssd=true
node "k0-default-pool-35609c18-z7tb" labeled
```

首先是给node创建label



```shell
$ kubectl get nodes --selector ssd=true
NAME                            STATUS    AGE
k0-default-pool-35609c18-z7tb   Ready     1d
```

使用label selector查看Pod



```yaml
apiVersion: extensions/v1beta1
kind: "DaemonSet"
metadata:
  labels:
    app: nginx
    ssd: "true"
  name: nginx-fast-storage
spec:
  template:
    metadata:
      labels:
        app: nginx
        ssd: "true"
    spec:
      nodeSelector:
        ssd: "true"
      containers:
        - name: nginx
          image: nginx:1.10.0
```

DaemonSet的yaml文件，使用了nodeSelector，这样DaemonSet Pod就会被发布到上述特定node上。

**ps：如果从node上移除labels，相关DaemonSet Pod也会移除**





**4 更新DaemonSet**

​	在Kubernetes 1.6之前，更新DaemonSet的具体方法是手动删除每一个DaemonSet管理的Pod，然后让DaemonSet自动重建。1.6的时候DaemonSet获得了与Deployment同样的滚动更新能力。spec.updateStrategy.type字段的RollingUpdate可以设置滚动更新策略。spec.template字段的任何变动，都会导致滚动更新。

​	DaemonSet的滚动更新有两个参数来控制。

​	spec.minReadySeconds，规定了在更新下一个Pod前，当前Pod必须在多长时间内ready。主要是为了确保在更新下一个Pod前，当前Pod是健康的。

​	spec.updateStrategy.rollingUpdate.maxUnavailable，规定了同时更新的最大并发量。这是一个应用相关的参数，设置为1是安全的，常见的策略，但是更新时间会较长，等于pod数量乘以minReadySeconds。如果增大这个值，滚动更新速度会加快，但是一旦滚动更新失败，影响的node会较多。该值的设定与您的应用和集群环境相关。可行的办法是设置为1，当用户和管理员抱怨滚动更新速度的时候再调整，<(*￣▽￣*)/。



```shell
$ kubectl rollout status daemonSets my-daemon-set
```

显示滚动更新的状态





**5 删除DaemonSet**

```shell
$ kubectl delete -f fluentd.yaml
```

**ps：别删错了**



​	上述语句会删除DaemonSet和DaemonSet管理的所有Pod，如果不想删除Pod，可以指定 --cascade = false。

