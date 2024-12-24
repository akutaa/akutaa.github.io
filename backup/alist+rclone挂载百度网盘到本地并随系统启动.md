
由于开始摄影，不得不考虑数据备份问题。照片文件是若干10~50M的小文件，且生产出来之后基本只需要冷备份，所以也不怎么需要版本管理之类的功能。目前文档都用15G空间的免费OneDrive同步，体验虽说不佳但也还能用，主要是很喜欢“不需要离开文件资源管理器就能备份”的操作。所以我对于照片备份的想法是，不需要定时备份，只要每次导入照片之后直接一拖上传到备份盘就足矣。

本着这个思路，一开始考虑的是组NAS一类自建服务。不过由于肉眼可见的逐渐折腾不动，以及居住空间的不安定性~~指经常搬家~~，最终决定返璞归真，每年花150买百度网盘SVIP，然后挂载本地。至于为什么买百度，也只能说这是国内网盘的最优解了~~怎么，你不服气吗？~~。

首先使用的是alist+raiDrive的方案，但出现了取回文件提示没权限的问题。加上新版本的raiDrive有弹窗，遂弃。随后换成了闭源的CloudDrive2，一切都很好，但是取回文件到最后一部一定卡住。AirLiveDrive等也试过了，都有问题。最终用了alist+rclone的方案。

大部分内容参考了[这个帖子](https://blog.csdn.net/qq_16747625/article/details/138796829)，但过程中也是踩了几个坑。

# 配置alist开启WebDAV

参考alist v3的[文档](https://alist.nn.ci/zh/guide/drivers/baidu.html)，挂载百度网盘即可。里面针对各种策略做了详细的解释。因为我是SVIP且希望尽量稳定，所以选择官方接口，WebDAV策略302重定向。需要注意的是，这里的挂载路径可以随意填写，作为挂载之后的根路径。如这里写的是“百度网盘”。

![alt text](assets/image-13.png)

# 配置rclone

[rclone](https://github.com/rclone/rclone/releases)依赖[WinFsp](https://github.com/winfsp/winfsp/releases)，可以事先安装。

在rclone.exe所在路径下打开命令行，执行指令`rclone.exe config`。注意可能要使用`./rclone.exe config`的语法，注意命令行的提示。当然，也可以直接注册rclone为环境变量。前面的alist同理。

配置`New remote`->输入自定义的名称（如baidu）->选择WebDAV->输入地址`http://127.0.0.1:5244/dav/百度网盘/`->选择`Other site/service or software`->输入用户名（即AList配置的用户名）->输入密码（AList配置的密码）->回车跳过`Option bearer_token`配置->`Edit advanced config?`选择No->选择y保存配置->配置完成。

运行指令`rclone.exe lsd baidu:`，如果能正确输出目录则配置成功。

需要注意的是，`rclone.exe lsd`命令后跟的是rclone中配置的remote名称，而不是alist中的网盘挂载路径（尽管两者可以相同，你也可以将remote取名为“百度网盘”）。

# 配置nssm实现开机自动挂载

nssm可以将exe注册为系统服务。将alist.exe和rclone.exe注册为系统服务后，可以实现优雅的开机自动挂载。当然也可以用脚本实现。

下载[nssm](https://nssm.cc/download)后，在win64路径下~~大家应该都是64位系统了吧~~运行命令`nssm.exe install alist`打开nssm GUI。这里依然可以参考[这个帖子](https://blog.csdn.net/qq_16747625/article/details/138796829)。记得在`I/O`选项卡填log路径，对排障很有用。

接着`nssm.exe install rclone`注册rclone。值得注意的是这里的`Arguments`，我填写的是：
```
mount baidu:/ W: --cache-dir D:/Cache/ --vfs-cache-mode full --vfs-cache-max-size 10G --vfs-disk-space-total-size 12T --header "Referer:https://pan.baidu.com/" --header "User-Agent:pan.baidu.com" --config="C:\Users\<用户名>\AppData\Roaming\rclone\rclone.conf"
```

这里`baidu:`是rclone中remote的名称；`W:`是拟挂载为的盘符；`D:/Cache/`是缓存路径（建议指定在系统盘之外速度较快的SSD上）；`10G`是VFS缓存大小，可以自己指定（关于VFS缓存机制可参见[这篇文章](https://qjfyx.com/web/rcvfscd2.html)）；`12T`是我的网盘总大小，可以自己根据情况指定；`header`和`User-Agent`参数是因为百度网盘的策略，对20M以上文件的请求必须由这个请求头发起；`config`是为了指定rclone配置文件所在位置。一开始不带这个参数，遇到了问题，查看log发现了如下的信息：

```
2024/12/23 21:54:30 NOTICE: Config file "C:\\Windows\\system32\\config\\systemprofile\\AppData\\Roaming\\rclone\\rclone.conf" not found - using defaults
2024/12/23 21:54:30 CRITICAL: Failed to create file system for "baidu:/": didn't find section in config file
```

问题原因是被注册为system服务后，rclone会去system路径下找配置文件，然而没有权限。这似乎是因为我在使用nssm之前手动运行了`mount`命令，配置文件被放在了用户路径下。加了`config`参数指定之后就正常了。

之后记得在`Dependencies`选项卡将前面注册的alist作为依赖。

完成后，应该就可以在系统服务管理界面看到这两个服务，可以暂停或设置自动启动。系统启动后会自动挂载。唯一的问题是不能显示已用空间，不过空间不吃紧的情况下问题不大。

![alt text](assets/image-14.png)