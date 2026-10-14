# 导入镜像tar包（正确命令）
docker load -i asw-kylin-v10-arm64.tar
# 查看导入后的镜像
docker images
# 给无标签镜像打标签（<none>时执行），有镜像名和标签时不要执行
docker tag 【IMAGE_ID】 asw-kylin-v10:arm64

# 查看容器详情，看镜像信息
docker inspect 容器名

# 直接输出容器对应的镜像ID
docker inspect --format '{{.Image}}' 容器名

# 停止并删除旧容器
# 标准输出（1 号流 stdout）：走管道 |，可以传给后面命令；
# 错误输出（2 号流 stderr）：不走管道，直接打印到控制台，绕过管道。
# 2>&1：把2 号错误流重定向到 1 号标准流，想让错误信息也能被后面命令处理（这里就是丢弃），就需要2>&1合并流，让错误信息进入管道
# linux下
#!/bin/bash
# 停止并删除旧容器（防止重名报错）
docker stop asw-kylin-v10 >/dev/null 2>&1
docker rm asw-kylin-v10 >/dev/null 2>&1
# powershell 下
# 停止并删除旧容器
docker stop asw-kylin-v10 2>&1 | Out-Null
docker rm asw-kylin-v10 2>&1 | Out-Null
# 新建容器
docker run -d `
  --name asw-kylin-v10 `
  --restart unless-stopped `
  --platform linux/arm64 `
  -p 8787:8787 `
  -v asw-kylin-v10-data:/var/lib/asw `
  asw-kylin-v10:arm64
# 后台运行容器（守护进程），不占用终端窗口；不输出容器内控制台打印。如果程序崩溃退出，容器会停止,，可重启。
  -d / --detach
# 指定容器名称，后续docker stop/exec/logs直接用名字，不用记容器 ID。同一个名字不能重复创建容器。
--name 容器名
# 容器重启策略 主机 / Docker 重启 → 自动启动容器 ,仅手动执行docker stop停止后，才不会自动拉起
--restart unless-stopped
# 指定镜像架构。Windows amd64 主机跑 arm64 麒麟镜像必须加，启用 QEMU 模拟，防止exec format error。
--platform linux/arm64
# 端口映射 -p 8787:8787,宿主机 Windows 的 8787 端口，流量转发到容器内部 8787 端口，外部可访问反潜服务。,多端口：-p 8787:8787 -p 8080:8080
-p 宿主机端口:容器端口
# 	数据挂载 -v asw-kylin-v10-data:/var/lib/asw 命名卷，容器删除后，卷里的业务数据仍然保留，持久化存储。
-v 卷名:容器内路径
# 实例化当前容器对应的镜像 ，可用镜像名称：标签，也可用镜像ID,最好用ID,因为标签只是别名，镜像真正唯一标识是IMAGE ID
asw-kylin-v10:arm64

# 镜像和容器删除时，必须删除容器后再删除镜像，执行 docker rmi 删除镜像时会报错，提示镜像被容器占用，如果加-f强制执行的话，强制 rmi 只是把镜像标签删掉，容器实例还在，容器依然可以启动。只是镜像变成悬空状态（<none>），不推荐这么操作，容易造成环境混乱
# 删除镜像（通过镜像名称删除，有标签时候）
docker rmi asw-kylin-v10:arm64  
# 通过镜像 ID 删除（当镜像为<none>无标签时使用）
docker rmi 镜像ID

# 停止容器
docker stop asw-kylin-v10
# 删除容器，如果rm 后加-f，表示强制删除
docker rm asw-kylin-v10
# 删除数据卷（不需要保留业务数据才执行）
# 配置了挂载卷的情况 删除容器 docker rm：命名卷 asw-kylin-v10-data 不会被删除，业务数据保留；新容器挂载同一个卷可以继续读取数据
docker volume rm asw-kylin-v10-data
#4 删除镜像
docker rmi asw-kylin-v10:arm64

# 环境清理流程
#1. 停止容器
docker stop asw-kylin-v10
#2. 删除容器
docker rm asw-kylin-v10
#3. 删除业务数据卷（不需要保留数据才执行）
docker volume rm asw-kylin-v10-data
#4. 删除镜像
docker rmi asw-kylin-v10:arm64


# docker ps -a：列出全部容器，包含正在运行（Up）以及已经停止退出（Exited）的容器
# |：管道符，把前一条命令输出的文本传给后面
# sls：Select-String 的简写，PowerShell 的文本过滤工具，相当于 Linux 的 grep
# asw-kylin-v10：过滤关键词，只输出包含该字符串的行

# 快速筛选名称 / ID 包含 asw-kylin-v10 的容器，windows上用sls，linux上用grep，可以通过输出的容器ID或者容器名对容器进行停止、启动、删除
docker ps -a | sls asw-kylin-v10

# 使用容器名查看日志，可以通过输出的容器ID或者容器
docker logs 容器名 或者docker logs 容器ID



# 打包镜像流程
# 1.先新建文件，名字就叫 Dockerfile（无后缀）
# dockerfile里面包含
# 指定基础镜像（这里用银河麒麟V10 arm64）
FROM kylin:v10-arm64
# 设置工作目录
WORKDIR /opt/asw_app
# 拷贝本地的服务端程序、依赖包到镜像内
COPY ./server_main /opt/asw_app/
COPY ./lib/*.so /usr/lib/
# 暴露服务端口（仅文档标记，不会真正开放端口）
EXPOSE 8787
# 设置容器启动命令（容器启动时运行的主程序）
CMD ["/opt/asw_app/server_main"]


# 2.构建镜像命令
# 末尾的 . 代表Dockerfile在当前目录
# 镜像名:tag 就是 asw-kylin-v10:arm64
docker build -t asw-kylin-v10:arm64 .

# 如果在 Windows amd64 机器构建 arm64 镜像，需要开启 buildx 跨架构构建
docker buildx build --platform linux/arm64 -t asw-kylin-v10:arm64 . --load
# 查看镜像
docker images
# 打包导出 tar 包
docker save -o asw-kylin-v10-arm64.tar asw-kylin-v10:arm64


# 注意跨架构导出镜像会有问题，但可以通过export导出容器，在通过import 容器变成镜像，再创建新容器   
有网电脑：
  Docker 容器（里面是 ARM64 Debian 文件系统）
        ↓ docker export
  debian-bookworm-arm64-rootfs.tar （纯文件系统，没有 Docker 的壳）
        ↓ U盘拷贝
离线电脑：
  debian-bookworm-arm64-rootfs.tar
        ↓ docker import
  Docker 镜像（重新包上 Docker 的壳）
        ↓ docker run
  新的 Docker 容器

# 非跨架构
有网机器（能上外网，装了 Docker）
  1. docker pull 拉镜像
  2. docker save 导出成 .tar 文件
          ↓
    U盘/内网拷文件
          ↓
离线机器（不能上网，装了 Docker）
  3. docker load 导入镜像
  4. 直接用，和在线没区别

# 跨架构流程（有网->离线）
# 1. 有网机器拉 ARM64 Debian 镜像，**编译构建阶段**`debian:bookworm`（完整版），**最终运行时镜像**`debian:bookworm-slim`（精简版）
docker pull --platform linux/arm64 debian:bookworm  
docker pull --platform linux/arm64 debian:bookworm-slim  
# 2.查看镜像
  docker images
# 3.创建临时容器
docker create --name temp-export-debian debian:bookworm
# 4.导出容器到镜像
docker export <容器ID> -o E:\kylin-v10\image-build\debian-bookworm-arm64.tar
# 5.**验证产物**（可选但建议），看条目数（debian bookworm 约 6600+）和是否含 `bin`、`etc` 等目录，确认文件系统完整
tar -tf E:\kylin-v10-aarch64\image-build\debian-bookworm-arm64.tar | Measure-Object -Line
# 6.清理临时容器
docker rm <容器ID>
# 7.** 目标机器还原**（把容器变成可用镜像）
docker import debian-bookworm-arm64.tar debian:bookworm-arm64





# 非跨架构流程
# 1.拉 ARM64 Debian 镜像
docker pull --platform linux/arm64 debian:bookworm
# 2.查看镜像
  docker images
# 3.导出成 tar 文件（这就是普通文件，能拷走）
docker save 镜像名 -o debian-bookworm-arm64.tar
# 4.导入镜像
docker load -i debian-bookworm-arm64.tar 
# 5.查看镜像
  docker images



debian:bookworm（空基础镜像）
   ↓ 容器内 apt 装：build-essential + cmake + openssl（编译工具链）
   ↓ 套件自带：Node 离线依赖 + CMake 压缩包（不用装，解压即用）
   ↓ 编译 C++ + 构建前端 + 打包 release
   ↓ 产物拷给精简镜像 bookworm-slim → 最终计算服务镜像


# **① 启动容器并进入 bash**（容器已建好，但退出了）
docker start asw-builder
docker exec -it asw-builder bash



# docker-desktops构建服务镜像流程
# 1.**启动编译容器**（挂载整个构建套件）
docker run -it --platform linux/arm64 --name asw-builder `
  -v E:\workspace\windows-source-build-kit:/build `
  debian:bookworm bash
# 2.编译和打包
bash scripts/kylin/build-cpp.sh
node node_modules/vue-tsc/bin/vue-tsc.js -b
node node_modules/vite/bin/vite.js build
node scripts/kylin/package-server.mjs

# build-cpp.sh 编译 C++ 算法
做什么	用 CMake 编译反潜计算的核心算法程序
内部动作	① 检查系统（麒麟 V10 + aarch64）→ ② cmake -S cpp -B build/... 生成构建配置 → ③ cmake --build 调 g++ 编译全部 C++ 源码 → ④ 把结果 analysis 安装到 build/bin/linux/
产物	build/bin/linux/analysis（可执行算法程序，ARM64 二进制）
# `vue-tsc.js -b` —— 前端类型检查
做什么	用 vue-tsc（TypeScript 检查器）检查 Vue 前端代码的类型是否正确
产物	无 ——通过即好，有类型错误会直接报错中断
类比	相当于 "编译前的体检"，防止把有 bug 的前端构建出去
# `vite.js build` —— 构建前端
做什么	用 Vite 把 Vue 源码打包成浏览器能直接运行的静态页面
内部动作	编译 .vue/.ts → 压缩合并 JS/CSS/ 图片 → 输出到 dist/
产物	dist/（前端静态资源，web 页面部分）
# `package-server.mjs` —— 汇总打包服务端
做什么	把前面所有产物收拢成一个完整可部署的服务端目录
收拢什么	前端 dist/ → web/；算法程序 analysis → bin/；Node 运行时 → runtime/；服务代码 services/、shared/；依赖 h5wasm、sql.js；NC 数据；启动脚本 start-server.sh
产物	release/server-kylin-v10-aarch64/（最终交付目录，拷走即用）


# 容器装好环境后一定要执行commit成新镜像，不然重启环境就没了
docker commit asw-builder debian:bookworm-dev
