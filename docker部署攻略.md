### 基础

使用docker可以快速的将代码和代码运行的环境进行整合，并通过jenkins进行快速分发，简化持续集成流程。

### 原理

使用docker部署简单的说就是通过写Dockerfile文件，将代码和环境一起构建一个镜像，并将镜像打标签上传到镜像托管，然后在服务器上拉指定的镜像运行。

### 前提条件
由于需要分发的环境有staging和production，所以很容易想到需要两个镜像来分别来保存，如我在yuanbenlian项目中分别使用yuanbenlian/staging和yuanbenlian/production两种命名方式，对应的镜像仓库也需要创建两份。

### 镜像仓库

首先需要在[阿里云镜像仓库](https://cr.console.aliyun.com/cn-shanghai/)创建命名空间，然后在指定的命名空间下创建镜像仓库

### 打包镜像流程

首先需要明确的是发布一个镜像必须要确定一个版本号，版本号可以方便用来日后的回滚，通常我们可以使用`npm version major | minor | patch` 命令来快速实现项目的版本升级,该命令同时会生成一条`git commit`记录，包含了本次的版本tag，所以需要将tag一起推送到远端，可以使用`git push --tag`命令。
确定好版本号之后使用`npm run build`进行项目的打包，等待打包完成之后就可以开始制作docker镜像
```
# 使用./docker/staging/Dockerfile配置文件构建镜像并命名为yuanbenlian/staging:[version]
docker build -f ./docker/staging/Dockerfile -t "yuanbenlian/staging:[version]" .
# 然后查看刚刚构建的镜像, 并找到镜像的id
docker images
# 重新打标签，接下来这几步可以在阿里云镜像仓库中看到
docker tag [imgId] "registry.cn-shanghai.aliyuncs.com/ybl/yuanbenlian-staging:[version]"
docker login --username=七印科技 -p [password] registry.cn-shanghai.aliyuncs.com
docker push "registry.cn-shanghai.aliyuncs.com/ybl/yuanbenlian-staging:[version]"
# 完成推送镜像到远程仓库
```

### 服务端部署

在服务端拉指定版本的docker镜像然后运行就可以了
```
docker pull registry.cn-shanghai.aliyuncs.com/ybl/yuanbenlian-staging:[version]
# 确保将之前运行的停掉
docker stop ybl-staging
# 运行容器，不加版本号默认latest, --rm的意思是stop之后自动删除该容器
docker run --rm -d -p 3000:3000 --name ybl-staging registry.cn-shanghai.aliyuncs.com/ybl/yuanbenlian-staging:[version]
```

### jenkins部署

在jenkins中分为三个任务，一个build任务，一个服务器部署任务，还有一个同时运行两个job的任务，在yuanbenlian/production项目中，release_yuanbenlian_build， release_yuanbenlian_ssr，release_yuanbenlian_pipe分别指的就是上面所说的三个任务，通常我们完成一个版本要发的时候直接运行release_yuanbenlian_pipe任务就行了，当线上出现问题需要回滚的时候，我们只需要运行release_yuanbenlian_ssr任务传入指定版本号运行对应的容器就行了