# openshift build

[TOC]

## Build基本介绍
 ---
### Openshift与Kubernets资源比较
先通过命令来从系统资源对象的角度openshift与kunetes的基本区别

    [cloud@centos ~]$ oc get all
    
    NAME            TYPE      FROM         LATEST
    bc/cakephp-ex   Source    Git@master   1
    bc/nodejs-ex    Source    Git@master   13
    
    NAME                  TYPE      FROM         STATUS                          STARTED   DURATION
    builds/cakephp-ex-1   Source    Git@master   Failed (ExceededRetryTimeout)             
    
    NAME                    DOCKER REPO                                    TAGS      UPDATED
    is/cakephp-ex           172.30.1.1:5000/myproject/cakephp-ex                     
    is/nodejs-010-centos7   172.30.1.1:5000/myproject/nodejs-010-centos7   latest    5 days ago
    is/nodejs-ex            172.30.1.1:5000/myproject/nodejs-ex            latest    5 days ago
    is/parksmap             172.30.1.1:5000/myproject/parksmap             1.0.0     7 days ago
    
    NAME            REVISION   DESIRED   CURRENT   TRIGGERED BY
    dc/cakephp-ex   0          1         0         config,image(cakephp-ex:latest)
    dc/nodejs-ex    4          1         0         config,image(nodejs-ex:latest)
    dc/parksmap     1          1         1         config,image(parksmap:1.0.0)
    
    NAME             DESIRED   CURRENT   READY     AGE
    rc/nodejs-ex-1   1         1         1         5d
    rc/nodejs-ex-2   0         0         0         5d
    rc/nodejs-ex-3   0         0         0         5d
    rc/nodejs-ex-4   0         0         0         5d
    rc/parksmap-1    1         1         1         7d
    
    NAME             CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
    svc/cakephp-ex   172.30.21.153   <none>        8080/TCP   18h
    svc/nodejs-ex    172.30.226.43   <none>        8080/TCP   5d
    svc/parksmap     172.30.87.225   <none>        8080/TCP   7d
    
    NAME                   READY     STATUS    RESTARTS   AGE
    po/nodejs-ex-1-h4f3f   1/1       Running   0          5d
    po/parksmap-1-k1n0g    1/1       Running   0          6h

使用kubectl查询：

    [cloud@centos ~]$ kubectl get all
    NAME                   READY     STATUS    RESTARTS   AGE
    po/nodejs-ex-1-h4f3f   1/1       Running   0          5d
    po/parksmap-1-k1n0g    1/1       Running   0          6h
    
    NAME             DESIRED   CURRENT   READY     AGE
    rc/nodejs-ex-1   1         1         1         5d
    rc/nodejs-ex-2   0         0         0         5d
    rc/nodejs-ex-3   0         0         0         5d
    rc/nodejs-ex-4   0         0         0         5d
    rc/parksmap-1    1         1         1         7d
    
    NAME             CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
    svc/cakephp-ex   172.30.21.153   <none>        8080/TCP   18h
    svc/nodejs-ex    172.30.226.43   <none>        8080/TCP   5d
    svc/parksmap     172.30.87.225   <none>        8080/TCP   7d

我们发现openshift相比kubenetes增加一些资源对象类型：BC（BuildConfig）、build、is(imagestream)、DC（DeploymentConfig，所以OpenShift从PaaS平台以及DevOps角度上讲，增加了与开发者紧密相关的构建过程，可以理解为CI过程，kubernetes则主要覆盖CD过程。

---

### 什么是Build？
每一个build对用户来讲都是表示着一次完整构建，当然build对象本身描述的是该构建过程所有相关的配置信息。

下面举例说明：

    [cloud@centos ~]$ oc export  builds/cakephp-ex-1 -o json
    {
     (1) "kind": "Build",
        "apiVersion": "v1",
        "metadata": {
            "name": "cakephp-ex-1",
            "creationTimestamp": null,
            "labels": {
                "app": "cakephp-ex",
                "buildconfig": "cakephp-ex",
                "openshift.io/build-config.name": "cakephp-ex",
                "openshift.io/build.start-policy": "Serial"
            },
            "annotations": {
                "openshift.io/build-config.name": "cakephp-ex",
                "openshift.io/build.number": "1"
            }
        },
        "spec": {
            "serviceAccount": "builder",
     (2)    "source": {
                "type": "Git",
                "git": {
                    "uri": "https://github.com/openshift/cakephp-ex.git",
                    "ref": "master"
                }
            },
     (3)    "strategy": {
                "type": "Source",
                "sourceStrategy": {
                    "from": {
                        "kind": "DockerImage",
                        "name": "centos/php-70-centos7@sha256:6eab2de3501f637a20eba52c12c49dff72c4bb162ce40043c245cc6add287782"
                    }
                }
            },
     (4)   "output": {
                "to": {
                    "kind": "ImageStreamTag",
                    "name": "cakephp-ex:latest"
                },
                "pushSecret": {
                    "name": "builder-dockercfg-sbqww"
                }
            },
     (5)    "resources": {},
     (6)    "postCommit": {},
     (7)    "nodeSelector": null,
     (8)    "triggeredBy": [
                {
                    "message": "Build configuration change"
                }
            ]
        },
     (9)  "status": {
            "phase": "New",
            "reason": "ExceededRetryTimeout",
            "message": "Failed to create build pod: pods \"cakephp-ex-1-build\" is forbidden: failed quota: quota1: must specify cpu,memory.",
            "outputDockerImageReference": "172.30.1.1:5000/myproject/cakephp-ex:latest",
            "config": {
                "name": "cakephp-ex"
            },
            "output": {}
        }
    }

### 什么是BuildConfig？

    [cloud@centos ~]$ oc export  bc/cakephp-ex -o json
    {
     (1) "kind": "BuildConfig",
        "apiVersion": "v1",
        "metadata": {
            "name": "cakephp-ex",
            "creationTimestamp": null,
            "labels": {
                "app": "cakephp-ex"
            },
            "annotations": {
                "openshift.io/generated-by": "OpenShiftNewApp"
            }
        },
        "spec": {
      (2)    "triggers": [
                {
                    "type": "GitHub",
                    "github": {
                        "secret": "XqZqQrmN2_ml5UqepQyf"
                    }
                },
                {
                    "type": "Generic",
                    "generic": {
                        "secret": "xAeGxPx9MzNwZsw_6LeY"
                    }
                },
                {
                    "type": "ConfigChange"
                },
                {
                    "type": "ImageChange",
                    "imageChange": {}
                }
            ],
      (3)    "runPolicy": "Serial",
      (4)    "source": {
                "type": "Git",
                "git": {
                    "uri": "https://github.com/openshift/cakephp-ex.git",
                    "ref": "master"
                }
            },
      (5)     "strategy": {
                "type": "Source",
                "sourceStrategy": {
                    "from": {
                        "kind": "ImageStreamTag",
                        "namespace": "openshift",
                        "name": "php:7.0"
                    }
                }
            },
      (6)    "output": {
                "to": {
                    "kind": "ImageStreamTag",
                    "name": "cakephp-ex:latest"
                }
            },
      (7)    "resources": {},
      (8)    "postCommit": {},
      (9)    "nodeSelector": null
        },
        "status": {
            "lastVersion": 0
        }
    }


---
## Build的基本操作

- 运行一个Build
	* oc start-build < buildconfig_name >
	* oc start-build --from-build=< build_name >
	* oc start-build < buildconfig_name > --follow
	* oc start-build < buildconfig_name > --env=< key >=< value >
- 取消一个Build
    * oc cancel-build < build_name >
    * oc cancel-build < build1_name > < build2_name > < build3_name >
    * oc cancel-build bc/< buildconfig_name >
    * oc cancel-build bc/< buildconfig_name >  --state=< state >
- 删除一个BuildConfig
    * oc delete bc < BuildConfigName >
    * oc delete --cascade=false bc < BuildConfigName >
   
- 查看构建细节
    * oc describe build < build_name >
- 查看构建日志
    * oc logs -f build/< build_name >
    * oc logs -f bc/< buildconfig_name >  // log latest
    * oc logs --version=< number > bc/< buildconfig_name >
    * 设置日志的打印级别
      * oc start-build < buildconfig_name > --build-loglevel=5 //highest

## Build运行的规则
构建策略的设置只能在BC中进行配置，build是不能配置的。因为build本身表示某一次具体的构建，是否串行还是并行只能有上层的BC来进行控制。

> 代码：https://github.com/openshift/nodejs-ex.git
> 编译镜像：docker.io/openshift/nodejs-010-centos7:latest

- 串行
- 串行最新
- 并行

## Build 相关输入项

 - [source] Inline Dockerfile definitions  
 - [source] Content extracted from existing images  
 - [source] Git repositories  
 - [source] Binary inputs  
 - [build inputs] Input secrets 
 - [build inputs] External artifacts 


### **DockerFile** 
如果BuildConfig.spec.source.type的类型是Dockerfile，那么input就只能是inline DockerFile。

    source:
      type: "Dockerfile"
      dockerfile: "FROM centos:7\nRUN yum install -y httpd"
### **Image Source**
主要是把构建过程中需要的额外的文件打包到image中，在引用的时候可以通过引用image的方式，从而来获取（拷贝）build过程所有需要的文件。引用image的方式可以是直接应用docker image，也可以引用image stream。注意拷贝image中文件时，一定要定义好两个路径信息，一个destinationDir，另一个是sourcePath。
destinationDir必须是相对路径，相对于构建进程的路径。而sourcePath必须是绝对路径，也就是保存构建文件image内的绝对路径。举个例子：

    source:
      git:
        uri: https://github.com/openshift/ruby-hello-world.git
      images: (1)
      - from: (2)
          kind: ImageStreamTag
          name: myinputimage:latest
          namespace: mynamespace
        paths: (3)
        - destinationDir: injected/dir (4)
          sourcePath: /usr/lib/somefile.jar (5)
      - from:
          kind: ImageStreamTag
          name: myotherinputimage:latest
          namespace: myothernamespace
        pullSecret: mysecret (6)
        paths:
        - destinationDir: injected/dir
          sourcePath: /usr/lib/somefile.jar
参数项说明：
(1) 保存镜像或者文件的数组列表
(2) 保存构建文件（将会被拷贝）的镜像信息
(3) 一组成对（destinationDir/sourcePath）的路径信息列表, 简单理解为从哪里拷贝文件并拷贝到哪里去
(4) 目的目录-相对目录.
(5) 源路径-绝对路径，在image中拷贝文件的路径
(6)secret信息（可选），如果在pull image时需要
    

> 需要注意的是，在用户定制构建中（custom build）不支持这一特性，也就是不支持把构建文件存放的另一个image中。
### **Git 仓库地址**
Prerequisite：BuildConfig.spec.source.type = Git
如果build 源类型是git，那么必须提供一个git代码仓库路径。而inline Dockerfile可选的。如果配置的dockerfile一项，那么在ContextDir包含的dockerfile（如果存在）将会被替换。也就是buildconfig配置优先。

    source:
      type: "Git"
      git: (1)
        uri: "https://github.com/openshift/ruby-hello-world"
        ref: "master"
      contextDir: "app/dir" (2)
      dockerfile: "FROM openshift/ruby-22-centos7\nUSER example" (3)
(1) git仓库的路径信息，ref为具体的分支信息，格式可以为SHA1或者分支名称
(2) 主要是应用所在的代码目录是给出git url中的一个子目录，即应用代码真正存在与对应的子目录下，而非默认的根目录。如果指定该字段，那么默认的根目录路径将被替换。
(3) 如果在source类型给git的配置中，包含了dockerfile项，那么在应用目录下的存在的已有的dockerfile文件内容将会被覆盖。

> 如果build config中不指定ref信息，那么openshift 默认clone
> 主分支的HEAD代码，就是clone深度为1（--depth=1），这个深度不是目录深度，而是版本管理深度。为1表示只clone最后一个版本，其他版本信息不保存。就是没有了版本控制。

#### *proxy*

    source:
      type: Git
      git:
        uri: "https://github.com/openshift/ruby-hello-world"
        httpProxy: http://proxy.example.com
        httpsProxy: https://proxy.example.com
        noProxy: somedomain.com, otherdomain.com
#### [*Source Clone Secrets*](https://docs.openshift.org/latest/dev_guide/builds/build_inputs.html#source-code)

> 与secret相关的内容，在Binary Source会有基本介绍，后续会输出一个单独文档进行讲解

### **二进制文件**

> BuildConfig.spec.source.type = Binary binary类型的source比较特别，因为它只能通过使用oc start-build命令来进行使用。这种方式的源是不能够通过进行触发同时也不能通过web console的方式进行构建。 使用的参数：

- --from-file：
- --from-dir 与 --from-repo：
- --from-archive：

> 如果BuildConfig中配置了Binary source，而在oc start-build命令同时也制定了binary，那么buildconfig中的 Binary会被替代。
> 如果BuildConfig配置了Git source，而在oc start-build命令同时也指定了binary，客户端的binary会被处理。
> 注意：Git source 与Binary source互斥。
参数比较多，下面主要通过来进行理解：

### **构建使用的Secrets**
使用的场景主要是需要用户的认证信息，但是由于安全原因又不想暴露该信息。这时我们就需要输出secret，当然首先应该创建。Secret的详细解释可以参考[k8s](https://kubernetes.io/docs/user-guide/secrets/)，[openshift](https://docs.openshift.org/latest/dev_guide/secrets.html)也给出很详细的解释。
这里举两个例子简单说明一下

   

    $ oc secrets new secret-npmrc .npmrc=~/.npmrc (例1)
    $ oc secrets new my-secret .dockerconfigjson=.docker/config.json (例2)
    $ oc export  secret my-secret
    apiVersion: v1
    data:
      .dockerconfigjson: ewoJImF1dGhzIjogewoJCSJodHRwczovL2luZGV4LmRvY2tlci5pby92MS8iOiB7CgkJCSJhdXRoIjogImFHRnBZbWxoYm5ocFlXOWtiMjVuWW1WcE9tUnZZMnRsY2tGaE9EZzRPRGc0IgoJCX0KCX0KfQ==
    kind: Secret
    metadata:
      creationTimestamp: null
      name: my-secret
    type: kubernetes.io/dockerconfigjson
           
容器启动时，根据secret的配置，把系统（etcd）中保存的secret信息，保存的容器的数据卷中，文件名称通常与原始挂在认证配置文件相同。但是需要注意的是最新的docker配置文件: .docker/config.json, 而该配置文件通过secret保存后，重新映射到容器内的数据卷后，名字为.dockerconfigjson。应为我们的数据保存层secret时，是不支持多级目录作为key的。

我们下面以例子1中创建的secret secret-npmrc 进行build过程中如何使用secret

BuildConfig中配置(更新已存在的BuildConfig)
   
    source:
      git:
        uri: https://github.com/openshift/nodejs-ex.git
      secrets:
        - secret:
            name: secret-npmrc
      type: Git

命令行配置(创建新的BuildConfig)

    $ oc new-build \
        openshift/nodejs-010-centos7~https://github.com/openshift/nodejs-ex.git \
        --build-secret secret-npmrc
secret在image中的保存位置，通常就在工作目录下，也就是代码的路径下（sti stragety），当然这个路径是可以设置的。在DockerFile中使用WORKDIR，在BuildConfig中可以指定destinationDir。例如：

    source:
      git:
        uri: https://github.com/openshift/nodejs-ex.git
      secrets:
        - secret:
            name: secret-npmrc
          destinationDir: /etc
      type: Git

> 注意1：如果构建的策略是sti，目的目录如果设置，那么必须存在，因为在整个build过程中，不会创建任何新的目录。
> 注意2：如果构建的策略是sti，目前所有的secret的权限被设置成0666，就是说可以任意的读写，写的原因主要是在执行完assemble脚后，进行文件清空。对于清空的原因主要还是安全的考虑，因为assemble之后，主要就是编译运行阶段，这时的image内最初需要使用的secret，现在已经不需要了。

### **使用外部依赖包**
主要是在构建过程中，需要依赖额外的报，比如jar文件。对于不同的构建策略来说，具体的使用方式是不同的。
对于sti stragety，我们需要再assemble脚本中，加入下载依赖的包的shell命令，比如：wget  *.jar -o app_dep.jar，在运行应用的时候可能也需要更改run脚本，来运行该依赖。
对于docker stragety，我们需要修改dockerfile，同时加入RUN命令来下载相关的依赖，比如：RUN wget *.jar -O app_dep.jar

> 对于这几种构建策略，依赖的保存位置，可以通过设置环境变量的方式来更改。

- Using the .s2i/environment file (仅sti 策略有效)
- Setting in BuildConfig（all）
- Providing explicitly using oc start-build --env (仅手动构建有效)

### **访问私有仓库需要的证书信息**
主要就是secret的理解，对于secret后面单独介绍**strong text**
主要了解，如果创建secret，如果把secret赋予个service account。
如openshift中：
$ oc secrets new dockerhub ~/.docker/config.json
$ oc secrets link builder dockerhub
$ oc set build-secret --push bc/sample-build dockerhub
或者直接是修改BC
PUSH:

    spec:
      output:
        to:
          kind: "DockerImage"
          name: "private.registry.com/org/private-image:latest"
        pushSecret:
          name: "dockerhub"
PULL:

    strategy:
      sourceStrategy:
        from:
          kind: "DockerImage"
          name: "docker.io/user/private_repository"
        pullSecret:
          name: "dockerhub"
      type: "Source"

## Build的构建策略

- s2i build
- *docker build*
- *custom build*
- *pipeline build*

### S2I build

首先STI是一个工具，该工具根据开发者（用户的输入）生成Image

#### Build Process
- 三要素

 * sources
 * S2I script
 * builder image
 
- 主要流程
在构建的过程中，S2I负责要把sources与S2I scripts注入到编译镜像中，为了实现这一目标，S2I通过对sources 以及S2I scripts进行打包，然后把tar文件注入到构建镜像中。在构建镜像中对该tar文件具体解压的位置由***--destination***参数或者构建镜像的***io.openshift.s2i.destination*** label指定。如果没有指定，那么默认使用/tmp目录。同时构建的过程还需要安装tar工具以及/bin/bash，如果builder image中没有tar与/bin/bash那么sti 就会强拉一个安装该基本环境的docker image。下面是sti 构建的流图：
![sti flow](https://github.com/openshift/source-to-image/blob/master/docs/sti-flow.png)


####S2I scripts
  开发者可以使用任何语言来编写自己的S2I scripts，只要这个语言在构建容器中能够被有效的执行。S2I支持多种的选项来确认assemble/run/save-artifacts scripts各功能脚本的存放位置。每次构建前s2i都会按照如下的顺序检查下面的配置路径，来确定脚本的存放位置。

 1. --scripts-url
 2. .s2i/bin 目录
 3. io.openshift.s2i.scripts-url label （builder image）

同时--scripts-url 与  io.openshift.s2i.scripts-url 又同时可以按照如下的格式进行脚本路径的的指定

 - image:///path_to_scripts_dir  说明：镜像中的绝对路径
 - file:///path_to_scripts_dir   说明：docker运行宿主机上的相对或者绝对路径（相对的是？）
 - http(s)://path_to_scripts_dir  说明：存放s2i脚本的网络地址

下面主要说明S2I相关的主要脚本类别与功能配置

| 脚本类别 | 主要功能描述 |
|---|---|
| assemble(必选) | 主要负责通过制定的sources构建出对应的应用，同时把编译完成的组件或者应用放到合适的位置。主要的工作流如下: 1. 准备编译环境，如果需要增量编译务必要设置***save-artifacts***选项（默认为可选项）2.把指定的source放到制定的目录下 3.开始构建应用 4.把构建出的应用|
|run（必选）|执行或者启动应用的脚本|
|save-artifacts（可选）|主要是用来支持增量编译，加入编译过程，主要过程是打包（tar）相关依赖，然后再次编译的时候进行注入。是否有默认的配置？？|
|usage（可选）|主要是向其他用户说明如何使用该image|
|test/run(可选)|主要允许用户创建新的进程来检查是否image正常的工作|
举例：
example 1. assemble script

    #!/bin/bash
    
    # restore build artifacts
    if [ "$(ls /tmp/s2i/artifacts/ 2>/dev/null)" ]; then
        mv /tmp/s2i/artifacts/* $HOME/.
    fi
    
    # move the application source
    mv /tmp/s2i/src $HOME/src
    
    # build application artifacts
    pushd ${HOME}
    make all
    
    # install the artifacts
    make install
    popd

example 2. run script

    #!/bin/bash
    
    # run the application
    /opt/application/run.sh

example 3. save-artifacts script

    #!/bin/bash
    
    pushd ${HOME}
    if [ -d deps ]; then
        # all deps contents to tar stream
        tar cf - deps
    fi
    popd

example 4. usage script:

    #!/bin/bash
    
    # inform the user how to use the image
    cat <<EOF
    This is a S2I sample builder image, to use it, install
    https://github.com/openshift/source-to-image
    EOF

####使用images的[ONBUILD](https://docs.docker.com/engine/reference/builder/#onbuild)功能？

*如果用户的builder image不含有执行S2I所需要的基本条件，即镜像中没有安装/bin/bash 或者 tar工具，那么就不会执行一个container build来对存放build的输入。但是如果builder中包含了ONBUILD指令，那么这个S2I过程就会失败。因为这就意味着执行了比较通用的容器内构件过程，这个过程相比S2I是不安全的，必须要显式的对权限进行限制。*
因此，如果用户想要顺利的执行S2I的过程，那么必须满足下面的一个条件：

 1. builder image中安装了/bin/bash 已经tar工具
 2. builder image中不能包含ONBUILD指令
 
### Docker Build
*下一篇*

### Custom Build
*下一篇*

### Pipeline Build
*下一篇*

## sti 工具

> 仓库地址：https://github.com/openshift/source-to-image.git

**s2i build 命令**

| 选项参数 | 参数类型 | 功能说明 |
| --- | --- | --- | 
| --destination | string | 指明解压打包的源代码以及assemble相关脚本文件的容器内路径 | 
| -s, --scripts-url | string  | 远端存放assemble脚本的url信息 |
| --copy | string | 使用本地文件系统的源代码 |
| --context-dir | string | 仓库代码子目录 |
|  -E, --environment-file | string   | 制定保存环境变量的文件路径 |
|  --exclude | 正则表达式 | 不包含在构建(编译)过程中的文件信息，默认的表达式类型 "(^|/)\.git(/|$)",如果设置成"",表示所有的文件都参与构建过程 |
| --description | string | 应用的描述 |
| -e, --env | string | 编译环境变量设置 NAME=VALUE|
| --dockercfg-path | string | 应该是sti使用的dockercfg信息，因为可能要pull builder image 默认是"$HOME/.docker/config.json" |
| -n, --application-name | string | 指明应用名称 |
|  --assemble-user | string | 制定执行assemble脚本的用户 |
| --incremental-pull-policy | string | 表示什么时候pull 之前支持增量编译的镜像，(always, never or if-not-present) (default "if-not-present") |
| -p, --pull-policy | string | pull编译镜像的规则，(always, never or if-not-present) |
| -q, --quiet | 不需赋值 | 不显示非错误性的输出 Suppress all non-error output |
|-r, --ref | string | 分支或者tag |
|   --callback-url | string | build完成后，调用HTTP POST方法执行该url |
|  --rm | 不需赋值 | 在增量编译中，删除之前使用的镜像  |
|  -a, --runtime-artifact | string  | 在双编程框架中，指定定拷贝编译容器中的哪些文件或者目录到运行时镜像中|
| --runtime-image |string | 指定运行时使用的image |
| --save-temp-dir  | 不需要赋值 | 保存s2i过程中的使用的临时目录信息 |
| -s, --scripts-url | string | 指定保存assemble, assemble-runtime and run scripts 远端url信息|

查看编译镜像：

    [cloud@centos openshift]$ docker run -ti centos/ruby-22-centos7 /bin/bash
    bash-4.2$ pwd   
    /opt/app-root/src
    bash-4.2$ ls -la
    total 0
    drwxrwxr-x. 3 default root 18 Feb 16 13:08 .
    drwxrwxr-x. 4 default root 42 Feb 21 09:57 ..
    drwxrwxr-x. 3 default root 19 Feb 16 13:08 .pki
远端代码仓库文件：

![enter image description here](https://github.com/guangxuli/openshift-ebook/blob/master/build/ruby-ex.jpg)

开始构建：

    [cloud@centos openshift]$ s2i build https://github.com/openshift/ruby-hello-world centos/ruby-22-centos7 hello-world-app:1.0
    ---> Installing application source ...
    ---> Building your Ruby application from source ...
    ---> Running 'bundle install --deployment --without development:test' ...
    Fetching gem metadata from https://rubygems.org/..........
    Installing rake 10.3.2
    Installing i18n 0.6.11
    Installing json 1.8.6
    Installing minitest 5.4.2
    Installing thread_safe 0.3.4
    Installing tzinfo 1.2.2
    Installing activesupport 4.1.7
    Installing builder 3.2.2
    Installing activemodel 4.1.7
    Installing arel 5.0.1.20140414130214
    Installing activerecord 4.1.7
    Installing mysql2 0.3.16
    Installing rack 1.5.2
    Installing rack-protection 1.5.3
    Installing tilt 1.4.1
    Installing sinatra 1.4.5
    Installing sinatra-activerecord 2.0.3
    Using bundler 1.7.8
    Your bundle is complete!
    Gems in the groups development and test were not installed.
    It was installed into ./bundle
    ---> Cleaning up unused ruby gems ...

查看应用镜像输出结果：

    [cloud@centos openshift]$ docker images | grep hello
    hello-world-app    1.0      956c161ead31        About a minute ago   456.8 MB
    
查看应用镜像内部文件：

    [cloud@centos openshift]$ docker run -it hello-world-app:1.0  /bin/bash
    bash-4.2$ pwd
    /opt/app-root/src
    bash-4.2$ ls -la
    total 40
    drwxrwxr-x. 10 default root  274 Mar  7 11:15 .
    drwxrwxr-x.  4 default root   42 Feb 21 09:57 ..
    drwxrwxr-x.  2 default root   20 Mar  7 11:15 .bundle
    -rw-rw-r--.  1 default root   11 Mar  7 11:15 .gitignore
    drwxrwxr-x.  3 default root   19 Mar  7 11:15 .pki
    drwxrwxr-x.  3 default root   36 Mar  7 11:15 .s2i
    -rw-rw-r--.  1 default root  278 Mar  7 11:15 Dockerfile
    -rw-rw-r--.  1 default root  111 Mar  7 11:15 Gemfile
    -rw-rw-r--.  1 default root  952 Mar  7 11:15 Gemfile.lock
    -rw-rw-r--.  1 default root  182 Mar  7 11:15 README.md
    -rw-rw-r--.  1 default root  222 Mar  7 11:15 Rakefile
    -rw-rw-r--.  1 default root 1444 Mar  7 11:15 app.rb
    drwxrwxr-x.  3 default root   18 Mar  7 11:15 bundle
    drwxrwxr-x.  2 default root   45 Mar  7 11:15 config
    -rw-rw-r--.  1 default root   92 Mar  7 11:15 config.ru
    drwxrwxr-x.  3 default root   21 Mar  7 11:15 db
    -rw-rw-r--.  1 default root   93 Mar  7 11:15 models.rb
    -rwxrwxr-x.  1 default root   35 Mar  7 11:15 run.sh
    drwxrwxr-x.  2 default root   28 Mar  7 11:15 test
    drwxrwxr-x.  2 default root   22 Mar  7 11:15 views
    
与远端原始代码目录比较发现，远端的代码已经完全注入到新的镜像中。

**s2i rebuild** 

>  s2i rebuild < image > [< new-tag >] [flags]

支持的选项包括： --callback-url、--destination、--dockercfg-path、--incremental、--incremental-pull-policy、--pull-policy、--quiet、--rm、--save-temp-dir。这些选项的功能参考s2i build命令

实践：

> [cloud@centos ~]$  s2i rebuild -p never  hello-world-app:1.2

[cloud@centos ~]$ s2i rebuild -p never  hello-world-app:1.2
    
    Checking if image "hello-world-app:1.2" is available locally ...
    Checking if image "centos/ruby-22-centos7" is available locally ...
    Checking if image "centos/ruby-22-centos7" is available locally ...
    Already on 'master'
    ---> Installing application source ...
    ---> Building your Ruby application from source ...
    ---> Running 'bundle install --deployment --without development:test' ...
    Fetching gem metadata from https://rubygems.org/..........
    Installing rake 10.3.2
    Installing i18n 0.6.11
    Installing json 1.8.6
    Installing minitest 5.4.2
    Installing thread_safe 0.3.4
    Installing tzinfo 1.2.2
    Installing activesupport 4.1.7
    Installing builder 3.2.2
    Installing activemodel 4.1.7
    Installing arel 5.0.1.20140414130214
    Installing activerecord 4.1.7
    Installing mysql2 0.3.16
    Installing rack 1.5.2
    Installing rack-protection 1.5.3
    Installing tilt 1.4.1
    Installing sinatra 1.4.5
    Installing sinatra-activerecord 2.0.3
    Using bundler 1.7.8
    Your bundle is complete!
    Gems in the groups development and test were not installed.
    It was installed into ./bundle
    ---> Cleaning up unused ruby gems ...
重新构建的信息s2i如何知道？通过查看镜像信息发现。

    [cloud@centos ~]$ docker inspect hello-world-app:1.2
    
    "OnBuild": null,
                "Labels": {
                    "build-date": "20161214",
                    "io.k8s.description": "Platform for building and running Ruby 2.2 applications",
                    "io.k8s.display-name": "hello-world-app:1.2",
                    "io.openshift.builder-base-version": "bfd4736",
                    "io.openshift.builder-version": "06c0ec324aa6edf151f4ea1e7304199c72011bec",
                    "io.openshift.expose-services": "8080:http",
                    "io.openshift.s2i.build.commit.author": "Ben Parees \u003cbparees@users.noreply.github.com\u003e",
                    "io.openshift.s2i.build.commit.date": "Fri Mar 3 15:29:12 2017 -0500",
                    "io.openshift.s2i.build.commit.id": "022d87e4160c00274b63cdad7c238b5c6a299265",
                    "io.openshift.s2i.build.commit.message": "Merge pull request #58 from junaruga/feature/fix-for-ruby24",
                    "io.openshift.s2i.build.commit.ref": "master",
                    "***io.openshift.s2i.build.image": "centos/ruby-22-centos7***",
                    "***io.openshift.s2i.build.source-location": "https://github.com/openshift/ruby-hello-world***",
                    "***io.openshift.s2i.scripts-url": "image:///usr/libexec/s2i***",
                    "io.openshift.tags": "builder,ruby,ruby22",
                    "io.s2i.scripts-url": "image:///usr/libexec/s2i",
                    "license": "GPLv2",
                    "name": "CentOS Base Image",
                    "vendor": "CentOS"
                }

**s2i completion 命令**

  

     [cloud@centos ~]$ s2i completion bash  
        ...
    s2i_rebuild()  {
                last_command="s2i_rebuild"
                commands=()
                flags=()
                two_word_flags=()
                local_nonpersistent_flags=()
                flags_with_completion=()
                flags_completion=()
                flags+=("--callback-url=")
                local_nonpersistent_flags+=("--callback-url=")
                flags+=("--destination=")
                two_word_flags+=("-d")
                local_nonpersistent_flags+=("--destination=")
                flags+=("--dockercfg-path=")
                local_nonpersistent_flags+=("--dockercfg-path=")
                flags+=("--force-pull")
                local_nonpersistent_flags+=("--force-pull")
                flags+=("--incremental")
                local_nonpersistent_flags+=("--incremental")
                flags+=("--incremental-pull-policy=")
                local_nonpersistent_flags+=("--incremental-pull-policy=")
                flags+=("--pull-policy=")
                two_word_flags+=("-p")
                local_nonpersistent_flags+=("--pull-policy=")
                flags+=("--quiet")
                flags+=("-q")
                local_nonpersistent_flags+=("--quiet")
                flags+=("--rm")
                local_nonpersistent_flags+=("--rm")
                flags+=("--save-temp-dir")
                local_nonpersistent_flags+=("--save-temp-dir")
                flags+=("--ca=")
                flags+=("--cert=")
                flags+=("--key=")
                flags+=("--loglevel=")
                flags+=("--tls")
                flags+=("--tlsverify")
                flags+=("--url=")
                two_word_flags+=("-U")
                must_have_one_flag=()
                must_have_one_noun=()
                noun_aliases=()
            }

### oc 客户端build过程
#### 使用本地代码库编译
> [cloud@centos ruby-hello-world]$  oc new-build . --docker-image=centos/ruby-22-centos7:latest

    --> Found Docker image bce81d0 (2 weeks old) from Docker Hub for "centos/ruby-22-centos7:latest"

    Ruby 2.2 
    -------- 
    Platform for building and running Ruby 2.2 applications

    Tags: builder, ruby, ruby22

    * An image stream will be created as "ruby-22-centos7:latest" that will track the source image
    * The source repository appears to match: ruby
    * A source build using source code from https://github.com/openshift/ruby-hello-world.git#master will be created
      * The resulting image will be pushed to image stream "ruby-hello-world:latest"
      * Every time "ruby-22-centos7:latest" changes a new build will be triggered

    --> Creating resources with label build=ruby-hello-world ...
        imagestream "ruby-22-centos7" created
        imagestream "ruby-hello-world" created
        buildconfig "ruby-hello-world" created
    --> Success
        Build configuration "ruby-hello-world" created and build triggered.
        Run 'oc logs -f bc/ruby-hello-world' to stream the build progress


> [cloud@centos ruby-hello-world]$ oc get all

    NAME                  TYPE      FROM         LATEST
    bc/ruby-hello-world   Source    Git@master   1
    
    NAME                        TYPE      FROM          STATUS     STARTED              DURATION
    builds/ruby-hello-world-1   Source    Git@022d87e   Complete   About a minute ago   6s
    
    NAME                  DOCKER REPO                                  TAGS      UPDATED
    is/ruby-22-centos7    172.30.1.1:5000/myproject/ruby-22-centos7    latest    About a minute ago
    is/ruby-hello-world   172.30.1.1:5000/myproject/ruby-hello-world   latest    41 seconds ago
    
    NAME                          READY     STATUS      RESTARTS   AGE
    po/ruby-hello-world-1-build   0/1       Completed   0          1m

> [cloud@centos ruby-hello-world]$ docker images

    REPOSITORY                                      TAG                 IMAGE ID            CREATED              SIZE
    172.30.1.1:5000/myproject/ruby-hello-world      latest              11bf0cfee2e6        About a minute ago   456.8 MB

#### 使用远端代码库编译

> oc new-build
> openshift/nodejs-010-centos7~https://github.com/openshift/nodejs-ex.git

    [cloud@centos ruby-hello-world]$ oc new-build openshift/nodejs-010-centos7~https://github.com/openshift/nodejs-ex.git
    --> Found Docker image b3b1ce7 (3 months old) from Docker Hub for "openshift/nodejs-010-centos7"
    
        Node.js 0.10 
        ------------ 
        Platform for building and running Node.js 0.10 applications
    
        Tags: builder, nodejs, nodejs010
    
        * An image stream will be created as "nodejs-010-centos7:latest" that will track the source image
        * A source build using source code from https://github.com/openshift/nodejs-ex.git will be created
          * The resulting image will be pushed to image stream "nodejs-ex:latest"
          * Every time "nodejs-010-centos7:latest" changes a new build will be triggered
    
    --> Creating resources with label build=nodejs-ex ...
        imagestream "nodejs-010-centos7" created
        imagestream "nodejs-ex" created
        buildconfig "nodejs-ex" created
    --> Success
        Build configuration "nodejs-ex" created and build triggered.
        Run 'oc logs -f bc/nodejs-ex' to stream the build progress.

> [cloud@centos ruby-hello-world]$ oc get all

    NAME                  TYPE      FROM         LATEST
    bc/ruby-hello-world   Source    Git@master   1
    
    NAME                        TYPE      FROM          STATUS     STARTED         DURATION
    builds/ruby-hello-world-1   Source    Git@022d87e   Complete   7 minutes ago   6s
    
    NAME                  DOCKER REPO                                  TAGS      UPDATED
    is/ruby-22-centos7    172.30.1.1:5000/myproject/ruby-22-centos7    latest    7 minutes ago
    is/ruby-hello-world   172.30.1.1:5000/myproject/ruby-hello-world   latest    7 minutes ago
    
    NAME                          READY     STATUS      RESTARTS   AGE
    po/ruby-hello-world-1-build   0/1       Completed   0          7m

#### 其他构建

> oc new-build https://github.com/openshift/ruby-hello-world#beta2
> oc new-build -D $'FROM centos:7\nRUN yum install -y httpd'
> oc new-build https://github.com/openshift/ruby-hello-world
> --build-secret npmrc:.npmrc



