[TOC]

# Python3安装

安装依赖

```
# yum install openssl-devel bzip2-devel expat-devel gdbm-devel readline-devel sqlite-devel gcc gcc-c++ -y
```

下载介质

[DownLoad](https://www.python.org/ftp/python/3.6.5/Python-3.6.5.tgz)

解压安装包

```
# tar -xzvf Python-3.6.5.tgz
```

启用ssl模块(Modules/Setup)，取消注释SSL相关内容

```
# Socket module helper for socket(2)
_socket socketmodule.c timemodule.c

# Socket module helper for SSL support; you must comment out the other
# socket line above, and possibly edit the SSL variable:
SSL=/usr/local/ssl
_ssl _ssl.c \
-DUSE_SSL -I$(SSL)/include -I$(SSL)/include/openssl \
-L$(SSL)/lib -lssl -lcrypto
```

编译安装

```
# ./configure --with-ssl --prefix=/usr/local/python3.6
# make && make install -j 4
```

建立软连接

```
# ln -s /usr/local/python3.6/bin/python3 /usr/bin/python3
# ln -s /usr/local/python3.6/bin/pip3 /usr/bin/pip3
```

查看python版本

```
# python3 -V
# pip3 -V
```

