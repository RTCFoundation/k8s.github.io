### 为什么使用helm

k8s上的应用对象，都是由特定的资源描述组成，包括deployment、service等，保存各自文件中或者集中写到一个配置文件，最后通过kubectl apply -f部署

如果应用只是由一个或几个这样的服务组成，上面的部署方式就ok了

但对于一个复杂的应用，会有很多类似上面的资源描述文件。例如微服务架构应用，组成应用的服务可能多达几十个。

如果有更新或回滚应用的需求，可能要修改和维护所涉及大量资源文件，所以这种组织和管理应用的方式就显得力不从心了

由于缺少对发布的应用版本管理和控制，使kubernetes上的应用维护和更新等面临诸多的挑战，主要面临一下问题：

1. 如何将这些服务作为一个整体管理

2. 这些资源文件如何高效复用

3. 不支持应用级别的版本管理

### Helm介绍

Helm上一个kubernetes的包管理工具，类似linux下的apt/yum等，可以方便的将之前打包好的yaml文件部署到k8s上

Helm有三个重要概念

- Helm：命令行客户端工具，主要用于k8s应用chart的创建、打包、发布和管理

- Chart：应用描述，一系列用于描述k8s资源相关文件的集合

- Release：基于Chart的部署实体，一个chart运行生成一个对应的Release，将在k8s中创建真实运行的资源对象

### Helm客户端

##### helm 常见的命令

| 命令       | 描述                                                         |
| ---------- | ------------------------------------------------------------ |
| create     | 创建一个chart并指定名字                                      |
| dependency | 管理chart依赖                                                |
| get        | 下载一个release，可用子命令：all、hooks、manifest、notes、values |
| history    | 获取release历史                                              |
| install    | 安装一个chart                                                |
| list       | 列出release                                                  |
| package    | 将chart目录大报道chart存档文件中                             |
| pull       | 从远程仓库中下载chart并解压到本地 如helm pull stable/mysql --untar |
| repo       | 添加，列出，删除，更新和索引chart仓库，可用子命令：add、index、list、remove、update |
| rollback   | 从之前版本回滚                                               |
| search     | 根据关键字搜索chart，可用子命令：hub，repo                   |
| show       | 查看chart详细信息，可用子命令：all、chart、readme、values    |
| Status     | 显示已命名版本的状态                                         |
| template   | 本地呈现模版                                                 |
| uninstall  | 卸载一个release                                              |
| upgrade    | 更新一个release                                              |
| version    | 查看helm客户端版本                                           |

##### 配置国内chart仓库

- 微软仓库（http://mirror.azure.cn/kubernetes/charts/）这个仓库推荐，基本上官网有的chart这里都有。

- 阿里云仓库（https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts ）

- 官方仓库 （ https://hub.kubeapps.com/charts/incubator）官方chart仓库，国内有点不好使。官方chart仓库，国内有点不好使。

添加存储库：

```
$ helm repo add azure http://mirror.azure.cn/kubernetes/charts
$ helm repo add aliyun https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts 
```

查看配置的存储库：

```
$ helm repo list
NAME    URL                                                   
azure   http://mirror.azure.cn/kubernetes/charts              
aliyun  https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
$ helm search repo azure
```

删除存储库

```
$ helm repo remove aliyun
```

##### Helm基本使用

- chart模版配置

```
$ mkdir ~/lesson/helm/demo2 -p && cd ~/lesson/helm/demo2
$ kubectl create deploy nginx --image=nginx:1.16 --dry-run=client -o yaml > deployment.yml
$ vim deployment.yml 
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.16
        name: nginx

$ kubectl create deploy nginx --image=nginx:1.16
$ kubectl expose deploy nginx --port=80 --target-port=80 --type=NodePort nginx --dry-run=client -o yaml > service.yaml
$ vim service.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  type: NodePort
```

- 按照上述配置放入helm

```
$ cd ~/lesson/helm/mychart/templates/ && rm -fr *
$ mv ~/lesson/helm/demo2/* ./
```

- helm发布升级和回滚

```
$ helm install web1 mychart   #发布
$ vim mychart/values.yml
image:
  repository: nginx
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: "1.16"
$ vim mychart/templates/deploy.yaml
   - image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
        name: nginx
$ helm upgrade web1 mychart --set image.tag="1.17"     #手动指定版本升级
$ helm history web1             #查看历史
$ kubectl get pod,svc           #获取svc ip
$ curl 10.0.0.231 -I            #测试svc返回的nginx是1.17
$ helm rollback web1 1          #回滚到1版本
$ curl 10.0.0.231 -I            #测试svc返回的nginx是1.16
```

- 打包推送charts仓库共享给别人使用

```
$ helm package mychart
Successfully packaged chart and saved it to: /root/lesson/helm/mychart-0.1.0.tgz
$ helm install mychart-0.1.0.tgz   #其他人也能安装使用
$ helm uninstall web1       #卸载应用
```

### 开发chart

开发chart大致流程

1. 先创建模版 helm create demo
2. 修改Chart.yaml，Values.yaml，添加常用的变量

3. 在templates目录下创建部署镜像所需要的yaml文件，并使用变量替换yaml里经常变动的字段

##### 创建模版

```
$  cd ~/lesson/helm/
$ helm create demo
```

##### 修改Chart.yaml，Values.yaml，添加常用的变量

- 修改values.yaml

```
$vim values.yaml 
replicaCount: 1
image:
  repository: test/java-demo
  tag: "latest"

service:
  type: ClusterIP
  port: 80
  targetport: 8080

ingress:
  enabled: false
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  hosts: example.cropy.cn
  tls: []
  #  - secretName: example-cropy-cn-tls

resources:
  limits:
    cpu: 2000m
    memory: 1Gi
  requests:
    cpu: 100m
    memory: 128Mi
```

- 修改_helpers.tpl（子模块）

```
$ vim templates/_helpers.tpl
{{- define "demo.fullname" -}}
{{- .Chart.Name }}-{{ .Release.Name }}
{{- end -}}
{{/*标签选择器*/}}
{{- define "demo.selectorLabels" -}}
app: {{ template "demo.fullname" . }}
release: {{ .Release.Name }}
{{- end -}}
{{/*公共标签选择器*/}}
{{- define "demo.labels" -}}
app: {{ template "demo.fullname" . }}
release: {{ .Release.Name }}
chart: {{ .Chart.Name }}
{{- end -}}
```

##### 在templates目录下创建部署镜像所需要的yaml文件，并使用变量替换yaml里经常变动的字段

- 清除生成的yaml文件，并创建NOTES.txt和_helpers.tpl

```
$ cd demo/templates/ && rm -fr *
$ touch NOTES.txt && touch _helpers.tpl
```

创建deployment.yaml

```
$ kubectl create deployment web --image=nginx:1.16 --dry-run=client -o yaml > deployment.yaml
```

创建service.yaml

```
$ kubectl expose deployment web --port=80 --target-port=80 --type=NodePort --name=web --dry-run=client -o yaml > service.yaml 
```

创建ingress.yaml

```
$ vim ingress.yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: {{ include "demo.fullname" . }}
  labels:
    {{- include "demo.labels" . | nindent 4 }}
spec:
  rules:
  - host: {{ .Values.ingress.host }}
    http:
      paths:
      - path: /
        backend:
          serviceName: {{ include "demo.fullname" . }}
          servicePort: {{ .Values.service.port }}
{{- end }}
```

修改NOTES.txt

```
$ vim templates/NOTES.txt 
---
访问地址：
{{- if .Values.ingress.enabled }}
  http://{{ .Values.ingress.host }}
{{- end }}
{{- if contains "NodePort" .Values.service.type }}
  export NODE_PORT=$(kubectl get --namespace {{ .Release.Namespace }} -o jsonpath="{.spec.ports[0].nodePort}" services {{ include "demo.fullname" . }})
  export NODE_IP=$(kubectl get nodes --namespace {{ .Release.Namespace }} -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://${NODE_IP}:${NODE_PORT}

{{- else if contains "ClusterIP" .Values.service.type }}
  export POD_NAME=$(kubectl get pods --namespace {{ .Release.Namespace }} -l "app.kubernetes.io/name={{ include "demo.fullname" . }},app.kubernetes.io/instance={{ .Release.Name }}" -o jsonpath="{.items[0]
.metadata.name}")
{{- end }}
```

