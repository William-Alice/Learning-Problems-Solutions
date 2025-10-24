# mysql 无法启动故障分析

## 出现问题

虚拟机未关机，主机意外更新重启

## 异常表现

[root@localhost ~]# docker restart mysql92
mysql92
[root@localhost ~]# docker ps
CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES
93ad9b064c11 nacos/nacos-server:v2.1.0-slim "bin/docker-startup.…" 2 weeks ago Up 2 hours 0.0.0.0:8848->8848/tcp, :::8848->8848/tcp, 0.0.0.0:9848-9849->9848-9849/tcp, :::9848-9849->9848-9849/tcp nacos
48b9d7fdfb1b mysql:9.2 "docker-entrypoint.s…" 2 weeks ago Restarting (1) Less than a second ago

## 日志内容

[root@localhost ~]# docker logs -f mysql92
2025-10-24 10:31:03+08:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 9.2.0-1.el9 started.
2025-10-24 10:31:03+08:00 [ERROR] [Entrypoint]: mysqld failed while attempting to check config
command was: mysqld --verbose --help --log-bin-index=/tmp/tmp.ENPfZM8OQJ
mysqld: error while loading shared libraries: /lib64/libaio.so.1: invalid ELF header
2025-10-24 10:31:04+08:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 9.2.0-1.el9 started.
2025-10-24 10:31:04+08:00 [ERROR] [Entrypoint]: mysqld failed while attempting to check config
command was: mysqld --verbose --help --log-bin-index=/tmp/tmp.cJ5DZ5aKCP
mysqld: error while loading shared libraries: /lib64/libaio.so.1: invalid ELF header
2025-10-24 10:31:04+08:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 9.2.0-1.el9 started.
2025-10-24 10:31:04+08:00 [ERROR] [Entrypoint]: mysqld failed while attempting to check config
command was: mysqld --verbose --help --log-bin-index=/tmp/tmp.UihmwmLZFR
mysqld: error while loading shared libraries: /lib64/libaio.so.1: invalid ELF header
2025-10-24 10:31:05+08:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 9.2.0-1.el9 started.
2025-10-24 10:31:05+08:00 [ERROR] [Entrypoint]: mysqld failed while attempting to check config
command was: mysqld --verbose --help --log-bin-index=/tmp/tmp.0LDuYgNmWt
mysqld: error while loading shared libraries: /lib64/libaio.so.1: invalid ELF header
2025-10-24 10:31:06+08:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 9.2.0-1.el9 started.
2025-10-24 10:31:06+08:00 [ERROR] [Entrypoint]: mysqld failed while attempting to check config
command was: mysqld --verbose --help --log-bin-index=/tmp/tmp.1bAAz6p2T5
mysqld: error while loading shared libraries: /lib64/libaio.so.1: invalid ELF header
2025-10-24 10:31:08+08:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 9.2.0-1.el9 started.
2025-10-24 10:31:08+08:00 [ERROR] [Entrypoint]: mysqld failed while attempting to check config
command was: mysqld --verbose --help --log-bin-index=/tmp/tmp.eE6rl5bbIx
mysqld: error while loading shared libraries: /lib64/libaio.so.1: invalid ELF header
2025-10-24 10:31:11+08:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 9.2.0-1.el9 started.
2025-10-24 10:31:11+08:00 [ERROR] [Entrypoint]: mysqld failed while attempting to check config
command was: mysqld --verbose --help --log-bin-index=/tmp/tmp.MmEuJUA3aR
mysqld: error while loading shared libraries: /lib64/libaio.so.1: invalid ELF header
2025-10-24 10:31:18+08:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 9.2.0-1.el9 started.
2025-10-24 10:31:18+08:00 [ERROR] [Entrypoint]: mysqld failed while attempting to check config
command was: mysqld --verbose --help --log-bin-index=/tmp/tmp.XtyRIqcDQV
mysqld: error while loading shared libraries: /lib64/libaio.so.1: invalid ELF header
2025-10-24 10:31:31+08:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 9.2.0-1.el9 started.
2025-10-24 10:31:31+08:00 [ERROR] [Entrypoint]: mysqld failed while attempting to check config
command was: mysqld --verbose --help --log-bin-index=/tmp/tmp.rajxJ9X2Dx
mysqld: error while loading shared libraries: /lib64/libaio.so.1: invalid ELF header
2025-10-24 10:31:57+08:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 9.2.0-1.el9 started.
2025-10-24 10:31:57+08:00 [ERROR] [Entrypoint]: mysqld failed while attempting to check config
command was: mysqld --verbose --help --log-bin-index=/tmp/tmp.JxnFfDUcdG
mysqld: error while loading shared libraries: /lib64/libaio.so.1: invalid ELF header

## 异常分析

`error while loading shared libraries: /lib64/libaio.so.1: invalid ELF header`

MySQL 启动时需要加载 /lib64/libaio.so.1 这个 “共享库文件”，但系统检测到这个文件的 “身份信息（ELF 头）无效”，无法识别它是一个合法的依赖库

### 可能原因

1. 库文件本身损坏 / 不完整：之前你删除过容器内的 libaio.so.1，虽然尝试重新安装，但因为容器系统类型不确定（不知道用 yum/dnf/apt/apk），导致安装失败，最终容器内的 libaio.so.1 还是个 “坏文件”（ELF 头无效）；
2. 库文件版本 / 架构不兼容：比如你之前误把宿主机（Oracle Linux 9）的 libaio.so.1 复制到容器内，但容器镜像的系统版本（比如 Alpine/Debian）和宿主机不匹配，导致库文件的 “身份信息”（ELF 头）和容器系统不兼容，被判定为 “无效”；
3. 库文件路径被覆盖：比如挂载宿主机目录时，不小心用宿主机的 libaio.so.1 覆盖了容器内的同名文件，而宿主机的库和容器系统不兼容。

### 问题解决

1. 删除旧的容器重新安装 mysql

```
# 删除旧的容器
docker rm -f mysql92

# 重新安装
docker run -d \
  --name mysql92 \
  -p 3306:3306 \
  -e TZ=Asia/Shanghai \
  -e MYSQL_ROOT_PASSWORD=root \
  -v ./mysql/data:/var/lib/mysql \
  -v ./mysql/init:/docker-entrypoint-initdb.d \
  -v ./mysql/conf:/etc/mysql/conf.d \
  --network hm-net \
  --restart=always \
  mysql:9.2
```

#### 推论：镜像可能损坏

2. 删除镜像，并重新安装 mysql

```
# 删除旧镜像
docker rmi mysql:9.2

# 重新安装
docker run -d \
  --name mysql92 \
  -p 3306:3306 \
  -e TZ=Asia/Shanghai \
  -e MYSQL_ROOT_PASSWORD=root \
  -v ./mysql/data:/var/lib/mysql \
  -v ./mysql/init:/docker-entrypoint-initdb.d \
  -v ./mysql/conf:/etc/mysql/conf.d \
  --network hm-net \
  --restart=always \
  mysql:9.2
```

#### 问题解决,故障原因：mysql:92 镜像因为异常停机导致镜像损坏
