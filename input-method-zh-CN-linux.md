# Ubuntu 下安装 fcitx 中文输入法
1. 更新源
`sudo apt-get update`
2. 安装
`sudo apt-get install fcitx fcitx-config-gtk fcitx-sunpinyin fcitx-googlepinyin fcitx-module-cloudpinyin im-switch`
如果要安装 fcitx-sogoupinyin，需要另外添加 ppa。
3. 配置
`sudo im-switch -s fcitx -z default`
4. 在英文 Locale 下启动 fcitx 输入法，可能出现错误：
```
No system wide default defined just for locale en_US .
Use "all_ALL" quasi-locale and set IM.
update-alternatives: error: alternative /etc/X11/xinit/xinput.d/fcitx for xinput-all_ALL not registered, not setting.
```
解决方法：
 1. 在 /etc/X11/xinit/xinput.d/ 下新建一个文件 en_US：
`sudo vim /etc/X11/xinit/xinput.d/en_US`
文件内容如下：
```
XMODIFIERS="@im=fcitx"
XIM=fcitx
XIM_PROGRAM=/usr/bin/fcitx
XIM_ARGS=""
GTK_IM_MODULE=XIM
QT_IM_MODULE=XIM
DEPENDS="fcitx"
```
 2. `sudo update-alternatives --install /etc/X11/xinit/xinput.d/all_ALL xinput-all_ALL /etc/X11/xinit/xinput.d/fcitx 30`
 3. 重新执行
`sudo im-switch -s fcitx -z default`
5. 重启
