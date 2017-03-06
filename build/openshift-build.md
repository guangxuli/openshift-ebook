# openshift build

[TOC]

## sti
主要的作用还是生成Docker Image
两个基本概念
### Build Process
构建的三个过程主要有三个元素组成，分别为

 - sources
 - S2I script
 -  builder image
在构建的过程中，S2I负责要把sources与S2I scripts注入到编译镜像中，为了实现这一目标，S2I通过对sources 以及S2I scripts进行打包，然后把tar文件注入到构建镜像中。在构建镜像中对该tar文件具体解压的位置由***--destination***参数或者构建镜像的***io.openshift.s2i.destination*** label指定。如果没有指定，那么默认使用/tmp目录。
同时构建的过程还需要安装tar工具以及/bin/bash，如果builder image中没有tar与/bin/bash那么sti 就会强拉一个安装该基本环境的docker image。下面是sti 构建的流图：
![sti flow](https://github.com/openshift/source-to-image/blob/master/docs/sti-flow.png)

###[sources](https://docs.openshift.org/latest/dev_guide/builds/build_inputs.html#how-build-inputs-work)
 - Inline Dockerfile definitions
 - Content extracted from existing images
 - Git repositories
 - Binary inputs
 - Input secrets
 - External artifacts

###S2I scripts
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

###使用images的[ONBUILD](https://docs.docker.com/engine/reference/builder/#onbuild)功能

*如果用户的builder image不含有执行S2I所需要的基本条件，即镜像中没有安装/bin/bash 或者 tar工具，那么就不会执行一个container build来对存放build的输入。但是如果builder中包含了ONBUILD指令，那么这个S2I过程就会失败。因为这就意味着执行了比较通用的容器内构件过程，这个过程相比S2I是不安全的，必须要显式的对权限进行限制。*
因此，如果用户想要顺利的执行S2I的过程，那么必须满足下面的一个条件：

 1. builder image中安装了/bin/bash 已经tar工具
 2. builder image中不能包含ONBUILD指令

## pipeline 


## docker file


## custom


----------


----------


----------


----------

