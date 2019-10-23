# Docker

## docker常用命令

* `docker image ls` 列出已下载镜像
* `docker ps` 列出当前正在运行的docker镜像
* `docker ps -a` 列出之前运行过的docker镜像



## 删除docker镜像

* 用命令`docker ps -a`列出之前运行过的镜像，找到要删除的镜像的`CONTAINER ID`,执行命令先删除`CONTAINER ID`。删除命令：`docker rm <CONTRANER ID>`
* 用命令`docker images`列出所有镜像，找到要删除的镜像`IMAGE ID`，执行删除命令：`docker rmi <IMAGE ID>`



## docker中运行jar包

* `Dockerfile`文件：

```tex
FROM java:8
MAINTAINER zhanglin

ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

ADD hq-demo-0.0.1-SNAPSHOT.jar  app.jar

EXPOSE 9099

CMD ["java","-jar","app.jar"]
```

* 可执行文件，如`hq-demo.sh`

```shell
#!/bin/bash

cd /home/httech/app/hq-demo/

CONTAINER=$(docker ps -a|awk '/hq-demo-/ {print $1}')
if [[ -n "$CONTAINER" ]]; then
   echo "Removing hq-demo Container..."
   docker rm -fv $CONTAINER
   # Wait till the existing containers are removed
   sleep 5
fi

IMAGE=$(docker images -a|awk '/hq-demo/ {print $1}')
if [[ -n "$IMAGE" ]]; then
   echo "Removing hq-demo Image..."
   docker rmi $IMAGE
   sleep 5
fi

echo "Creating hq-demo Image..."
docker build -t hq-demo .
sleep 3


echo "Running hq-demo Container..."



docker run --name hq-demo -p 9099:9099 -d -v /home/httech/app/hq-demo/logs:/logs \
hq-demo
```

