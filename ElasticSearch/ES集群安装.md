#### 创建软件安装文件夹
```bash
sudo mkdir /opt/apps
```
#### 准备安装包

- jdk-8u181-linux-x64.tar.gz
- node-v10.8.0-linux-x64.tar.xz
- elasticsearch-head-master.zip

### jdk安装

- 创建文件夹
```bash
sudo mkdir -p /usr/lib/jvm
```
- 安装包解压到 /usr/lib/jvm

```bash
sudo tar -zxvf jdk-8u181-linux-x64.tar.gz -C /usr/lib/jvm
```

- 配置 Jdk的环境变量

```bash
sudo vim /etc/profile
```

```bash
export JAVA_HOME=/usr/lib/jvm/jdk1.8.0_181
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export PATH=${JAVA_HOME}/bin:$PATH
```
- 使环境变量及时生效

```bash
source /etc/profile
```
- 验证

```bash
java -version
``` 

### NodeJS安装

- 解压缩

```bash
sudo  tar -xvf node-v10.8.0-linux-x64.tar.xz  -C /opt/apps
```

- 配置环境变量

```bash
sudo vim /etc/profile
```
```bash
export NODE_HOME=/opt/apps/node-v10.8.0-linux-x64
export PATH=${NODE_HOME}/bin:$PATH
```
- 使环境变量生效

```bash
source /etc/profile
```

- 环境验证

```bash
node -v
npm -v
```
### ElasticSearch 安装

- 解压软件包

```bash
sudo tar -zxvf elasticsearch-6.3.2.tar.gz  -C /opt/apps
```

- 修改权限

```bash
sudo chown ppb -R /opt/apps/elasticsearch-6.3.2/
```

- 修改配置文件

```bash
 vim /opt/apps/elasticsearch-6.3.2/config/elasticsearch.yml
```
 
 - 添加如下内容

 ```yml
 cluster.name: ppb-CLUSTER
 node.name: node-01
 path.data: /opt/es/data
 path.logs: /opt/es/logs
 bootstrap.memory_lock: false
 network.host: 0.0.0.0
 http.port: 9200
 discovery.zen.ping.unicast.hosts: ["node-01", "node-02"]
 discovery.zen.minimum_master_nodes: 2
 http.cors.enabled: true
 http.cors.allow-origin: "*"
 ```
- 设置jvm参数

```bash
vim /opt/apps/elasticsearch-6.3.2/config/jvm.options
-Xms8g
-Xmx8g
```
- 修改系统参数

```bash
sudo vim /etc/security/limits.conf
```
```
* soft nofile 65536
* hard nofile 65536
* soft nproc  65536
* hard nproc  65536
```
```bash
 sudo vim /etc/sysctl.conf

 vm.max_map_count= 262144
 ```
 
- 使其生效

```
sudo sysctl -p 
```

- 启动es

```bash
/opt/apps/elasticsearch-6.3.2/bin/elasticsearch -d
```

### ElasticSearch-head 安装
- 解压缩

```bash
sudo unzip elasticsearch-head-master.zip -d /opt/apps/
```

- 下载构建插件

  - //设置淘宝源 安装时使用国内源 公司可能需要外网权限
    - npm config set registry http://registry.npm.taobao.org/

- 设置权限

```bash
npm config set registry https://registry.npmjs.org/
sudo chown ppb -R node-v10.8.0-linux-x64/
sudo chown ppb -R elasticsearch-head-master/
npm install -g grunt  -cli
cd /opt/apps/elasticsearch-head-master
npm install
```
- 修改HEAD源码

```bash
sudo vim /opt/apps/elasticsearch-head-master/Gruntfile.js
```

```js

connect: {
    server: {
        options: {
            port: 9100,
            hostname: '*',
            base: '.',
            keepalive: true
        }
    }
}
```
- 修改app.js文件

```bash
vim  /opt/apps/elasticsearch-head-master/_site/app.js
```
- 修改内容如下 文件的第4355 行
es-node表示es节点ip
```js
this.base_uri = this.config.base_uri || this.prefs.get("app-base_uri") || "http://es-node:9200";
```

- 启动elastic-head

```bash
cd /opt/apps/elasticsearch-head-master/
grunt server &
```

http://host:9100/  集群监控界面


