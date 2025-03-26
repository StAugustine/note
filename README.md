# 笔记积累

收集、归纳、分享



1.上传相关rpm包：

wkhtmltox-0.12.6-1.centos7.x86_64.rpm

（wkhtmltox-0.12.6-1.centos8.x86_64.rpm存在问题，执行命令的时候会提示）

2。安装相关依赖：

到/usr/lib64目录下执行命令： dnf install libpng15

wget https://vault.centos.org/centos/8/AppStream/x86_64/os/Packages/compat-openssl10-1.0.2o-3.el8.x86_64.rpm

rpm -ivh compat-openssl10-1.0.2o-3.el8.x86_64.rpm

yum install -y fontconfig libX11 libXext libXrender libjpeg libpng xorg-x11-fonts-75dpi xorg-x11-fonts-Type1

3。最后执行命令：rpm -ivh wkhtmltox-0.12.6-1.centos7.x86_64.rpm



4。执行命令 wkhtmltoimage -version 如果没有error报错表示安装成功


https://blog.csdn.net/qq_31386371/article/details/143424240