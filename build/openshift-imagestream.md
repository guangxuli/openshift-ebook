
#OpenShift-ImageStream
[TOC]
##什么是ImageStream？
简单理解为OpenShift 系统的一个新对象类型，类似BC（BuildConfig），主要提供了镜像的管理功能，保存着整个project下所有相关镜像的详细信息。
## 为什么需要，主要的使用背景是什么？

- 便于自动化CI/CD过程，CI的build与CD的deploy可以watch imagestream的变化，从而完成自动化的构建或者部署流程
-  同时也解耦了业务代码与镜像仓库的直接通信，便于管理