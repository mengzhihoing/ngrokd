# ngrokd
来源：http://sunnyos.com/article-show-48.html
准备vps，域名，ping域名会显示vps的ip

安装git，我安装的是2.6版本，防止会出现另一个错误，安装git所需要的依赖包

<pre>
yum -y install zlib-devel openssl-devel perl hg cpio expat-devel gettext-devel curl curl-devel perl-ExtUtils-MakeMaker hg wget gcc gcc-c++
</pre>

2、下载git
<pre>
wget https://www.kernel.org/pub/software/scm/git/git-2.6.0.tar.gz
</pre>

3、解压git
<pre>
tar zxvf git-2.6.0.tar.gz
</pre>

4、编译git
<pre>
cd git-2.6.0
./configure --prefix=/usr/local/git
make
make install
</pre>

5、创建git的软连接
<pre>
ln -s /usr/local/git/bin/* /usr/bin/
</pre>

安装go环境
准备go环境，我的系统是32位的centos所以我下载386的包
1、下载go的软件包（原文的链接失效，已更新，64位用下面的链接）
<pre>
wget http://www.golangtc.com/static/go/1.4.2/go1.4.2.linux-386.tar.gz
#wget http://www.golangtc.com/static/go/1.4.2/go1.4.2.linux-amd64.tar.gz(64位)
</pre>

2、解压出来可以随便指定位置
<pre>
tar -zxvf go1.4.2.linux-386.tar.gz
mv go /usr/local/
</pre>

3、go的命令需要做软连接到/usr/bin
<pre>
ln -s /usr/local/go/bin/* /usr/bin/
</pre>
编译ngrok
文本模式复制代码，把ngrok.sunnyos.com改成自己的域名
<pre>
cd /usr/local/
git clone https://github.com/inconshreveable/ngrok.git
export GOPATH=/usr/local/ngrok/
export NGROK_DOMAIN="ngrok.sunnyos.com"
cd ngrok
</pre>

为域名生成证书
<pre>
openssl genrsa -out rootCA.key 2048
openssl req -x509 -new -nodes -key rootCA.key -subj "/CN=$NGROK_DOMAIN" -days 5000 -out rootCA.pem
openssl genrsa -out server.key 2048
openssl req -new -key server.key -subj "/CN=$NGROK_DOMAIN" -out server.csr
openssl x509 -req -in server.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out server.crt -days 5000
</pre>
在软件源代码目录下面会生成一些证书文件，我们需要把这些文件拷贝到指定位置(这步问你是否覆盖，输入y)
<pre>
cp rootCA.pem assets/client/tls/ngrokroot.crt
cp server.crt assets/server/tls/snakeoil.crt
cp server.key assets/server/tls/snakeoil.key
</pre>
如果是在天朝的服务器需要改，香港或者国外的服务器不需要(本人服务器在美国，略过)
<pre>
vim /usr/local/ngrok/src/ngrok/log/logger.go
log "github.com/keepeye/log4go"
</pre>
指定编译环境变量，如何确认GOOS和GOARCH，可以通过go env来查看

编译服务端
<pre>
cd /usr/local/go/src
GOOS=linux GOARCH=386 ./make.bash
cd /usr/local/ngrok/
GOOS=linux GOARCH=386 make release-server
</pre>
编译客户端，我的是mac os 64位操作系统，所以我的是下面的命令（客户端文件路径服务器的/usr/local/ngrok/bin/下面，）
<pre>
cd /usr/local/go/src
GOOS=darwin GOARCH=amd64 ./make.bash
cd /usr/local/ngrok/
GOOS=darwin GOARCH=amd64 make release-client
</pre>
Windows的客户端编译
<pre>
cd /usr/local/go/src
GOOS=windows GOARCH=amd64 ./make.bash
cd /usr/local/ngrok/
GOOS=windows GOARCH=amd64 make release-client
</pre>


客户端配置文件（扩展名.cfg）
<pre>
server_addr: "ngrok.sunnyos.com:4443"
trust_host_root_certs: false
</pre>
服务端启动
<pre>
/usr/local/ngrok/bin/ngrokd -domain="$NGROK_DOMAIN" -httpAddr=":80"
</pre>
客户端文件路径/usr/local/ngrok/bin/
客户端使用
<pre>
./ngrok -config=./ngrok.cfg -subdomain=blog 80
setsid ./ngrok -config=./ngrok.cfg -subdomain=test 80 #在linux下如果想后台运行
</pre>
---问题总汇，以下非重点，出现问题再看--------------------------
出现这个错误说明我们需要安装hg
<pre>
package code.google.com/p/log4go: exec: "hg": executable file not found in $PATH
</pre>
解决办法
<pre>
yum install hg -y
</pre>

编译到 go get gopkg.in/yaml.v1 的时候卡住不走了，说明是git比较低，版本需要大于1.7.9.5以上

fatal: Unable to find remote helper for 'https' 出现这个问题，可以重新安装 curl curl-devel 然后再重装git
安装git-core
<pre>
wget https://www.kernel.org/pub/software/scm/git/git-core-0.99.6.tar.gz
tar zxvf git-core-0.99.6.tar.gz
cd git-core-0.99.6
make prefix=/usr/libexec/git-core install
export PATH=$PATH:/usr/libexec/git-core/
</pre>

一、查看端口占用情况的命令：lsof -i
这里返回了Linux当前所有打开端口的占用情况。第一段是进程，最后一列是侦听的协议、侦听的IP与端口号、状态。如果端口号是已知的常用服务（如80、21等），则会直接显示协议名称，如http、ftp、ssh等。
二、查看某一端口的占用情况： lsof -i:端口号

如图，查看80端口显示出nginx占用此端口，状态是listen
三、结束占用端口的进程：killall 进程名
虽然我们不建议用这种本末倒置的方法来解决冲突问题，但某些情况下还是可以直接结束掉占用进程的（比如重启Apache时进程没有完全退出，导致重启失败）
killall nginx
执行这条命令就可以了，本文结束！


<pre>
ngrokc -SER[Shost:ngrokd.ml,Sport:4443,Atoken:bf7b9226883affe7afb] -AddTun[Type:tcp,Lhost:192.168.123.1,Lport:1688,Rport:45536] -AddTun[Type:http,Lhost:192.168.123.1,Lport:80,Sdname:home] &
</pre>
kms tcp 1688

