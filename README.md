# openwrt 官方源码编译简明教程

## 编译环境要求

- linux系统：ubuntu 22.04.05（可通过vmware虚拟机软件安装系统）
- linux下可科学上网
- linux终端中用普通用户编译

#### linux下科学上网的两种方法
- 使用http、https网络代理（主机有v2rayn等代理客户端）：主机代理客户端打开允许局域网连接；ubuntu网络设置中使用网络代理，ip为主机ip地址，端口号在代理客户端中查看

- 适用于ubuntu/Debian的代理客户端 clash verge：https://www.kjfx.cc/524.html

-------------------

## 编译流程

### 安装、更新编译依赖包
分别执行下列命令

    sudo apt update -y

    sudo apt full-upgrade -y

    sudo apt install -y ack antlr3 asciidoc autoconf automake autopoint binutils bison build-essential \
    bzip2 ccache cmake cpio curl device-tree-compiler fastjar flex gawk gettext gcc-multilib g++-multilib \
    git gperf haveged help2man intltool libc6-dev-i386 libelf-dev libglib2.0-dev libgmp3-dev libltdl-dev \
    libmpc-dev libmpfr-dev libncurses5-dev libncursesw5-dev libreadline-dev libssl-dev libtool lrzsz \
    mkisofs msmtp nano ninja-build p7zip p7zip-full patch pkgconf python2.7 python3 python3-pyelftools \
    libpython3-dev qemu-utils rsync scons squashfs-tools subversion swig texinfo uglifyjs upx-ucl unzip \vim wget xmlto xxd zlib1g-dev

### 克隆官方源码
    git clone https://github.com/openwrt/openwrt

该步骤会生成openwrt的文件夹，进入文件夹，并获取版本信息

    cd openwrt

    git tag

按enter键拉到最新版本，确认需要的版本，一般选择最新版本。建议去官网查询自己硬件支持的最新版本：
https://openwrt.org/toh/views/toh_fwdownload?dataflt%5B0%5D=supported%20current%20rel_%3D23.05.2

目前，2024年10月26日，官方最新版是23.05.5，根据我自己的经验，x86_64选最新版有一些插件兼容性问题，我的路由器红米AC2100官方目前支持到23.05.4，接下来以23.05.4为例编译

切换版本

    git checkout v23.05.4

### 添加第三方插件源
命令行方式添加：

    echo "src-git op_pkg https://github.com/Hino-111/op_pkg.git" >> "feeds.conf.default"

也可以在openwrt文件夹下的 feeds.conf.default 文件中手动按照一定格式添加

对于自己想要编译的插件，可以上网搜索插件的github源码下载下来再上传到自己的github仓库，统一集成到自己的github仓库中，记得点星支持原作者。若是自己创建的github私库，需要在主页-设置-开发者工具-创建个人访问令牌：勾选私库管理、下载等选项，生成密钥，复制密钥；后续更新feeds时需要输入github用户名和密钥

更新feeds：

    ./scripts/feeds update -a

文件传输过程有时候文件的权限会发生改变，对于某些插件没有执行权限，后续编译会报错。赋予feeds文件夹执行权限：

    chmod -R 755 feeds

安装feeds的插件到package，编译会编译package中的插件：

    ./scripts/feeds install -a

某些插件需要go 1.22支持，升级go版本：

    rm -rf feeds/packages/lang/golang && git clone https://github.com/sbwml/packages_lang_golang -b 22.x feeds/packages/lang/golang

### 一次编译

运行配置面板：

    make menuconfig

方向键选择，enter确认，空格键切换选项；*表示选择并编译
- 配置cpu架构、类型
- 配置编译生成镜像的文件类型、磁盘大小
- base-system中取消dnsmag，勾选dnsmag-full（某些插件需要）
- luci>collection:勾选luci（web登录面板）
- luci>module：勾选luci-compat；进入translate，选择面板语言
- luci>application：勾选必须的插件（不怎么需要的先不选）
- luci>proto：勾选wireguard，有开VPN实现连接内网nas的smb文件共 享的需求勾选，否则忽略

最后保存并退出

下载编译工具：

    make download -j$(nproc) V=s

建议多下几次，末尾没有error出现，过程中没有download failed出现，不要手动打断过程导致文件下载一半的情况出现（ctrl+c），如果网速十分慢难以等待，建议切断网络或代理让其下载失败，更换网络环境再尝试。

编译（一般2~3小时，网络没问题的话）：

    make -j$(nproc) V=s

j$(nproc)表示最大cpu线程编译，优点速度快，缺点出错后报错信息无法定位。
如果出现错误：

    make  V=s

单线程编译，速度慢，定位到错误信息进行搜索排查。
建议先全线程编译，运气好一次过了，即使报错后，已经编译了很多内容，再用单线程编译不至于太慢。如果一开始用单线程编译，可能要等5、6个小时。
编译出错千千万，网络问题占一半，出错先看是否有下载失败，超时，ssl等问题，一般是网络原因，其他的复制出错信息上网搜索。

### 二次编译

二次编译目的是减少某些插件与必需插件产生冲突，如果不兼容，舍弃非必需插件，或者寻找替代

方式一：单独编译.ipk包：
勾选要添加的插件，只增不减；如果要取消已勾选的插件，需要清楚之前的配置（rm -rf ./tmp && rm -rf .config），再重新配置

    make menuconfig

在openwrt>package中找到要编译的插件包，如luci-app-homeproxy：

    make package/feeds/op_pkg/luci-app-mihomo/compile V=s

方式二：直接编译到镜像：
先清除一次编译生成的镜像，不会删除编译工具，因此会比一次编译快。

    make clean

勾选要添加的插件，只增不减；如果要取消已勾选的插件，需要清楚之前的配置（rm -rf ./tmp && rm -rf .config），再重新配置

    make menuconfig

下载、编译：

    make download -j$(nproc) V=s

    make -j$(nproc) V=s

### 下载镜像和ipk包到主机

生成的镜像文件在openwrt>bin>targets...下；生成的ipk文件在openwrt>bin>packages...下。

#### 打开ubuntu ssh 登录
新打开一个终端

    sudo -i

    sudo apt-get update

    sudo apt-get upgrade

    apt-get install ssh

    sudo /etc/init.d/ssh start

设置root登陆密码

    passwd root

开启密码验证以及重启ssh服务

    sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin yes/g' /etc/ssh/sshd_config

    sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication yes/g' /etc/ssh/sshd_config

    service sshd restart

#### 通过wondows powershell下载
根据自己实际的ubuntu ip地址和文件路径修改，下载单个文件到主机

    scp root@192.168.1.167:/openwrt/bin/targets/.../openwrt-x86-64-generic-ext4-combined.img.gz C:/myfile

下载整个package文件夹到主机：

    scp -r root@192.168.1.167:/openwrt/bin/package C:/myfile

将命令中的路径位置颠倒可以将主机文件上传到服务器

#### 通过ssh软件如finalshell下载

首先下载finalshell软件，并通过ssh登录到服务器，在下方文件区直接右键鼠标进行下载

-----------------------

## openwrt插件推荐

- luci-app-ddns-go（第三方）：动态域名解析，好用
- luci-app-acme（官方）：用于域名tls证书申请，实现https登录等
- wireguard（官方）：一种VPN协议，一种安全实现公网访问内网的方式之一
- luci-app-homeproxy（第三方）：sing-box内核，类似ssr-plus，简洁易用稳定
- adguardhome（官方）：去广告等
- luci-app-autoreboot（第三方）：一个自动重启openwrt的插件，看需求；红米AC2100不适用

