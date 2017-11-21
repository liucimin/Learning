
[Xiaowei's Note](https://xiaoweiqian.github.io/note/)
## Introduction

Docker pluginV2 管理机制可以使得第三方插件以镜像的方式管理起来，插件不再需要独立部署和维护，而是由docker统一维护，各个节点可以方便的从镜像仓库安装、升级和删除插件。

## Prepare

- 网络插件开发参考

[网络插件开发](https://xiaoweiqian.github.io/note/docker-network-remote-driver/)

## Configration

推荐使用alpine基础镜像(4M)作为plugin的运行环境，插件制作过程分为如下几个步骤。

- 插件编译

  插件编译和插件运行环境必须一致，因此使用alpine作为编译环境。

  1. 编写Dockerfile.dev

     若插件基于golang开发，使用golang:1.7.5-alpine3.5基础镜像作为插件编译环境

     ```
      FROM golang:1.7.5-alpine3.5

      COPY . /go/src/github.com/XiaoweiQian/macvlan-driver
      WORKDIR /go/src/github.com/XiaoweiQian/macvlan-driver

      RUN set -ex \
          && apk add --no-cache --virtual .build-deps \
          gcc libc-dev linux-headers \
          && go install --ldflags '-extldflags "-static"' \
          && apk del .build-deps

      CMD ["/go/bin/macvlan-driver"]
     ```


     ```

  2. 编写Makefile

     启动插件编译容器，把编译成功的插件二进制文件拷贝到本地

     ```
      compile:
          @echo "### compile docker-macvlan plugin"
          @docker build -q -t builder -f Dockerfile.dev .
          @echo "### extract docker-macvlan"
          @docker create --name tmp builder
          @docker cp tmp:/go/bin/macvlan-driver ./docker-macvlan
          @docker rm -vf tmp
          @docker rmi builder

     ```

- 插件打包

  一个标准的docker第三方插件必须包含config.json文件和rootfs文件系统。

  1. 编写Dockerfile

     以alpine镜像为基础运行环境，把插件可执行文件打包到镜像中

     ```
      FROM alpine

      RUN mkdir -p /run/docker/plugins

      COPY docker-macvlan /usr/bin/docker-macvlan

      CMD ["/usr/bin/docker-macvlan"]

     ```

  2. 编写config.json

     配置插件基本参数

     ```
      {
      "description": "macvlan Net plugin for Docker swarm",
      "documentation": "",
      "entrypoint": [
          "docker-macvlan"
      ],
      "interface": {
          "socket": "macvlan_swarm.sock",
          "types": [
          "docker.networkdriver/1.0"
          ]
      },
      "linux": {
          "capabilities": [
          "CAP_SYS_ADMIN",
          "CAP_NET_ADMIN"
          ]
      },
      "mounts": null,
      "env": [
          {
          "description": "Extra args to `macvlan_swarm` and `plugin`",
          "name": "EXTRA_ARGS",
          "settable": [
              "value"
          ],
          "value": ""
          }
      ],
      "network": {
          "type": "host"
      },
      "workdir": ""
      }

     ```

  3. 编写Makefile

     生成插件文件系统rootfs，和config.json拷贝到插件目录plugin

     ```
      rootfs:
          @echo "### docker build: rootfs image with docker-macvlan"
          @docker build -q -t ${PLUGIN_NAME}:rootfs .
          @echo "### create rootfs directory in ./plugin/rootfs"
          @mkdir -p ./plugin/rootfs
          @docker create --name tmp ${PLUGIN_NAME}:rootfs
          @docker export tmp | tar -x -C ./plugin/rootfs
          @echo "### copy config.json to ./plugin/"
          @cp config.json ./plugin/
          @docker rm -vf tmp
          @docker rmi ${PLUGIN_NAME}:rootfs 

     ```

- 插件上传

  插件可以上传到hub公共仓库，也可以上传到本地私有仓库，如果要上传本地私有库，插件的名字必须按照此格式192.168.1.2:5000/plugin_name，才能上传成功。

  1. 编写Makefile

     通过指定插件目录plugin创建本地插件，启用插件并上传到镜像仓库

     ```
      create:
          @echo "### remove existing plugin ${PLUGIN_NAME} if exists"
          @docker plugin rm -f ${PLUGIN_NAME} || true
          @echo "### create new plugin ${PLUGIN_NAME} from ./plugin"
          @docker plugin create ${PLUGIN_NAME} ./plugin

      enable:
          @echo "### enable plugin ${PLUGIN_NAME}"
          @docker plugin enable ${PLUGIN_NAME}

      push:  clean compile rootfs create enable
          @echo "### push plugin ${PLUGIN_NAME}"
          @docker plugin push ${PLUGIN_NAME}

     ```

- 插件安装

  所有docker节点通过docker plugin install命令安装插件

  ```
    docker plugin install ${PLUGIN_NAME} --grant-all-permissions

  ```