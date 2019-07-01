layout: post
title: kubernetes安装dashboard
author: Pyker
categories: kubernetes
tags:
  - kubernetes
date: 2019-06-16 21:34:00
---

# kubernetes dashboard介绍
`Dashboard`是一个基于web的Kubernetes用户界面。您可以使用Dashboard将容器化应用程序部署到Kubernetes集群，对容器化应用程序进行故障诊断，并管理集群资源。您还可以使用Dashboard来获得运行在集群上的应用程序的概述以及创建或修改单个Kubernetes资源(如Deployment、Jobs、DaemonSets等)。例如，您可以扩展部署、启动滚动更新、重启pod或使用deploy向导部署新应用程序。

# 部署Dashboard
默认情况下，kubernetes没有安装它，因此你需要运行以下命来安装:
> 注意：我们使用的是环境是[上一篇](https://www.ipyker.com/2019/06/15/kubeadm-multi-cluster.html)文章的环境。

```bash
# 下载dashboard的yaml文件
$ wget https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```
正常情况你已经可以直接运行`kubectl apply -f kubernetes-dashboard.yaml`命令来运行dashboard了。如果你的node节点能科学上网下载dashboard镜像的话。但是为了我们正常使用的需要，我们还需要对该`kubernetes-dashboard.yaml`文件进行如下配置：
* 暴露端口
* 配置证书（否则火狐浏览器可以正常访问，其他浏览器将显示`NET::ERR_CERT_INVALID`，提示证书未找到）

## 暴露端口
默认配置文件下，dashboard的service端口使用的cluster ip，并没有把端口暴露出去，所以只能在kubernetes集群内部访问，这明显不是我们想要的结果。因此我们修改配置文件，增加`type: NodePort`
```bash
# ------------------- Dashboard Service ------------------- #

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  ports:
    - port: 443
      targetPort: 8443
  type: NodePort     # 我们在此处加入type: NodePort 
  selector:
    k8s-app: kubernetes-dashboard

```
> 当然如果你不使用NodePort暴露端口的话，我们还可以使用ingress-nginx进行暴露。目前我们这里使用NodePort方式。

## 配置证书
当我们直接运行`kubectl apply -f kubernetes-dashboard.yaml`后，dashboard的pod就算已经是running状态了，我们也只能使用火狐浏览器访问，其他浏览器均访问不了，显示`NET::ERR_CERT_INVALID`证书未找到的错误，因此我们需要手动来生成有效的dashboard证书。

有些人可能会说kubernetes-dashboard.yml文件中不是有这个`--auto-generate-certificates`配置吗？ 它不就是自动生成dashboard证书的吗？但是他确实是没给我们生成，如下：
```bash
kubectl describe secret kubernetes-dashboard-certs  -n kube-system
Name:         kubernetes-dashboard-certs
Namespace:    kube-system
Labels:       k8s-app=kubernetes-dashboard
Annotations:  
Type:         Opaque

Data
====    # 这里根本没有证书
```
可以看到这个secret为空，并没有证书，因此我们需要手动生成，并且卷的方式挂载进去。可以使用hostPath方式，nfs方式等等都行，我们这里使用hostPath方式。 为此我们开始生成证书。

### 创建dashboard证书
```bash
$ mkdir -pv /etc/kubernetes/pki/dashboard && cd /etc/kubernetes/pki/dashboard
$ (umask 077;openssl genrsa -out dashboard.key 2048)  # 创建dashboard私钥
$ openssl req -new -key dashboard.key -out dashboard.csr -subj "/O=ipyker/CN=dashboard"   # 创建证书请求
$ openssl x509 -req -in dashboard.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out dashboard.crt -days 3650 # 使用kubernetes的ca证书对dashboard证书请求签署
$ ls /etc/kubernetes/pki/dashboard
dashboard.crt  dashboard.csr  dashboard.key
```
此时我们已经生成了dashboard的证书和密钥了，由于我们采用hostPath方式挂载证书，因此我们需要将该证书目录复制到所有node节点上。来避免当dashboard的pod重建到其他node上找不到证书的情况。
```bash
$ scp -r /etc/kubernetes/pki/dashboard/ node1:/etc/kubernetes/pki/
$ scp -r /etc/kubernetes/pki/dashboard/ node2:/etc/kubernetes/pki/
$ scp -r /etc/kubernetes/pki/dashboard/ node3:/etc/kubernetes/pki/
```
我们以hostPath方式挂载卷配置如下：
```
volumes:
- name: kubernetes-dashboard-certs
  hostPath:
    path: /etc/kubernetes/pki/dashboard
    type: Directory
```
看了配置文件的你一定发现我们挂载卷的名称和sercet的名称怎么重复了？都是`kubernetes-dashboard-certs`。是的，我们这里因为使用hostPath挂载，所以系统默认的使用sercet方式挂载的配置都无效，我们需要把它都删掉。修改后完整的配置如下：
```yaml
# Copyright 2017 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# ------------------- Dashboard Secret ------------------- #

#apiVersion: v1
#kind: Secret
#metadata:
#  labels:
#    k8s-app: kubernetes-dashboard
#  name: kubernetes-dashboard-certs
#  namespace: kube-system
#type: Opaque

#---
# Dashboard Secret语句块我们全部注释掉，不使用secret挂载。
# ------------------- Dashboard Service Account ------------------- #

apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system

---
# ------------------- Dashboard Role & Role Binding ------------------- #

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kubernetes-dashboard-minimal
  namespace: kube-system
rules:
  # Allow Dashboard to create 'kubernetes-dashboard-key-holder' secret.
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["create"]
  # Allow Dashboard to create 'kubernetes-dashboard-settings' config map.
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["create"]
  # Allow Dashboard to get, update and delete Dashboard exclusive secrets.
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs"]
  verbs: ["get", "update", "delete"]
  # Allow Dashboard to get and update 'kubernetes-dashboard-settings' config map.
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["kubernetes-dashboard-settings"]
  verbs: ["get", "update"]
  # Allow Dashboard to get metrics from heapster.
- apiGroups: [""]
  resources: ["services"]
  resourceNames: ["heapster"]
  verbs: ["proxy"]
- apiGroups: [""]
  resources: ["services/proxy"]
  resourceNames: ["heapster", "http:heapster:", "https:heapster:"]
  verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kubernetes-dashboard-minimal
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kubernetes-dashboard-minimal
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system

---
# ------------------- Dashboard Deployment ------------------- #

kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
    spec:
      containers:
      - name: kubernetes-dashboard
        image: k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1   # 该镜像地址需要科学上网哦
        ports:
        - containerPort: 8443
          protocol: TCP
        args:
          - --auto-generate-certificates
          # Uncomment the following line to manually specify Kubernetes API server Host
          # If not specified, Dashboard will attempt to auto discover the API server and connect
          # to it. Uncomment only if the default does not work.
          # - --apiserver-host=http://my-address:port
        volumeMounts:
        - name: kubernetes-dashboard-certs
          mountPath: /certs
          # Create on-disk volume to store exec logs
        - mountPath: /tmp
          name: tmp-volume
        livenessProbe:
          httpGet:
            scheme: HTTPS
            path: /
            port: 8443
          initialDelaySeconds: 30
          timeoutSeconds: 30
      volumes:
#      - name: kubernetes-dashboard-certs  # secret方式挂载全部注释或删除
#        secret:
#          secretName: kubernetes-dashboard-certs 
      - name: tmp-volume
        emptyDir: {}
      - name: kubernetes-dashboard-certs
        hostPath:			# 此处是我们采用的hostPath方式挂载证书
          path: /etc/kubernetes/pki/dashboard
          type: Directory
      serviceAccountName: kubernetes-dashboard
      # Comment the following tolerations if Dashboard must not be deployed on master
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule

---
# ------------------- Dashboard Service ------------------- #

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  ports:
    - port: 443
      targetPort: 8443
  type: NodePort
  selector:
    k8s-app: kubernetes-dashboard
```
此时我们再来运行以下命令来启动dashboard：
```bash
$ kubectl apply -f kubernetes-dashboard.yaml
```
如果你不能科学上网从`k8s.gcr.io`上下载dashboard镜像，请在所有node节点上运行以下命令手动下载。
```bash
$ docker pull registry.cn-shenzhen.aliyuncs.com/pyker/kubernetes-dashboard:v1.10.1
$ docker tag  registry.cn-shenzhen.aliyuncs.com/pyker/kubernetes-dashboard:v1.10.1 k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1
$ docker rmi registry.cn-shenzhen.aliyuncs.com/pyker/kubernetes-dashboard:v1.10.1
```

# 访问dashboard
当dashboard部署成功后，我们首先查看他的nodeport端口是多少。
```bash
$ kubectl get svc -n kube-system  kubernetes-dashboard
NAME                   TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
kubernetes-dashboard   NodePort   10.101.56.113   <none>        443:30320/TCP   62m
```
现在你可以在局域网任意一台主机上的任意浏览器以`https://nodeip:nodeport`的方式访问该dashboard了。虽然会提示网站不安全是因为你的证书是自签的，并不是第三方机构签署的。

## dashboard登陆认证
dashboard默认有两种登陆验证的方式，分别为：
* Kubeconfig文件认证
* token令牌认证

### token令牌认证
创建kubernetes-dashboard集群管理员角色
```bash
$ cat > dashboard-admin.yaml << EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-admin
  namespace: kube-system
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: dashboard-admin
subjects:
  - kind: ServiceAccount
    name: dashboard-admin
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
EOF
```
```bash
$ kubectl apply -f dashboard-admin.yaml
```
查看serviceaccount生成的secret文件
```bash
$ kubectl get secret -n kube-system | grep dashboard-admin
dashboard-admin-token-h84v9                      kubernetes.io/service-account-token   3      101m
```
```bash
$ kubectl describe secret dashboard-admin-token-h84v9 -n kube-system
Name:         dashboard-admin-token-h84v9
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: dashboard-admin
              kubernetes.io/service-account.uid: c03f5dec-9c05-11e9-aec0-000c29b438b3

Type:  kubernetes.io/service-account-token

Data
====
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4taDg0djkiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiYzAzZjVkZWMtOWMwNS0xMWU5LWFlYzAtMDAwYzI5YjQzOGIzIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1hZG1pbiJ9.JFxxIEltq8AiYdTTKxy-QfmR2l6lcyPNOuGlAGR8bXTjVfveCQFj8GEg2eyOJsrjwqgZRN_CwvSpoTjxD39Xv2nvTj-jvWONuLhfr8cHLnTpSgB1KC3oYc0k27w5S0jmZ0Xsimn2pIXAaqzvX3ndZCKW4FngXjRwq27N6pWvynCOuY_625OeDW0Nuonx2AMBXZuu3cqdkM3JSxCc1NDjlJWiTj2Na1-A6uKnG1BxjfGYG5cVTdt4BbBr1Xvom-ky8ts9fUODZozmRyDcJ8kiezx2KTGE4TVMDtotC4TrE0pfWwoo2UsCi0SFmvSA5Vvy9gFHTIi9PskmsYq5ejkn6Q
ca.crt:     1025 bytes
```
其中token就是我们以token令牌认证登陆的密钥信息，我们可以在dashboard输入该字符串进行登陆。

### Kubeconfig文件认证
Kubeconfig文件认证其实就是将token写入到类似`~/.kube/config`的文件中，以该文件进行登陆的。步骤如下：
* 获取base64加密的token

```bash
$ kubectl get secret dashboard-admin-token-h84v9 -n kube-system -o jsonpath={.data.token}
ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNklpSjkuZXlKcGMzTWlPaUpyZFdKbGNtNWxkR1Z6TDNObGNuWnBZMlZoWTJOdmRXNTBJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5dVlXMWxjM0JoWTJVaU9pSnJkV0psTFhONWMzUmxiU0lzSW10MVltVnlibVYwWlhNdWFXOHZjMlZ5ZG1salpXRmpZMjkxYm5RdmMyVmpjbVYwTG01aGJXVWlPaUprWVhOb1ltOWhjbVF0WVdSdGFXNHRkRzlyWlc0dGFEZzBkamtpTENKcmRXSmxjbTVsZEdWekxtbHZMM05sY25acFkyVmhZMk52ZFc1MEwzTmxjblpwWTJVdFlXTmpiM1Z1ZEM1dVlXMWxJam9pWkdGemFHSnZZWEprTFdGa2JXbHVJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5elpYSjJhV05sTFdGalkyOTFiblF1ZFdsa0lqb2lZekF6WmpWa1pXTXRPV013TlMweE1XVTVMV0ZsWXpBdE1EQXdZekk1WWpRek9HSXpJaXdpYzNWaUlqb2ljM2x6ZEdWdE9uTmxjblpwWTJWaFkyTnZkVzUwT210MVltVXRjM2x6ZEdWdE9tUmhjMmhpYjJGeVpDMWhaRzFwYmlKOS5KRnh4SUVsdHE4QWlZZFRUS3h5LVFmbVIybDZsY3lQTk91R2xBR1I4YlhUalZmdmVDUUZqOEdFZzJleU9Kc3Jqd3FnWlJOX0N3dlNwb1RqeEQzOVh2Mm52VGotanZXT051TGhmcjhjSExuVHBTZ0IxS0Mzb1ljMGsyN3c1UzBqbVowWHNpbW4ycElYQWFxenZYM25kWkNLVzRGbmdYalJ3cTI3TjZwV3Z5bkNPdVlfNjI1T2VEVzBOdW9ueDJBTUJYWnV1M2NxZGtNM0pTeENjMU5EamxKV2lUajJOYTEtQTZ1S25HMUJ4amZHWUc1Y1ZUZHQ0QmJCcjFYdm9tLWt5OHRzOWZVT0Rab3ptUnlEY0o4a2llengyS1RHRTRUVk1EdG90QzRUckUwcGZXd29vMlVzQ2kwU0ZtdlNBNVZ2eTlnRkhUSWk5UHNrbXNZcTVlamtuNlE=
```
* 解密获取到的加密tonken

```bash
$ echo ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNklpSjkuZXlKcGMzTWlPaUpyZFdKbGNtNWxkR1Z6TDNObGNuWnBZMlZoWTJOdmRXNTBJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5dVlXMMyVmpjbVYwTG01aGJXVWlPaUprWVhOb1ltOWhjbVF0WVdSdGFXNHRkRzlyWlc0dGFEZzBkamtpTENKcmRXSmxjbTVsZEdWekxtbHZMM05sY25acFkyVmhZMk52ZFc1MEwzTmxjblpwWTJVdFlXTmpiM1Z1ZEM1dVlXMWxJam9p1ZFdsa0lqb2lZekF6WmpWa1pXTXRPV013TlMweE1XVTVMV0ZsWXpBdE1EQXdZekk1WWpRek9HSXpJaXdpYzNWaUlqb2ljM2x6ZEdWdE9uTmxjblpwWTJWaFkyTnZkVzUwT210MVltVXRjM2x6ZEdWdE9tUmhjMmhpYjJGeVpDMdlNwb1RqeEQzOVh2Mm52VGotanZXT051TGhmcjhjSExuVHBTZ0IxS0Mzb1ljMGsyN3c1UzBqbVowWHNpbW4ycElYQWFxenZYM25kWkNLVzRGbmdYalJ3cTI3TjZwV3Z5bkNPdVlfNjI1T2VEVzBOdW9ueDJBTUJYWnV1M2NxZGTRUVk1EdG90QzRUckUwcGZXd29vMlVzQ2kwU0ZtdlNBNVZ2eTlnRkhUSWk5UHNrbXNZcTVlamtuNlE= | base64 -d
eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4taDg0djkiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiYzAzZjVkZWMtOWMwNS0xMWU5LWFlYzAtMDAwYzI5YjQzOGIzIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1hZG1pbiJ9.JFxxIEltq8AiYdTTKxy-QfmR2l6lcyPNOuGlAGR8bXTjVfveCQFj8GEg2eyOJsrjwqgZRN_CwvSpoTjxD39Xv2nvTj-jvWONuLhfr8cHLnTpSgB1KC3oYc0k27w5S0jmZ0Xsimn2pIXAaqzvX3ndZCKW4FngXjRwq27N6pWvynCOuY_625OeDW0Nuonx2AMBXZuu3cqdkM3JSxCc1NDjlJWiTj2Na1-A6uKnG1BxjfGYG5cVTdt4BbBr1Xvom-ky8ts9fUODZozmRyDcJ8kiezx2KTGE4TVMDtotC4TrE0pfWwoo2UsCi0SFmvSA5Vvy9gFHTIi9PskmsYq5ejkn6Q
```
* 配置dashboard-admin的集群信息

```bash
$ kubectl config set-cluster kubernetes --certificate-authority=/etc/kubernetes/pki/ca.crt --server="https://192.168.20.222:6443" --embed-certs=true --kubeconfig=/root/dashboard-admin.conf
```
* 配置用户token信息

```bash
$ kubectl config set-credentials dashboard-admin --token=eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4taDg0djkiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiYzAzZjVkZWMtOWMwNS0xMWU5LWFlYzAtMDAwYzI5YjQzOGIzIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1hZG1pbiJ9.JFxxIEltq8AiYdTTKxy-QfmR2l6lcyPNOuGlAGR8bXTjVfveCQFj8GEg2eyOJsrjwqgZRN_CwvSpoTjxD39Xv2nvTj-jvWONuLhfr8cHLnTpSgB1KC3oYc0k27w5S0jmZ0Xsimn2pIXAaqzvX3ndZCKW4FngXjRwq27N6pWvynCOuY_625OeDW0Nuonx2AMBXZuu3cqdkM3JSxCc1NDjlJWiTj2Na1-A6uKnG1BxjfGYG5cVTdt4BbBr1Xvom-ky8ts9fUODZozmRyDcJ8kiezx2KTGE4TVMDtotC4TrE0pfWwoo2UsCi0SFmvSA5Vvy9gFHTIi9PskmsYq5ejkn6Q --kubeconfig=/root/dashboard-admin.conf
```
* 配置上下文和当前上下文

```bash
$ kubectl config set-context dashboard-admin@kubernetes --cluster=kubernetes --user=dashboard-admin --kubeconfig=/root/dashboard-admin.conf
```
* 配置当前使用的上下文

```bash
$ kubectl config use-context dashboard-admin@kubernetes --kubeconfig=/root/dashboard-admin.conf
```
查看dashboard-admin.config配置
```bash
$ kubectl config view --kubeconfig=/root/dashboard-admin.conf 
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://192.168.20.222:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: dashboard-admin
  name: dashboard-admin@kubernetes
current-context: dashboard-admin@kubernetes
kind: Config
preferences: {}
users:
- name: dashboard-admin
  user:
    token: eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4taDg0djkiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiYzAzZjVkZWMtOWMwNS0xMWU5LWFlYzAtMDAwYzI5YjQzOGIzIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1hZG1pbiJ9.JFxxIEltq8AiYdTTKxy-QfmR2l6lcyPNOuGlAGR8bXTjVfveCQFj8GEg2eyOJsrjwqgZRN_CwvSpoTjxD39Xv2nvTj-jvWONuLhfr8cHLnTpSgB1KC3oYc0k27w5S0jmZ0Xsimn2pIXAaqzvX3ndZCKW4FngXjRwq27N6pWvynCOuY_625OeDW0Nuonx2AMBXZuu3cqdkM3JSxCc1NDjlJWiTj2Na1-A6uKnG1BxjfGYG5cVTdt4BbBr1Xvom-ky8ts9fUODZozmRyDcJ8kiezx2KTGE4TVMDtotC4TrE0pfWwoo2UsCi0SFmvSA5Vvy9gFHTIi9PskmsYq5ejkn6Q
```
确认没有错误后，我们将此文件copy出来到本机，然后访问dashboard页面，使用kubeconfig认证登陆即可。