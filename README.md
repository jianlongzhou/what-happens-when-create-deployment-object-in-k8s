# what-happens-when-create-deployment-object-in-k8s
## 前言
受到github仓库what-happens-when-k8s的影响，加上近期正在准备工作交界，闲暇之余，想自己也来按照自己的理解总结整理一下k8s相关的知识脉络，以前总会有突然无法想起某个知识点的时候，希望以这里为起点，之后养成多记录多总结的习惯，仅作为自己的笔记，相对前面提到的repo作为一个补充。

k8s作为一个容器编排系统，有很多组件组成并谐调工作，我想，如果能利用一个问题点把各个组件的工作流程衔接起来，会更加有助于我们理解k8s。
以下描述的是用户通过以下命令创建一个deployment对象时所发生的事情。
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
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
      - name: nginx
        image: nginx:1.12.2
```
```bash
kubectl create -f nginx-deployment.yaml
```

## 步骤

### kubectl
- 当使用kubectl命令通过yaml文件创建一个对象时，kubectl首先做客户端验证，检查过滤一些很明显统一的错误（比如镜像的格式错误），而不是无脑的发给apiserver增加apiserver端的压力。
- kubectl开始构造http用于请求apiserver，由于请求时需要做证书单向验证，kubectl访问apiserver时，apiserver会返回自己的服务端证书，kubectl会利用ca证书来验证服务端证书是否合法有效，ca证书存在于~/。kube/config文件中，这个文件也是kubectl默认读取的文件。
- kubectl构造的http请求头的Authorization字段中也会包含身份信息的token，用于向apiserver表明自己的身份，以便apiserver做认证和鉴权。

### apiserver
- 认证 Authentication：apiserver需要认证请求的用户是否存在，用户的登录凭据是否认证通过。
- 鉴权 Authorization：对应的用户是否有请求对应的权限，比如这里的depoyment的创建请求
- 准入控制 Admission Controller：apiserver中内置了很多的准入控制项目，比如ResourceQuota、ServiceAccount等，
