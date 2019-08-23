---
layout: post
title: "Python Multiple Version and Virtualenv with pyenv and pipenv"
subtitle: "pyenv+pipenv - Python多版本以及虚拟环境管理"
author: "Archon"
header-img: "img/header.jpg"
catalog: true
tags:
  - note
  - python
---

# env

> 这是针对Ubuntu的安装，其他Linux去[Github](#参考文档)查看安装说明

- Ubuntu

# [pyenv][1]

多版本管理

## 安装pyenv

```sh
git clone https://github.com/pyenv/pyenv.git ~/.pyenv
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
echo -e 'if command -v pyenv 1>/dev/null 2>&1; then\n  eval "$(pyenv init -)"\nfi' >> ~/.bashrc

```

如果运行有问题
> https://github.com/pyenv/pyenv/wiki/common-build-problems


## 通过pyenv安装python以及使用

```sh
# 查看可安装的python版本列表
pyenv install -l

# 安装特定版本的python
pyenv install 3.7.4

# 查看已经安装的python版本
pyenv versions

# 在当前目录使用指定版本的python
# PS: 子目录会找父目录的python版本
pyenv local 3.7.4

# 配置全局python版本
pyenv global 3.7.4
```


## 如果下载python版本过慢（选）

```sh
# 手动下载到~/.pyenv/cache
# 如: 
wget https://www.python.org/ftp/python/3.7.4/Python-3.7.4.tar.xz
mv Python-3.7.4.tar.xz ~/.pyenv/cache

pyenv install 3.7.4
```

---

# [pipenv][2]

Python虚拟环境（virtualenv）的高级API

## 安装pipenv

```sh
sudo apt install pipenv
```

## 通过pipenv创建virtualenv

```sh
# cd到目标文件夹
cd /path/prj

# 安装
# 指定python
# pipenv --python /path/python
# 如果Error: required: envname，手动创建Pipfile
# touch Pipfile
pipenv --python 3.7

# 查看当前的虚拟环境
pipenv --venv

# 使当前的虚拟环境生效
# 如有/bin/bash找不到，执行
# echo "export SHELL=/bin/bash" >> ~/.bashrc
pipenv shell

# 删除项目的virtualenv
pipenv --rm

# 安装python模块
# pipenv install -r requirements.txt
# 安装当前目录下的setup.py
# pipenv install -e .
pipenv install redis

# 用当前python环境运行命令
pipenv run pip -V
pipenv run python -V
```

---

# 参考文档

- [pyenv/pyenv][1]
- [pypa/piipenv][2]
- [The Python virtual environment with Pyenv & Pipenv][3]
- [python版本管理工具pyenv和包管理工具pipenv][4]

[1]: https://github.com/pyenv/pyenv
[2]: https://github.com/pypa/pipenv
[3]: https://dev.to/writingcode/the-python-virtual-environment-with-pyenv-pipenv-3mlo
[4]: https://www.cnblogs.com/zhangxinqi/p/9073191.html
