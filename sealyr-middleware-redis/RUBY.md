Ruby 是一种纯粹的面向对象编程语言，由日本的松本行弘创建于1993年。

http://www.ruby-lang.org/zh_cn/

目前最新的稳定版本是 2.5.1。

## yum 安装

_该方法适用于可连接外网的服务器，安装的ruby版本较旧。_

**1. 检测是否安装：**

```
[root@centos ~]# rpm -qa | grep ruby
ruby-libs-2.0.0.648-33.el7_4.x86_64
rubygem-psych-2.0.0-33.el7_4.x86_64
ruby-irb-2.0.0.648-33.el7_4.noarch
rubygem-json-1.7.7-33.el7_4.x86_64
rubygem-io-console-0.4.2-33.el7_4.x86_64
ruby-2.0.0.648-33.el7_4.x86_64
rubygem-rdoc-4.0.0-33.el7_4.noarch
rubygem-bigdecimal-1.2.0-33.el7_4.x86_64
rubygems-2.0.14.1-33.el7_4.noarch
```

**2. 卸载已安装的ruby：**

```
[root@centos ~]# yum -y remove ruby*
```

**3. 更新yum源:**

```
[root@centos ~]# yum -y update
```

**4. 重新安装：**

```
# rubygems 是 ruby 的包管理器
yum -y install ruby rubygems
```

**5. 查看 ruby 版本：**

```
[root@centos ~]# ruby -v
ruby 2.0.0p648 (2015-12-16) [x86_64-linux]
```

**6. 查看 gem 版本：**

```
[root@centos ~]# gem -v
2.0.14.1
```

## 编译安装

**1. 准备安装包**

```
- ruby-2.5.1.tar.gz
- zlib-1.2.11.tar.gz
- openssl-1.1.0h.tar.gz
```

**2. 安装ruby-2.5.1：**

```
[root@centos ~]# tar -zxvf ruby-2.5.1.tar.gz -C /usr/local/
[root@centos ~]# cd /usr/local/ruby-2.5.1/
[root@centos ruby-2.5.1]# ./configure
[root@centos ruby-2.5.1]# make
[root@centos ruby-2.5.1]# make install
```

**查看ruby版本：**

```
[root@centos ruby-2.5.1]# ruby -v
ruby 2.5.1p57 (2018-03-29 revision 63029) [x86_64-linux]
```

查看gem版本：

```
# rubygems 是ruby的包管理器
[root@centos ruby-2.5.1]# gem -v
2.7.6
```

**3. 使用gem安装redis包：**

```
[root@centos ruby-2.5.1]# gem install redis
ERROR:  Loading command: install (LoadError)
	cannot load such file -- zlib
ERROR:  While executing gem ... (NoMethodError)
    undefined method `invoke_with_build_args' for nil:NilClass
```

报错一：安装zlib：

```
[root@centos ~]#tar -zxvf zlib-1.2.11.tar.gz -C /usr/local/
[root@centos ~]# cd /usr/local/zlib-1.2.11/
[root@centos zlib-1.2.11]# ./configure
[root@centos zlib-1.2.11]# make
[root@centos zlib-1.2.11]# make install
```

接下来，编译ruby扩展：

```
[root@centos zlib-1.2.11]# cd /usr/local/ruby-2.5.1/ext/zlib/
[root@centos zlib]# ruby extconf.rb  --with-zlib-include=/usr/local/zlib-1.2.11/include/ --with-zlib-lib=/usr/local/zlib-1.2.11/lib

# 将Makefile文件中的`$(top_srcdir)/` 全部替换为 `../../`（共1处）
[root@centos zlib]# make && make install
```

**4. 再次安装redis：**

```
[root@centos ruby-2.5.1]# gem install redis
ERROR:  While executing gem ... (Gem::Exception)
    Unable to require openssl, install OpenSSL and rebuild Ruby (preferred) or use non-HTTPS sources
```

报错二：安装openssl：

```
[root@centos ~]# tar -zxvf openssl-1.1.0h.tar.gz -C /usr/local/
[root@centos ~]# cd /usr/local/openssl-1.1.0h/
[root@centos openssl-1.1.0h]# ./config -fPIC --prefix=/usr/local/openssl enable-shared
[root@centos openssl-1.1.0h]# ./config -t
[root@centos openssl-1.1.0h]# make
[root@centos openssl-1.1.0h]# make install
```

接下来，编译ruby扩展：

```
[root@centos openssl-1.1.0h]# cd /usr/local/ruby-2.5.1/ext/openssl/
[root@centos openssl]# ruby extconf.rb  --with-openssl-include=/usr/local/openssl/include/ --with-openssl-lib=/usr/local/openssl/lib

# 将Makefile文件中的`$(top_srcdir)/` 全部替换为 `../../`（共31处，建议使用编辑工具修改）
[root@centos zlib]# make && make install
```

**5. 再次安装redis：**

```
[root@centos ruby-2.5.1]# gem install redis
ERROR:  Could not find a valid gem 'redis' (>= 0), here is why:
          Unable to download data from https://rubygems.org/ - no such name (https://rubygems.org/specs.4.8.gz)
```

更换rubygems源：https://gems.ruby-china.org/

```
# 查看源
[root@centos ~]# gem sources -l
*** CURRENT SOURCES ***

https://rubygems.org/

# 更换源
[root@centos ~]# gem sources --add https://gems.ruby-china.org/ --remove https://rubygems.org/
Error fetching https://gems.ruby-china.org/:
	no such name (https://gems.ruby-china.org/specs.4.8.gz)
```

_报错：可能是网络的原因，因此只能使用本地安装。_


- 在ruby-china上查找redis的相关版本：https://gems.ruby-china.org/gems/redis/versions
- 下载地址：https://gems.ruby-china.org/downloads/redis-4.0.1.gem

本地安装：

```
[root@centos ~]# gem install -l ./redis-4.0.1.gem
Successfully installed redis-4.0.1
Parsing documentation for redis-4.0.1
Installing ri documentation for redis-4.0.1
Done installing documentation for redis after 0 seconds
```

终于成功了!

## 参考

- https://www.cnblogs.com/xuliangxing/p/7146868.html