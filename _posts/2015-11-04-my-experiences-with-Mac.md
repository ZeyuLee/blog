---
title: "mac选购、开发环境搭建及常用软件推荐"
layout: single
categories: Experience
tags: mac guide develop software
---

自己购买并使用MacBook Pro也有一段时间了，不论开会写代码都带着，身边有不少开发的小伙伴表示忍受不了信仰灯的诱惑，想趁着双11狠狠的剁一次手。恰逢15款发布，结合我浅显的使用体验，谈谈自己的感受。

## 选购

Mac定位是一款生产工具（想买来打游戏的还是看看外星人吧），对于一个会陪伴你3年以上，为你创造价值的武器，我个人觉得：在你能力范围内买最好的。

考虑到我没有视频、游戏和双系统的需要，综合便携性，我自己购买的是13寸的中配版本MacBook Pro（以下简称MBP）。MBP是全球联保的产品，1年内提供电话支持和维修服务。考虑到苹果产品维修价格过高，和其3年以上使用寿命，无脑推荐在购买MBP后再搭配Apple Care服务，可以延长2年的保修，淘宝价格在1100左右，选择可以邮寄序列号及手册的实体版本，不差钱的可以去实体店购买。

15款MacBook激进的使用了目前还不太普及的Type-C接口，又失去了信仰灯的加成，我个人完全不推荐，更详细的可以看Zealer的[评测](http://www.zealer.com/post/174.html)。

## 准备工作

在拿到机器，开始搭建开发环境 or 正式使用之前，请做以下两件事

1. 准备一块500G－1T的移动硬盘，苹果系统内置了时光机（Time Machine），支持定时自动备份，如果在使用中出现重大系统问题、笔记本丢失、换新的笔记本，可以非常方便的恢复系统，做到完全的平滑回滚，我用了1年大概使用了600G空间。

2. 升级操作系统：从我1年的使用经验来看，操作系统升级非常平滑，而且新版一般都有很多不错的功能，使用体验也变化不大。如果你升级系统出现崩溃，请用上面提到的时光机进行恢复，然后去买张彩票吧。

## 必备工具

1. 番羽土啬工具 [ShadowSocks](https://github.com/shadowsocks/shadowsocks-iOS/releases) + 账号，没有双币信用卡、不会搭建vps或嫌麻烦的，可以试试[枫叶主机](http://www.fyzhuji.com/aff.php?aff=1409)
2. 终端工具 [iterm2](https://www.iterm2.com/downloads.html)

## 搭建环境

### 终端工具

```bash
#切换zsh为默认shell
chsh -s /bin/zsh
#完全关闭iTerm后，确认切换是否成功
echo $SHELL
#输出 /bin/zsh 就对了
```

安装 [oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh) 插件

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

编辑配置文件，选择一个你喜欢的[主题](https://github.com/robbyrussell/oh-my-zsh/wiki/themes)，开启你需要的[扩展](https://github.com/robbyrussell/oh-my-zsh/wiki/plugins)

```bash
#oh-my-zsh配置文件
vi ~/.zshrc
#扩展配置 plugins=(svn git brew composer)
#主题配置 ZSH_THEME=random 不太确定哪个更适合你的，可以设置为随机
#所有的修改将在新开的iTerm标签生效
```

### 包管理器工具

homebrew是Mac平台下非常棒的包管理工具，对于快速搭建开发环境非常有帮助。我个人在安装前，一般都会先看看homebrew上有没有，个别情况下会使用源码编译安装。

homebrew安装的包默认在 /usr/local/Cellar/ ，在 /usr/local/opt/ 中通过软链接指向程序安装目录，配置文件一般在 /usr/local/etc/ 。

```bash
#以下所有brew操作最好配合番羽土啬工具使用，否则可能会出现下载失败的情况
#使用方法：开启shadowsocks并配置后，在需要执行的brew命令前，添加ALL_PROXY=socks5://127.0.0.1:1080

#安装nginx
brew install nginx

#从El Capitain开始，1000以下端口必须以root执行，不带sudo让nginx监听80端口需要调整nginx权限
chown root:wheel /usr/local/opt/nginx/bin/nginx
chmod u+s /usr/local/opt/nginx/bin/nginx
#调整nginx默认端口，改listen值为80，默认为8080端口
vi /usr/local/etc/nginx/nginx.conf

#添加自启动，并开启服务
cp /usr/local/opt/nginx/homebrew.mxcl.nginx.plist ~/Library/LaunchAgents
launchctl load ~/Library/LaunchAgents/homebrew.mxcl.nginx.plist
#检查nginx执行是否正常，有1行以上的结果基本就正常了
ps aux | grep nginx

#安装php及扩展
brew install homebrew/php/php56 homebrew/php/php56-igbinary homebrew/php/php56-mcrypt homebrew/php/php56-memcached homebrew/php/php56-msgpack homebrew/php/php56-pdo-dblib homebrew/php/php56-redis homebrew/php/php56-xhprof homebrew/php/php56-yaf homebrew/php/composer

#添加自启动，并开启服务
cp /usr/local/opt/php56/homebrew.mxcl.php56.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/homebrew.mxcl.php56.plist
#检查php执行是否正常
ps aux | grep php

#配置站点（省略）
#配置完成后可以执行 nginx -t 检查配置文件是否正确，nginx -s reload 重启nginx

#配置本地硬解，跟windows差不多
sudo vi /etc/host

#写个phpinfo() 检查nginx & php合体是否正确
vi /usr/local/opt/nginx/html/index.php
```

用homebrew管理软件

```bash
#更新brew并更新版本库
brew update
#查看过期的软件
brew outdated
#升级单个
brew upgrade xxxx
#升级全部
brew upgrade
#清理过期版本（确认新版本正常后再执行）
brew cleanup
```

### 配置环境变量

```bash
#添加到zsh
echo 'export PATH="$(brew --prefix php56)/sbin:$PATH"' >> ~/.zshrc
echo 'export PATH="/usr/local/sbin:$PATH"' >> ~/.zshrc
 
#如果你不打算用zsh，请添加到系统默认的bash
echo 'export PATH="$(brew --prefix php56)/sbin:$PATH"' >> ~/.bash_profile
echo 'export PATH="/usr/local/sbin:$PATH"' >> ~/.bash_profile
```

### 常用软件

Mac平台也有类似苹果手机的app store，一般称呼为Mac App Store（以下简称MAS）。由于苹果严格的审核政策和抢钱一样的分成策略，很多优秀的软件都没有上架MAS，即使上架到MAS的软件，其更新的实时性上也要差很多。如果能去官网下载，最好去官网。

除了QQ、微信、Evernote等这些主流免费应用外，以下列出的软件推荐在官网上下载：[Alfred](https://www.alfredapp.com/)、[FileZilla](https://filezilla-project.org/)、[Dash](https://kapeli.com/dash)、[Sequel Pro](http://www.sequelpro.com/)

剩下的自由发挥吧，这里有个[榜单](https://github.com/hzlzh/Best-App)可以参考，另外MAS也有限免和冰点价格，请参考[网站](http://appshopper.com/)，碰到不错的价格就入手吧。


## 2017年4月补充

### 选购

2016款有带Touchbar的版本了，目前只有少数应用做了兼容，作为开发人员，没有实体Esc键个人觉得是最大的痛苦。要不要Touchbar的版本以及传说中的渣手感键盘，建议先去实体店体验一下，这里做一个简单对比。

| 版本 | 价格 | 接口 | 续航 |
| --- | --- | --- | --- |
| 有 touchbar | 高 | 4 Type-C | 稍差 |
| 无 touchbar | 低 | 2 Type-C | 稍好 |

### homebrew

```bash
#服务管理
brew services list
brew services start zookeeper
brew services stop zookeeper
brew services restart zookeeper

#brew case管理三方软件
brew cask search firefox
brew cask install java
brew cask uninstall chrome
```

### 软件

上面说的ss已经好久不更新了，推荐一个新的工具 [ShadowsocksX-NG](https://github.com/shadowsocks/ShadowsocksX-NG/releases/)

这一年多自己安装软件，基本与之前推荐的榜单使用感受相同。

* [Alfred PowerPack](https://www.alfredapp.com/powerpack/buy/) （workflow缺少官方源和类似brew的管理方式，目前只发现这一个缺点）
* markdown编辑器 [MacDown](https://macdown.uranusjr.com/)
* Office全家桶 [Office365](https://www.microsoftstore.com.cn/office/office-365-personal/p/qq2-00000)
* 解压工具 [Keka](http://www.kekaosx.com/zh-cn/)
* 密码管理软件 [1Password](https://itunes.apple.com/cn/app/1password/id568903335?mt=8)
* 邮件客户端 [ThunderBird](https://www.mozilla.org/zh-CN/thunderbird/) （以前windows用的就是这个，平滑迁移）
* 系统监控 [iStat Menus](https://bjango.com/mac/istatmenus/)
* 鼠标Tips工具 [PopClip](https://itunes.apple.com/cn/app/popclip/id445189367?mt=12)
* 工具栏管理 [Bartender 2](https://www.macbartender.com/)