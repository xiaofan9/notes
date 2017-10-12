# ubuntu 下搭建ngrok服务器并使用nginx进行反代理

### 必备
1. 一台Ubuntu 服务器（要有公网ip）
2. 一个域名（建议一级域名

### 步骤

#### 安装git 注意git 版本不能过低

我是使用 ppa 源安装的，安装的是最新的git 版本

<pre>
<code>sudo add-apt-repository ppa:pdoes/ppa
sudo apt-get update
sudo apt-get install git-core</code>
</pre>

如果出现提示 无法找到add-apt-repository ，请按照以下步骤安装：

<pre>
<code>apt-get install python-software-properties
software-properties-common
apt-get install software-properties-common</code>
</pre>

#### 安装go
<pre>
<code>wget https://storage.googleapis.com/golang/go1.4.1.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.4.1.linux-amd64.tar.gz
mkdir $HOME/go
echo 'export GOROOT=/usr/local/go' >> ~/.bashrc 
echo 'export GOPATH=$HOME/go' >> ~/.bashrc 
echo 'export PATH=$PATH:$GOROOT/bin:$GOPATH/bin' >> ~/.bashrc
source $HOME/.bashrc 

//软连接到/usr/bin
ln -s /usr/local/go/bin/* /usr/bin/</code>
</pre>

#### 编译ngrok
<pre>
<code>cd /usr/local/
git clone https://github.com/inconshreveable/ngrok.git
export GOPATH=/usr/local/ngrok/
export NGROK_DOMAIN="www.test.com"
</code>
</pre>

#### 生成和拷贝证书

<pre>
<code>// 生成
cd ngrok
openssl genrsa -out rootCA.key 2048
openssl req -x509 -new -nodes -key rootCA.key -subj "/CN=$NGROK_DOMAIN" -days 5000 -out rootCA.pem
openssl genrsa -out server.key 2048
openssl req -new -key server.key -subj "/CN=$NGROK_DOMAIN" -out server.csr
openssl x509 -req -in server.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out server.crt -days 5000

// 拷贝
cp rootCA.pem assets/client/tls/ngrokroot.crt
cp device.crt assets/server/tls/snakeoil.crt
cp device.key assets/server/tls/snakeoil.key</code>
</pre>

#### 针对国内服务器
<pre>
<code>vim /usr/local/ngrok/src/ngrok/log/logger.go
log "github.com/keepeye/log4go"</code>
</pre>

#### 编译

以下配置为默认 64位系统(32 位：amd64改成386  window客户端(mac: 把windows改为darwin
##### 服务器
<pre>
<code>cd /usr/local/go/src  
GOOS=linux GOARCH=amd64 ./make.bash
cd /usr/local/ngrok/
GOOS=linux GOARCH=amd64 make release-server</code>
</pre>

##### 客户端
<pre>
<code>cd /usr/local/go/src
GOOS=windows GOARCH=amd64 ./make.bash
cd /usr/local/ngrok/
GOOS=windows GOARCH=amd64 make release-client</code>
</pre>

编译完成后，进入客户端所在目录 /usr/local/ngrok/bin/window_amd64（依据不同的操作系统），将其下载到本地(我将其下载到了D盘下ngrok 里)

#### 启动

##### 服务端
<pre>
<code>cd /usr/local/ngrok/bin/
./ngrokd -domain="test.com" -httpAddr=":8081" -httpsAddr=":8089"</code>
</pre>

##### 客户端

在ngrok.exe 同级目录里 新建ngrok.cfg，并写入以下内容：
<pre>
<code>server_addr: "test.com:4443"
trust_host_root_certs: false</code>
</pre>

###### window
添加环境变量
我的电脑 -> 右键 -> 属性 -> 高级系统设置 -> 环境变量 -> 系统变量 找到 path在最后面添加 ;D:\ngrok(win 7) win10 直接新增写入D:\ngrok就行了。

打开cmd 命令输入
<pre>
<code>ngrok -config=ngrok.cfg -subdomain test 80</code>
</pre>

###### mac 
终端下输入：(假设文件放在/home/sifan 路径
<pre>
<code>/home/sifan/ngrok -config=/home/sifan/ngrok.cfg -subdomain test 80</code>
</pre>


#### 域名解析
在域名解析处 添加一条A记录 * 解析到服务器地址


#### 配置代理
nginx 反代理配置
server {
        server_name     ~^(?<subdomain>\w+)\.test\.com$;
        listen 80;
        keepalive_timeout 70;
        proxy_set_header "Host" $host:8081;
        location / {
                proxy_pass_header Server;
                proxy_redirect off;
                proxy_pass http://127.0.0.1:8081;
        }
        access_log off;
        log_not_found off;
}

#### 参考借鉴
1. <a href='http://www.jianshu.com/p/b254547b9fe5'>ngrok服务器搭建步骤</a><br />
2. <a href='http://www.07net01.com/2016/09/1676429.html'>ubuntu安装ngrok并使用nginx代理</a>

#### 结语
1. 遇到问题，千万别心急，慢慢来，检测下相关步骤是否正确
2. 严格按照以上步骤，一般来说的肯定可以搭建成功
