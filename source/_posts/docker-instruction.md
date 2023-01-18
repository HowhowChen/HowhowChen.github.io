---
title: Docker超入門
date: 2023-01-16 12:54:06
tags:
  - [軟體開發,工具]
  - [容器]
categories:
  - [軟體開發, 工具]
  - 容器
---
## 連阿嬤都學得會的Docker

![](//upload.wikimedia.org/wikipedia/commons/thumb/4/4e/Docker_%28container_engine%29_logo.svg/610px-Docker_%28container_engine%29_logo.svg.png)

<!-- more -->

### 什麼是Docker?與虛擬機差異?
Docker的容器，可以想像是一個箱子(Box)，每個容器就是一個箱子，箱子內裝的是一台電腦，電腦會有一個應用程式在執行，而且電腦擁有自己的本機名稱、IP地址、硬碟空間。其CPU、記憶體、作業系統與本機端共用。



### 運行環境
Docker必須運作在linux虛擬機上，所以讀者使用Windows、Macos作業系統，必須安裝一個微型的虛擬機才能使用docker。筆者工作電腦是windows作業系統，使用vmware安裝一個ubuntu的虛擬機，在linux虛擬機上運作docker。以下筆者會以linux虛擬機操作docker，其餘方式請參考官網。

![](https://hemanta.io/static/858eedfeddf5d1700429223d43cd09c3/a27c6/docker-desktop.png)

### 環境搭建
1. 下載虛擬機
    Windows[vmware](https://www.vmware.com/latam/products/workstation-pro/workstation-pro-evaluation.html)
    Windows/MacOS [VirtualBox](https://www.virtualbox.org/wiki/Mac%20OS%20X%20build%20instructions)

2. 下載linux Ubuntu發行版作業系統https://ubuntu.com/download/desktop

3. 安裝Linux 虛擬機

### docker快速開始
1. 安裝docker

```shell
sudo apt install docker.io
```
2. 建立docker group以利後續不必用root操作docker

````shell
sudo groupadd docker
sudo usermod -aG $USER
newgrp docker
````
3. 確認版本
```
docker version
```
![](https://i.imgur.com/zo6axto.png)

4. 運行hello- world container
```shell
docker run hello-world
```
![](https://i.imgur.com/UwGh1cG.png)