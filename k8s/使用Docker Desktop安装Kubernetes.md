# 安装Kubernetes

- 安装docker desktop

- 打开docker desktop的settings

- 选择Kubernetes，将Enable Kubernetes和Show system containers(advanced)都勾选

- 点击Apply & Restart按钮进行安装重启，需要等待下载完成

# 安装Kubernetes Dashboard

1. 使用命令安装：`kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml`

2. 安装完后使用命令查看：`kubectl get pods -n kubernetes-dashboard -w`

3. 创建服务账号：`dashboard-admin`，所属命名空间：`kubernetes-dashboard`，执行以下命令：`kubectl create serviceaccount dashboard-admin -n kubernetes-dashboard`

4. 给服务账号`dashboard-admin`绑定`admin`角色，执行以下命令：`kubectl create clusterrolebinding dashboard-cluster-admin --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:dashboard-admin`

5. 创建Token并指定过期时间：`kubectl -n kubernetes-dashboard create token dashboard-admin --duration 3153600000s`，记录下Token：`eyJhbGciOiJSUzI1NiIsImtpZCI6IkgtcXk3Vm9VdlpxSEhUVG5SMkhZUkI1X3E2dmJsYjdZWmZpbWVPOGlzdlUifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjo0ODE3NDQ3MTU5LCJpYXQiOjE2NjM4NDcxNTksImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJkYXNoYm9hcmQtYWRtaW4iLCJ1aWQiOiIzZjc1YWM3MS01M2NkLTQ2MTgtYTkyZC1lYzBiZTgxYTkzZmQifX0sIm5iZiI6MTY2Mzg0NzE1OSwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmVybmV0ZXMtZGFzaGJvYXJkOmRhc2hib2FyZC1hZG1pbiJ9.WmsJ3nUERnpb0ynWDKWzFOda5YPfihfUe0MZU6gZS69UaxShw9_tsM1eNgn826lRyq_HYs8ZQoDh8GrYVgDqvXaDIUKchA-FM_2ZP3VBiYuXrcl5HBaR8O3AmhsA_VlcW3J-HH0MpATKTZeAE5r-Xe6tT2QDttWsfGNn3ubEE0smO-WDAVJl1BhSNhGTU_HpjoQJEdSP9ZfFg2gIUPdPfFYylnvxMr3VacMCVUGAVfhSWW5SVtTGmcKFaS4xndiTMPCAj_-T4Trw3GZMHMCZ-mucDvA_my7HPLy0aNojoA5frmfm3u6yU53lTcDWsy7Wui55ZF563oAqbLxy6bFe3A`

6. 启动Dashboard：`kubectl proxy`

7. 访问Dashboard：`[Kubernetes Dashboard](http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/)`，使用上面的Token即可进入到Dashboard

# 安装ingress-nginx

1. 使用命令安装：`kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.7.0/deploy/static/provider/cloud/deploy.yaml`
2. 使用命令查看安装进度并等待完成：`kubectl get pods --namespace=ingress-nginx --watch`

如果安装不成功，可以使用如下方式：

1. 下载配置文件`https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.7.0/deploy/static/provider/cloud/deploy.yaml`
2. 编辑配置文件，将配置文件中的`registry.k8s.io`替换为国内镜像，比如：`registry.lank8s.cn`
3. 使用命令安装：`kubectl apply -f deploy.yaml`
4. 使用命令查看安装进度并等待完成：`kubectl get pods --namespace=ingress-nginx --watch`

安装成功后继续进行验证：

1. 创建一个测试用web应用部署：`kubectl create deployment demo --image=httpd --port=80`
2. 暴露成服务：`kubectl expose deployment demo`
3. 创建ingress资源：`kubectl create ingress demo-localhost --class=nginx --rule="demo.localdev.me/*=demo:80"`
4. 转发本地端口到ingress控制器：`kubectl port-forward --namespace=ingress-nginx service/ingress-nginx-controller 8080:80`
5. 访问：`http://demo.localdev.me:8080`，页面返回`It works!`，表示成功

验证成功后删除临时的验证资源：

1. 删除ingress资源：`kubectl delete ingress demo-localhost`
2. 删除服务：`kubectl delete service demo`
3. 删除应用部署：`kubectl delete deployment demo`
