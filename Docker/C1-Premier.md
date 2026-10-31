> Docker 将应用所需的用户空间环境打包，并隔离运行
> 使环境配置可以随项目一起版本管理、重建和迁移
# 1. Docker Engine installation
- install
- sudo docker permission
- sudo systemctl enable docker
- docker pull iso cHost (172.17.0.1)onfig
- docker volume create xx
- portainer (docker manager)
- local::port
  
# 2. Docker Engine Architecture
> image, container, volume, network 

# 3. Boostup a Docker
## 3.1 image
- build image with Dockerfile
```
- Dockerfile
- docker build -t image1 . 根据目录下的Dockerfile构建一个image1
- docker create --name container1 image1 根据image1创建一个container1
```
- pull image from third-party
```
- docker pull image
- (hub.docker.com/-/debian) 
```
## 3.2 container
```
- docker start container1 启动container1的主程序
- docker ps -a /  docker inspect  container1 查看容器信息
- docker exec -it container1 cmd 宿主机发起请求，指定container1额外启动一个cmd进程
- docker stop container1 /  docker rm container1 停止，移除container1
```
- `docker run -it --name container1 image1` 创建一个新容器，启动它的主程序
- docker默认执行主程序-ctrl+d可能会结束主程序 (穿透执行解决)

# 4. Docker Data Persistence
> container's removal doesn't affect data
- Bind mount
- Docker Volume

# 5. Network
> docker内部创建虚拟网络，通过端口映射


# 5. Dev Container & Handoff Container


