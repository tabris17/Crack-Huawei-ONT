# 破解华为光猫（ONT）的设备连接数限制

版权归作者所有，任何形式转载请联系作者。
来源：https://www.douban.com/note/721583556/

> 注意：本文中描述的方法在华为HS8546V（软件版本：V3R017C10S208）上测试通过，但无法保证适用于所有的华为ONT设备。

## 起因

移动宽带的ONT设备型号为HS8546V，在默认配置下限制最大连接10台设备，超出限制的设备虽然可以连接WIFI，但是无法上网。必须破解设备，修改配置文件才能支持更多设备。

## 一、打开路由器的Telnet功能

在浏览器中输入“http://192.168.1.1”，打开路由器的Web设置界面。路由器的IP地址是`192.168.1.1`，默认设置下，必须使用网线连接的设备才能访问Web管理后台，WIFI连接的设备是无法访问的。使用如下账号登录：

    账号： CMCCAdmin
    密码： aDm8H%MdA

![登录界面](https://img3.doubanio.com/view/note/l/public/p61819964.webp)

选择菜单“`安全`”-“`ONT访问控制配置`”，勾选“`使能LAN侧PC通过TELNET访问设备`”，点击“应用”按钮使设置生效。

![打开设备telnet接口](https://img1.doubanio.com/view/note/l/public/p61819979.webp)

现在就可以通过telnet命令来访问路由器了。如果你使用windows系统，而且没有安装telnet客户端，可以通过“`控制面板`”-“`启用或关闭windws功能`”，勾选“`Telnet Client`”来进行安装。

![安装telnet客户端](https://img3.doubanio.com/view/note/l/public/p61820235.webp)

## 二、补全Shell

现在虽然打开了设备的telnet接口，但是无法执行任何命令，因为默认配置下，所有命令都是被锁掉的。需要安装补丁来进行开启，这步操作称作“**补全Shell**”。使用这个关键字，可以在网上搜到很多信息，但是不保证这些方法都是可用的。下面介绍的方法根据我的实际操作是可行的，但是不能保证在你的设备上也是可行的。

破解需要用到以下几个工具，我已经打包上传到github了。下载地址： https://github.com/tabris17/Crack-Huawei-ONT/releases/ 

![下载 crack-huawei-ont.zip](https://img1.doubanio.com/view/note/l/public/p61822737.webp)

将下载的压缩包解压后可以找到4个文件：

![破解使用的工具](https://img3.doubanio.com/view/note/l/public/p61820393.webp)


其中“`hs8546v_shell_sp.bin`”是破解补丁；“`华为光猫配置文件加解密工具.exe`”是用来编码配置文件的工具；“`tftpd64.464.zip`”中是一个tftp服务器软件，用来向光猫设备传输补丁文件；“`ont使能.exe`”是用来刷机的工具，但是根据我实际操作，使用该工具刷机无效，最后采用tftp传输的方式，不过这个工具也一并打包了。

首先，解压缩“`tftpd64.464.zip`”。运行“`tftpd64.exe`”：

![运行 Tftp64](https://img1.doubanio.com/view/note/l/public/p61822789.webp)


在软件界面里，设置“`Current Directory`”为“`hs8546v_shell_sp.bin`”文件所在的路径；设置“`Server interfaces`”为连接路由器的网卡地址，此处为 `192.168.1.142`。具体配置信息请根据运行环境自行设置。这个软件的作用是在当前主机上运行一个tftp服务，给路由器访问。之后上传补丁文件和配置文件时都需要用到。

接着，运行 telnet 连接路由器：

![运行telnet连接路由器](https://img3.doubanio.com/view/note/l/public/p61822534.webp)


使用如下账户登录：

    Login: root
    Password: adminHW

在命令行提示下执行如下命令进行刷机操作：

    su
    shell
    load pack by tftp svrip 192.168.1.142 remotefile hs8546v_shell_sp.bin

![运行命令刷机](https://img3.doubanio.com/view/note/l/public/p61823216.webp)


请将命令中“`192.168.1.142`”修改为你的主机地址。当看到返回消息：

    Software Operation Successful!RetCode=0x0!

表明刷机成功。重启路由器后就可以使用解锁的命令了。

## 三、修改配置

由于配置文件是经过编码的，所以无法在路由器上直接编辑，需要先将配置文件下载到本地主机。用telnet连接路由器并运行以下命令：

    su
    shell
    tftp -l /mnt/jffs2/hw_ctree.xml -p 192.168.1.142

执行成功后，可以在tftpd64的”`Current Directory`“下找到”`hw_ctree.xml`“文件。运行”`华为光猫配置文件加解密工具.exe`“进行解码：

![点击”解密“按钮](https://img1.doubanio.com/view/note/l/public/p61823407.webp)


用7-zip或其他压缩软件打开刚才生成的”`hw_ctree.xml.gz`“文件，获得解码后的”`hw_ctree.xml`“配置文件：

![获得解码后的配置文件](https://img3.doubanio.com/view/note/l/public/p61823435.webp)


用文本编辑器打开解码后的”`hw_ctree.xml`“，找到”`TotalTerminalNumber`“，可以看到默认值是10。将该值改成100（也可以修改成更大的数字）后保存。

![修改配置文件中的最大连接设备数](https://img1.doubanio.com/view/note/l/public/p61823479.webp)


用7-zip或其他压缩软件将修改后的”`hw_ctree.xml`“压缩成gzip格式的压缩包：

![压缩"hw_ctree.xml“文件](https://img1.doubanio.com/view/note/l/public/p61823568.webp)


并将生成的”`hw_ctree.gz`“文件用”`华为光猫配置文件加解密工具.exe`“工具进行编码：

![点击”加密“按钮](https://img1.doubanio.com/view/note/l/public/p61823618.webp)


最后将生成的新配置文件”`hw_ctree.xml`“上传到路由器覆盖原来的配置文件。运行命令：

    su
    shell
    tftp -l /mnt/jffs2/hw_ctree.xml -r hw_ctree.xml -g 192.168.1.142

重启路由器后使配置生效。自此，你的路由器就已经可以连接100个设备了。
