layout: post
categories: php
title: 使用PhpStorm+xdebug远程调试php
tags: [php, xdebug, phpstorm]
date: 2015/12/16
---

## 软件配置
| 软件 | 版本 |
| ---- | ---- |
| mac os | 10.11.2 |
| nginx| 1.6.3 |  
| php-fpm | |
| xdebug | 2.4.0rc3 |
| php | 5.6.7 |

## 安装xdebug
1. 从[xdebug](http://xdebug.org/download.php)官网下载xdebug的源码
2. 解压`tar -xvzf xdebug-2.4.0rc3.tgz`
3. `cd xdebug-2.4.0RC3`
4. `./configure`
5. `make`
6. `cp module/xdebug.so <any/path/you/want>`

## 加载xdebug
修改php.ini文件，新增以下配置后重启php-fpm

```
[Xdebug]
zend_extension=<xdebug.so所在的位置>           # 加载xdebug动态库，注意是zend_extension 
xdebug.remote_enable=1                       # 打开xdebug
xdebug.remote_autostart=1                    # 打开自动监听，否则需要每次调试时在url中手动添加该参数
xdebug.remote_port=<xdebug监听端口(默认9000)>  # php-fpm会默认监听9000端口，所以这地方需要换一个端口
```
## PhpStorm设置
1. 到`Preferences->Languages & Framworks->PHP`中配置解释器，如果没有先添加一个，xdebug正确加载会显示在右下角，否则说明xdebug没有正确配置
	
	![config interpreter](config-interpreter.png)
	
	![add interpreter](add-interpreter.png)
2. 到`Preferences->Languages & Framworks->PHP->Debug`中配置xdebug的监听端口，需要同`xdebug.remote_port`设置的一致
3. 使用`Run->Web Server Debug Validation`检查debug的配置
	
	![validate-config](validate-config.png)
4. 最后打开监听，添加断点就可以调试了
	
	![enable-listen](enable-listen.png)
	![stop-at-breakpoint](stop-at-breakpoint.png)

## 参考资料
[Configuring Xdebug](https://www.jetbrains.com/phpstorm/help/configuring-xdebug.html#updatingPhpIni)
