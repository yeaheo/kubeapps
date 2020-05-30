## Kubernetes 部署 Verddacio 私有仓库

对于 Verddacio，官方的定义是这样的: `Verdaccio is a lightweight private npm proxy registry built in Node.js` 。意思就是Verdaccio 是一个 Node.js 创建的轻量的私有 **Npm Proxy Registry**。它具有以下特性:

- 它是基于 Node.js 的网页应用程序；
- 私有 Npm Registry；
- 本地网络 Proxy；
- 可插入式应用程序；
- 易安装和使用；
- 提供 Docker 和 Kubernetes 支持；
- 与 yarn, npm 和 pnpm 100% 兼容；
- forked 于 sinopia@1.4.0 并且 100% 向后兼容。

官方还有段视频，有条件的可以看一看:

<iframe width="560" height="315" src="https://www.youtube.com/embed/hDIFKzmoCaA" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

为了更符合 Kubernetes 应用的特点，我是在 Kubernetes 集群里部署 Verddacio，然后最外层用的是 Traefik 2.0 版本暴露对应服务，具体环境如下说明：

```yml
Kubernetes 集群版本: v1.16.9
Verddacio 版本：v4.6.2
Traefik 版本：v2.2
存储方式：hostPath
```

> 存储方式需要依据自己实际情况而定，这里为了方便我用的是 `hostPath` 的方式，其他方式可以根据自己应用情况进行更改，因为是私有 npm 仓库，需要缓存包文件，所以持久化存储是必须的，要不部署起来也没有实际意义

还有就是：

> traefik 可以用 nginx-ingress-controller 代替，或者 service 类型直接改为 `NodePort` 根据自己实际情况而定

### 私有仓库工作流程

一般的 NPM 私有仓库的工作流程如下图所示：

![npm registry](https://aericio.oss-cn-beijing.aliyuncs.com/images/blog/private-npm.jpg)

大概流程就是用户 install 后向私有 npm 发起请求，服务器会先查询所请求的这个模块是否是私有模块或已经缓存过的公共模块，如果是则直接返回给用户；如果请求的是一个还没有被缓存的公共模块，那么则会向上游源请求模块并进行缓存后返回给用户。

上游的源可以是 官方仓库:<https://registry.npmjs.org/>，速度奇慢，也可以是淘宝镜像:<https://registry.npm.taobao.org>，速度快。

### 准备部署配置文件

配置文件主要包括配置文件和部署文件，我整理了一份放到了 [GitHub](https://github.com/yeaheo/kubeapps/tree/master/verdaccio) 上，感兴趣的可以下载下来，直接修改。本次部署在 `devops` 命名空间下。

#### Verddacio  Configmap 文件

首先，先创建 Verddacio 的配置文件，然后以 ConfigMap 的形式部署：

```yaml
# source verddacio-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: npm-verdaccio
  labels:
    app: verdaccio
data:
  config.yaml: |-
    storage: /verdaccio/storage/data
    web:
      enable: true
      title: Npm Registry
      gravatar: true
    auth:
      htpasswd:
        file: /verdaccio/storage/htpasswd
        max_users: 1000
    uplinks:
      npmjs:
        url: https://registry.npm.taobao.org/
    packages:
      '@*/*':
        access: $all
        publish: $authenticated
        proxy: npmjs

      '**':
        access: $all
        publish: $authenticated
        proxy: npmjs
    middlewares:
      audit:
        enabled: true
    logs:
      - {type: stdout, format: pretty, level: http}
```

修改完成后直接 apply 即可：

```bash
$ kubectl apply -f verdaccio-cm.yaml --namespace devops
```

#### Verdaccio deployment 文件

准备 deployment 文件，这里我用 `nodeName` 属性将其固定了一台机器上，也可以选择其他调度策略，并且存储用的 `hostPath` 方式，如下:

```yaml
# source verdaccio-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: verdaccio
  name: npm-verdaccio
spec:
  replicas: 1
  selector:
    matchLabels:
      app: verdaccio
  strategy:
    type: Recreate
    rollingUpdate: null
  template:
    metadata:
      labels:
        app: verdaccio
    spec:
      nodeName: 172.30.1.22  # 固定部署主机
      securityContext:
        runAsUser: 100
        runAsGroup: 101
      containers:
        - name: verdaccio
          image: "verdaccio/verdaccio:latest"
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 4873
              name: http
          livenessProbe:
            httpGet:
              path: /-/ping
              port: http
            initialDelaySeconds: 5
          readinessProbe:
            httpGet:
              path: /-/ping
              port: http
            initialDelaySeconds: 5
          resources:
            {}
          volumeMounts:
            - mountPath: /verdaccio/storage
              name: storage
              readOnly: false
            - mountPath: /verdaccio/conf
              name: config
              readOnly: true
      volumes:
      - name: config
        configMap:
          name: npm-verdaccio
      - name: storage
        hostPath:                 # 持久化存储
          path: /mnt/data
          type: DirectoryOrCreate
```

修改完成后，直接 apply 即可：

```bash
$ kubectl apply -f verdaccio-deploy.yaml --namespace devops
```

> 固定节点的方式和持久化存储的方式有很多种，根据自己情况选择即可

还有一个特别需要注意的是：

> 官方的镜像默认使用的是 verdaccio 用户启动服务的，如果我们不修改宿主机目录权限会报权限问题

修改宿主机目录权限：

```bash
## 创建目录
$ mkdir -pv /mnt/data/data
## 创建验证文件
$ touch /mnt/data/htpasswd
## 修改权限
$ chown -R 100:101 /mnt/data/
```

#### Verdaccio Service 文件

```yaml
# source verdaccio-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: npm-verdaccio
  labels:
    app: verdaccio
spec:
  ports:
    - port: 4873
      targetPort: http
      protocol: TCP
      name: 
  selector:
    app: verdaccio
  type: ClusterIP
```

同样修改完成后，直接 apply 即可：

```bash
$ kubectl apply -f verdaccio-svc.yaml --namespace devops
```

#### Ingress 暴露服务 

修改 Ingress 部署文件，这里 Traefik 用的是 CRD 资源 `IngressRoutes` :

```yaml
# source verdaccio-ingress.yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: verdaccio-ui
  labels:
    app: verdaccio
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`registry.npm.devops.io`)
      kind: Rule
      services:
        - name: npm-verdaccio
          port: 4873
```

修改完成后直接 apply 即可：

```bash
$ kubectl apply -f verdaccio-ingress.yaml --namespace devops
```

> 暴露服务的方式有很多种，自己自身情况选择一种即可

### 验证安装情况

上述部署完成后，我们可以验证一下部署情况:

```bash
$ kubectl get all -n devops
NAME                                    READY   STATUS    RESTARTS   AGE
pod/npm-verdaccio-5bf4b84589-5x5gg      1/1     Running   0          5h21m

NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/npm-verdaccio   ClusterIP   10.105.46.53    <none>        4873/TCP   27h

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/npm-verdaccio      1/1     1            1           27h

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/npm-verdaccio-5bf4b84589      1         1         1       8h
replicaset.apps/npm-verdaccio-67ddff8879      0         0         0       5h23m
replicaset.apps/npm-verdaccio-7865566fcf      0         0         0       27h
replicaset.apps/npm-verdaccio-c55db58f6       0         0         0       24h
```

### 测试仓库可用性

#### 设置仓库源

这里我随便用的域名 `registry.npm.devops.io` ，一般没有问题的话可以直接用浏览器访问的（需要手动添加解析），为了测试方便，需要将 npm 的默认源设置为 `http://registry.npm.devops.io`:

```bash
$ npm set registry http://registry.npm.devops.io
```

#### 添加用户/登录

我们需要创建个用户方便上传私有包：

```bash
$ npm adduser --registry http://registry.npm.devops.io
```

> 需要输入用户名、密码、邮箱

#### 上传私有包

创建用户并登录成功后，接下来就可以创建并上传私有包了，具体步骤如下：

```bash
## 准备 nodejs 项目
$ mkdir npmtest
$ cd npmtest && touch index.js
$ npm init

## 上传私有包
$ npm adduser --registry http://registry.npm.devops.io
$ npm publish --registry http://registry.npm.devops.io
```

如果遇到报错可以根据 pod 的日志排错，一般都是因为权限问题。
