# openshift build

[TOC]

## Build基本介绍

- 什么是Build
- 什么是BuildConfig

## Build的基本操作

- 运行一个Build
- 取消一个Build
- 删除一个BuildConfig
- 查看构建细节
- 查看构建日志

## Build运行的规则

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

不同的策略与构建源对应的关系
 | 构建源 | 构建策略 |
 | ----- | ----- |
 | DockerFile | Docker、Custom  |
 |  |  |
 

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

> 注意1：如果构建的策略是Docker strategy，那么destinationDir的值必须是相对目录。
> 注意2：如果构建的策略是sti，目的目录如果设置，那么必须存在，因为在整个build过程中，不会创建任何新的目录。
> 注意3：如果构建的策略是sti，目前所有的secret的权限被设置成0666，就是说可以任意的读写，写的原因主要是在执行完assemble脚后，进行文件清空。对于清空的原因主要还是安全的考虑，因为assemble之后，主要就是编译运行阶段，这时的image内最初需要使用的secret，现在已经不需要了。

[DockerStragety？？？？](https://docs.openshift.org/latest/dev_guide/builds/build_inputs.html#using-secrets-docker-strategy)
[Custom Strategy ？？？？](https://docs.openshift.org/latest/dev_guide/builds/build_inputs.html#using-secrets-custom-strategy)
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

- sti
- docker
- custom
- pipeline

### STI build

首先STI是一个工具，该工具能够还是生成Docker Image
STI的主要优势：

| 优势 | 描述 |
| --- | --- | 
|镜像灵活性||
|加速制作镜像| 1.与docker build的方式相比，不需要每个step都创建一层，2.支持增量编译|
|打补丁||
|操作效率||
|操作安全||
|用户效率||
|生态||
|可再生||

两个基本概念
#### Build Process
构建的三个过程主要有三个元素组成，分别为

 - sources
 - S2I script
 -  builder image
在构建的过程中，S2I负责要把sources与S2I scripts注入到编译镜像中，为了实现这一目标，S2I通过对sources 以及S2I scripts进行打包，然后把tar文件注入到构建镜像中。在构建镜像中对该tar文件具体解压的位置由***--destination***参数或者构建镜像的***io.openshift.s2i.destination*** label指定。如果没有指定，那么默认使用/tmp目录。
同时构建的过程还需要安装tar工具以及/bin/bash，如果builder image中没有tar与/bin/bash那么sti 就会强拉一个安装该基本环境的docker image。下面是sti 构建的流图：
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

####使用images的[ONBUILD](https://docs.docker.com/engine/reference/builder/#onbuild)功能

*如果用户的builder image不含有执行S2I所需要的基本条件，即镜像中没有安装/bin/bash 或者 tar工具，那么就不会执行一个container build来对存放build的输入。但是如果builder中包含了ONBUILD指令，那么这个S2I过程就会失败。因为这就意味着执行了比较通用的容器内构件过程，这个过程相比S2I是不安全的，必须要显式的对权限进行限制。*
因此，如果用户想要顺利的执行S2I的过程，那么必须满足下面的一个条件：

 1. builder image中安装了/bin/bash 已经tar工具
 2. builder image中不能包含ONBUILD指令
 
### Docker Build
需要一个仓库地址（url）并带有一份dockerfile文件。dockerfile应该包含了所有生成可运行docker image的构件

### Custom Build

### Pipeline Build
主要完成更先进的工作流，主要完成CI/CD的过程，配置过程有两种方式：

- 在BuildConfig中植入jenkins file
- 提供一个包含jenkins file的git 库地址
[pipeline的配置过程参考](oc%20edit%20template%20jenkins-ephemeral%20-n%20openshift)
[jenkins pipeline功能参考](https://jenkins.io/doc/pipeline/)





## [Testing S2I Images](https://docs.openshift.org/latest/creating_images/s2i_testing.html#creating-images-s2i-testing)

### Overview

### Testing Requirements

### Generating Scripts and Tools

### Testing Locally

### Basic Testing Workflow

### Using OpenShift Origin Build for Automated Testing

## 调试

## 实践

### sti 工具

sti build
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

 -u, --allowed-uids user.RangeList      Specify a range of allowed user ids for the builder and runtime images
 -n, --application-name string          Specify the display name for the application (default: output image name)
      --assemble-user string             Specify the user to run assemble with
      --callback-url string              Specify a URL to invoke via HTTP POST upon build completion
      --cap-drop stringSlice             Specify a comma-separated list of capabilities to drop when running Docker containers
      --context-dir string               Specify the sub-directory inside the repository with the application sources
  -c, --copy                             Use local file system copy instead of git cloning the source url
      --description string               Specify the description of the application
  -d, --destination string               Specify a destination location for untar operation
      --dockercfg-path string            Specify the path to the Docker configuration file (default "/home/cloud/.docker/config.json")
  -e, --env string                       Specify an single environment variable in NAME=VALUE format
  -E, --environment-file string          Specify the path to the file with environment
      --exclude string                   Regular expression for selecting files from the source tree to exclude from the build, where the default excludes the '.git' directory (see https://golang.org/pkg/regexp for syntax, but note that "" will be interpreted as allow all files and exclude no files) (default "(^|/)\.git(/|$)")
      --force-pull                       DEPRECATED: Always pull the builder image even if it is present locally
      --ignore-submodules                Ignore all git submodules when cloning application repository
      --incremental                      Perform an incremental build
      --incremental-pull-policy string   Specify when to pull the previous image for incremental builds (always, never or if-not-present) (default "if-not-present")
  -i, --inject string                    Specify a directory to inject into the assemble container
  -l, --location string                  DEPRECATED: Specify a destination location for untar operation
  -p, --pull-policy string               Specify when to pull the builder image (always, never or if-not-present) (default "if-not-present")
  -q, --quiet                            Operate quietly. Suppress all non-error output.
  -r, --ref string                       Specify a ref to check-out
      --rm                               Remove the previous image during incremental builds
      --run                              Run resulting image as part of invocation of this command
  -a, --runtime-artifact string          Specify a file or directory to be copied from the builder to the runtime image
      --runtime-image string             Image that will be used as the base for the runtime image
      --save-temp-dir                    Save the temporary directory used by S2I instead of deleting it
      --scripts string                   DEPRECATED: Specify a URL for the assemble and run scripts
  -s, --scripts-url string               Specify a URL for the assemble, assemble-runtime and run scripts
      --use-config                       Store command line options to .s2ifile
  -v, --volume string                    Specify a volume to mount into the assemble container

Global Flags:
      --ca string        Set the path of the docker TLS ca file (default "/home/cloud/.docker/ca.pem")
      --cert string      Set the path of the docker TLS certificate file (default "/home/cloud/.docker/cert.pem")
      --key string       Set the path of the docker TLS key file (default "/home/cloud/.docker/key.pem")
      --loglevel int32   Set the level of log output (0-5)
      --tls              Use TLS to connect to docker; implied by --tlsverify
      --tlsverify        Use TLS to connect to docker and verify the remote
  -U, --url string       Set the url of the docker socket to use (default "unix:///var/run/docker.sock")

### oc 客户端