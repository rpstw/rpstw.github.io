layout: post
title:  things about Docker
categories: [container]
toc: true

[TOC]

| 使用docker前                                                 | 使用docker后                                               |
| ------------------------------------------------------------ | ---------------------------------------------------------- |
| 代码部署在固定机器上，下次上线根本不知道要去哪里操作，只好手动遍历一遍线上机器 | 通过docker swarm统一管理上线的服务，不仅直接搜索服务、还能 |
| 上了线之后、发现有个包忘记安、安错了、和测试端版本不一致     | 这就是虚拟机或容器设计之初要解决的问题                     |
| 迁移脚本到另一个公用docker里，发现有个包忘记安、安错了、和之前环境版本不一致 | 同上                                                       |
| 维护另一个人的代码，发现根本跑不通，需要手动再安一遍ta安过的包 | 同上                                                       |
| 太多代码共享一个公用docker，一旦重启，要一个个手动重启       | Swarm集群避免单点失败问题，异常退出的容器会被自动重启      |
| 各种服务的ip写死在代码里，服务很难迁移                       | 同 swarm集群内的container可以按名字寻址                    |

# Archtecture of docker

- client
- daemon
- libnetwork
- runc

 since 2020 docker desktop will enable buildkit by default

# from Dockerfile to docker image

## The process of `docker-build`:

### v1

#### cli-side

1. Prepare build context
	1. Read .dockerignore and build patterns
	2. ValidateContextDirectory
	3. tar build context with patterns

2. POST `/build` https://docs.docker.com/engine/api/v1.41/#operation/ImageBuild
	query:
- path of tarred build context
- path of docker file
- image tag
- many other advanced parameters
	body:
tared build context

#### daemon-side 

Router -> Backend ->  build manager -> builder

3. parse Dockerfile to a list of stages each contains a list of commands
4. Initialize stage, pull and mount the base image
5. run every command of the stage

```sh
#1
Sending build context to Docker daemon  97.53MB #2
#3
Step 1/7 : FROM harbor.max.gg/heybox/heybox_django_mirror:1.0.0 #4
 ---> 374eadff470e
# start of 5
Step 2/7 : RUN pip install --index https://mirrors.bfsu.edu.cn/pypi/web/simple pytest
 ---> Using cache
 ---> dc1ab7875f6d
Step 3/7 : ARG WD=/home/code/heybox
 ---> Using cache
 ---> e36ed47a8c32
Step 4/7 : WORKDIR $WD
 ---> Using cache
 ---> 744808378b38
Step 5/7 : ENV PROJECT_ROOT_PATH $WD
 ---> Using cache
 ---> 9b3d1e43368f
Step 6/7 : ENV DJANGO_SETTINGS_MODULE dotamax.settings
 ---> Using cache
 ---> dbaeda21d09d
Step 7/7 : CMD ["gunicorn", "--access-logfile", "-", "--bind","0.0.0.0:80", "--reload", "dotamax.wsgi:application"]
 ---> Using cache
 ---> 731c21a93d22
# end of 5
Successfully built 731c21a93d22
Successfully tagged harbor.max.gg/heybox/django-dev:latest
```

### v2

todo

# implementation of docker file commands

Init: FROM



update Config and commit a new image without aufs/overlay2 layer: ENV ,WORKDIR,CMD

Config: platform independent infos



add a new layer: ADD, COPY,RUN



ADD/COPY

mount a new RWLayer at a path p called root path (p is different from the origin ro layer)

copy files to root path / dest path

commmit rw layer to a new ro layer ( tar the file tree at root path to another place and save it)

export image Create a new image based on lower image but with a new diff id(rolayer)



RUN

create a container

run sh inside it

commit the container

> how does container commit works?

# structure of a docker image 

run config  platform independent

Layer ids `Image.RootFS.DiffIDs`