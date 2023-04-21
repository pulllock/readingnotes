# 创建网络

`docker network create local_cicd`

# 安装Kubernetes

- 安装docker desktop

- 打开docker desktop的settings

- 选择Kubernetes，将Enable Kubernetes和Show system containers(advanced)都勾选

- 点击Apply & Restart按钮进行安装重启，需要等待下载完成

# 安装Kubernetes Dashboard

1. 使用命令安装：`kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml`

2. 创建服务账号：`dashboard-admin`，所属命名空间：`kubernetes-dashboard`，执行以下命令：`kubectl create serviceaccount dashboard-admin -n kubernetes-dashboard`

3. 给服务账号`dashboard-admin`绑定`admin`角色，执行以下命令：`kubectl create clusterrolebinding dashboard-cluster-admin --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:dashboard-admin`

4. 创建Token并指定过期时间：`kubectl -n kubernetes-dashboard create token dashboard-admin --duration 3153600000s`，记录下Token：`eyJhbGciOiJSUzI1NiIsImtpZCI6IkgtcXk3Vm9VdlpxSEhUVG5SMkhZUkI1X3E2dmJsYjdZWmZpbWVPOGlzdlUifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjo0ODE3NDQ3MTU5LCJpYXQiOjE2NjM4NDcxNTksImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJkYXNoYm9hcmQtYWRtaW4iLCJ1aWQiOiIzZjc1YWM3MS01M2NkLTQ2MTgtYTkyZC1lYzBiZTgxYTkzZmQifX0sIm5iZiI6MTY2Mzg0NzE1OSwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmVybmV0ZXMtZGFzaGJvYXJkOmRhc2hib2FyZC1hZG1pbiJ9.WmsJ3nUERnpb0ynWDKWzFOda5YPfihfUe0MZU6gZS69UaxShw9_tsM1eNgn826lRyq_HYs8ZQoDh8GrYVgDqvXaDIUKchA-FM_2ZP3VBiYuXrcl5HBaR8O3AmhsA_VlcW3J-HH0MpATKTZeAE5r-Xe6tT2QDttWsfGNn3ubEE0smO-WDAVJl1BhSNhGTU_HpjoQJEdSP9ZfFg2gIUPdPfFYylnvxMr3VacMCVUGAVfhSWW5SVtTGmcKFaS4xndiTMPCAj_-T4Trw3GZMHMCZ-mucDvA_my7HPLy0aNojoA5frmfm3u6yU53lTcDWsy7Wui55ZF563oAqbLxy6bFe3A`

5. 启动Dashboard：`kubectl proxy`

6. 访问Dashboard：`[Kubernetes Dashboard](http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/)`，使用上面的Token即可进入到Dashboard

# 安装docker-registry

- 安装并启动：`docker run -d --net local_cicd -p 5000:5000 --name registry registry:latest`

- 访问验证：`http://localhost:5000/v2/_catalog`

- 将已有的`openjdk:8-jre-alpine`镜像（该镜像是从docker hub上拉下来的：`docker pull openjdk:8-jre-alpine`）推送到私有仓库:
  
  - 标记镜像：`docker tag openjdk:8-jre-alpine localhost:5000/openjdk:8-jre-alpine`
  
  - 上传镜像：`docker push localhost:5000/openjdk:8-jre-alpine`
  
  - 访问验证：`http://localhost:5000/v2/_catalog`

- 使用：`docker push localhost:5000/image_name:latest`、`docker pull localhost:5000/image_name:latest`

# 安装gitlab

- 安装并启动：`docker run -d --net local_cicd -p 9443:443 -p 9080:80 -p 9022:22 --restart always --name gitlab --shm-size 256m gitlab/gitlab-ce:15.4.0-ce.0`

- 获取初始密码，执行命令：`docker exec -it gitlab grep 'Password:' /etc/gitlab/initial_root_password`

- 访问：`http://localhost:9080`，使用账号`root`和初始密码登录，登录后修改密码为：`12345678`

# 安装gitlab runner

- 安装：`docker run -d --net local_cicd --restart always --name gitlab-runner gitlab/gitlab-runner:latest`

- 开始注册：`docker exec -it gitlab-runner gitlab-runner register --non-interactive --executor "docker" --docker-image alpine:latest --url "http://gitlab" --registration-token "这里替换成GitLab Runner的Token" --description "first-runner" --tag-list "test runner, first runner"`

- Gitlab Runner的Token，Token可在GitLab的`Admin --> Overview --> Runners --> Register an instance runner --> Registration token`中找到

- 回到GitLab的Runners中就可以看到刚才设置的gitlab runner
