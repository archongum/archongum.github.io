---
layout: post
title: "WSL搭建Java开发环境"
subtitle: "Java develop enviroment for WSL 1"
author: "Archon"
header-img: "img/header.jpg"
catalog: true
tags:
  - java
  - wsl
  - linux
---

# 安装WSL Ubuntu 18.04
开启WSL并在微软应用市场安装

---

# 修改默认用户为root，并修改用户目录（选）
打开`PowerShell` <kbd>win</kbd>+<kbd>x</kbd>, <kbd>a</kbd>
```bash
ubuntu1804.exe config --default-user root
```

运行`bash`并修改
```bash
vim /etc/passwd
```

---

# 修改apt源，加快下载速度（选）
注：时间久了，源可能会不存在
```bash
# 备份
cp /etc/apt/sources.list /etc/apt/sources.list.bak

# 修改
vim /etc/apt/sources.list
# 完全代替修改为
deb http://mirrors.aliyun.com/ubuntu/ bionic main
deb http://mirrors.aliyun.com/ubuntu/ bionic-security main
deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main
```

---


# Upgrade ubuntu
```bash
apt update && apt upgrade
```

---


# Install xfce desktop

注：直接安装整个`xfce4`桌面来安装所需的图形界面组件（idea，eclipse等需要图形界面，或者自己单独安装需要的组件）
```bash
apt install xfce4
```


---



# Specify the display server
在`~/.bashrc`添加
```bash
export DISPLAY=:0.0
export LIBGL_ALWAYS_INDIRECT=1
```

---


# Install VcXsrv

Install the lastest version of [VcXsrv](https://sourceforge.net/projects/vcxsrv/).

---


# Open display server

Open **XLaunch**, choose “One large window” or “One large window without titlebar” and set the “display number” to 0.
Other settings leave as default and finish the configuration.

![](/img/2018-10-12-java-develop-enviroment-for-wsl-1/img-1.png)



---

# Run xfce desktop

运行`bash`
```bash
startxfce4
```


---

# Fix powerline fonts rendering（选）

Install the lastest version of [Hack](https://github.com/source-foundry/Hack#linux) fonts.

---


# Fix Unicode fonts rendering（选）

```bash
sudo apt-get install fonts-noto
sudo apt-get install fonts-noto-hinted
sudo apt-get install fonts-noto-mono
sudo apt-get install fonts-noto-unhinted
```

---


# Fix Chinese fonts rendering（选）

```bash
sudo apt-get install fonts-noto-cjk
```


---

# Install Chinese input method（选）

## 1.Install fcitx

```bash
sudo apt-get install fcitx
sudo apt-get install fcitx-pinyin
```

## 2.Add the following command to your bashrc file
在`~/.bashrc`添加
```bash
export XMODIFIERS=@im=fcitx
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
```


---

# 安装开发软件
## 安装Java
简单起见，直接`apt install openjdk-8-jdk`
或用[Oracle的jdk](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)

## 安装idea
[下载解压](https://www.jetbrains.com/idea/download/#section=linux)

方便启动，加入到PATH
```bash
# idea
export IDEA_HOME=/opt/idea/current
export PATH=$IDEA_HOME/bin:$PATH
```

## 安装dbeaver
[下载解压](https://dbeaver.io/download/)

方便启动，加入到PATH
```bash
# dbeaver
export DBEAVER_HOME=/opt/dbeaver/current
export PATH=$DBEAVER_HOME:$PATH

```

## 安装RedisDesktopManager
**完整版（给个star）：**[**RedisDesktopManager-Linux-WSL**](https://github.com/archongum/RedisDesktopManager-Linux-WSL)

```bash
# need to upgrade
apt update && apt upgrade

# change to latest tag
git clone --recursive https://github.com/uglide/RedisDesktopManager.git -b 0.9.8

cd RedisDesktopManager/
cd src/
./configure
qmake
make
make install

cd /opt/redis-desktop-manager/
mv qt.conf qt.backup

# optional, reduces rdm size 30MB+ to 2MB+
strip rdm
```
方便启动，加入到PATH
```bash
# RDM
export RDM_HOME=/opt/redis-desktop-manager
export PATH=$RDM_HOME:$PATH
```


---

# 其他
## 汇总需要添加到.bashrc
```bash
export DISPLAY=:0.0
export LIBGL_ALWAYS_INDIRECT=1

#pinyin
export XMODIFIERS=@im=fcitx
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx

# dbeaver
export DBEAVER_HOME=/opt/dbeaver/current
export PATH=$DBEAVER_HOME:$PATH

# idea
export IDEA_HOME=/opt/idea/current
export PATH=$IDEA_HOME/bin:$PATH

# RDM
export RDM_HOME=/opt/redis-desktop-manager
export PATH=$RDM_HOME:$PATH

```

## Git修改信息和记住密码
```bash
git config --global user.name archon
git config --global user.email qq349074225@live.com
git config --global credential.helper store
```

## 安装字体
1. 复制字体到`/usr/share/fonts/${自己新建的目录}`
2. 运行`fc-cache -f -v`
3. 参考[link1](https://github.com/source-foundry/Hack#linux), [link2](https://blog.csdn.net/bitcarmanlee/article/details/79729634)



---

# 参考文档
[RedisDesktopManager-Linux-WSL](https://github.com/archongum/RedisDesktopManager-Linux-WSL)
[wsl-tutorial](https://github.com/QMonkey/wsl-tutorial)
[redisdesktop linux](http://docs.redisdesktop.com/en/latest/install/#ubuntu-debian-fedora-centos-opensuse-archlinux-other-linux-di)
[redisdesktop build fail](https://github.com/uglide/RedisDesktopManager/issues/4118)


