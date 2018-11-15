## SearchGuard 安装

1. searchguard 必须与所选版本一致

### 在线安装

```bash
bin/elasticsearch-plugin install -b com.floragunn:search-guard-6:6.3.2-23.0
```

### 离线安装

```bash
bin/elasticsearch-plugin install -b file:../search-guard-6-6.3.2-23.0.zip
```

## 证书生成

### 在线生成方式

    https://search-guard.com/tls-certificate-generator/

### 离线工具生成

   #### 离线工具下载
   > https://repo1.maven.org/maven2/com/floragunn/search-guard-tlstool/1.5/

   根据以上链接下载所需要的工具解压
   * 目录结构
   > config 配置文件目录，工具可以根据配置文件模板为你生成证书
   >
   > dep 工具所依赖的jar包
   >
   > tools 生成证书的脚本

   
   #### 生成模板配置

   在config文件夹中创建tlsconfig.yml文件

   ```yml
    ca:
        root:
            # The distinguished name of this CA. You must specify a distinguished name.   
            dn: CN=root.ca.xxx.com,OU=CA,O=xxx Com\, Inc.,DC=xxx,DC=com

            # The size of the generated key in bits
            keysize: 2048

            # The validity of the generated certificate in days from now
            validityDays: 3650
            
            # Password for private key
            #   Possible values: 
            #   - auto: automatically generated password, returned in config output; 
            #   - none: unencrypted private key; 
            #   - other values: other values are used directly as password   
            pkPassword: auto 
            
            # The name of the generated files can be changed here
            file: root-ca.pem
        
    ### 
    ### Default values and global settings
    ###
    defaults:

        # The validity of the generated certificate in days from now
        validityDays: 3650 
        
        # Password for private key
        #   Possible values: 
        #   - auto: automatically generated password, returned in config output; 
        #   - none: unencrypted private key; 
        #   - other values: other values are used directly as password   
        pkPassword: auto      
        
        # Specifies to recognize legitimate nodes by the distinguished names
        # of the certificates. This can be a list of DNs, which can contain wildcards.
        # Furthermore, it is possible to specify regular expressions by
        # enclosing the DN in //. 
        # Specification of this is optional. The tool will always include
        # the DNs of the nodes specified in the nodes section.            
        #nodesDn:
        #- "CN=*.example.com,OU=Ops,O=Example Com\\, Inc.,DC=example,DC=com"
        # - 'CN=node.other.com,OU=SSL,O=Test,L=Test,C=DE'
        # - 'CN=*.example.com,OU=SSL,O=Test,L=Test,C=DE'
        # - 'CN=elk-devcluster*'
        # - '/CN=.*regex/' 

        # If you want to use OIDs to mark legitimate node certificates, 
        # the OID can be included in the certificates by specifying the following
        # attribute
        
        # nodeOid: "1.2.3.4.5.5"

        # The length of auto generated passwords            
        generatedPasswordLength: 12
        
        # Set this to true in order to generate config and certificates for 
        # the HTTP interface of nodes
        httpsEnabled: true
        
        # Set this to true in order to re-use the node transport certificates
        # for the HTTP interfaces. Only recognized if httpsEnabled is true
        
        # reuseTransportCertificatesForHttp: false
        
        # Set this to true to enable hostname verification
        #verifyHostnames: false
        
        # Set this to true to resolve hostnames
        #resolveHostnames: false
        
        
    ###
    ### Nodes
    ###
    # 
    # Specify the nodes of your ES cluster here
    #      
    nodes:
        - name: node-01
            dn: CN=node-01.xxx.com,OU=Ops,O=xxx Com\, Inc.,DC=xxx,DC=com
            dns: node-01.xxx.com
            ip: 111.111.111.11
        - name: node-02
            dn: CN=node-02.xxx.com,OU=Ops,O=xxx Com\, Inc.,DC=xxx,DC=com
            dns: 
              - node-02.xxx.com
            ip: 
              - 111.111.111.12
        - name: node-03
            dn: CN=node-03.xxx.com,OU=Ops,O=xxx Com\, Inc.,DC=xxx,DC=com
            dns: node-03.xxx.com
            ip: 
              - 111.111.111.13

    ###
    ### Clients
    ###
    #
    # Specify the clients that shall access your ES cluster with certificate authentication here
    #
    # At least one client must be an admin user (i.e., a super-user). Admin users can
    # be specified with the attribute admin: true    
    #        
    clients:
        - name: ppb
            dn: CN=ppb.xxx.com,OU=Ops,O=xxx Com\, Inc.,DC=xxx,DC=com
        - name: backend
            dn: CN=backend.xxx.com,OU=Ops,O=xxx Com\, Inc.,DC=xxx,DC=com
            admin: true
 
   ```
 > node配置说明
 >
 > * node-01 node-02 node-03 可以对应ES集群中的node节点名称
 >
 > * dns: 此项注意后面会用上。
 >
 > * ip 与es集群ip对应
 >
 > client配置
 >
 > * 上面的配置是client端证书生成用于client端访问es用的
 > 
 > xxx 替换成公司域名 xxx.com baidu.com  xxx即 baidu


#### 证书生成

- 执行命令生成证书
    ```bash
    <installation directory>/tools/sgtlstool.sh -c ../config/tlsconfig.yml -crt
    ```
- 证书在installation directory/tools/out目录下存放
 
  - 根证书
    - root-ca.pem Root certificate
    - root-ca.key Private key of the Root CA
    - root-ca.readme Passwords of the root and intermediate CAs
  - node节点证书
    - [nodename].pem Node certificate
    - [nodename].key Private key of the node certificate
    - [nodename]_http.pem REST certificate, only generated if reuseTransportCertificatesForHttp is false
    - [nodename]_http.key Private key of the REST certificate, only generated if reuseTransportCertificatesForHttp is false
    - [nodename]_elasticsearch_config_snippet.yml Search Guard configuration snippet for this node, add this to elasticsearch.yml
  - 客户端证书
    - [name].pem Client certificate
    - [name].key Private key of the client certificate
    - client-certificates.readme Contains the auto-generated passwords for the certificates

- 说明：
   - 根据配置文件

      - 客户端： 会生成5个文件 2个客户端配置，每个客户端2个文件，外加一个readme文件
      - node: 配置了3个节点，会生成3*5个文件
      - 根证书：唯一的，生成3个文件
      - 证书密码获取：当然在生成的时候你可以自己在配置文件配置密码，我这里是自动生成的
        - root-ca.readme 里面找到root证书的密码
        - client-certificates.readme 里面获取客户端证书密码
        - [nodename]_elasticsearch_config_snippet.yml存放着配置es节点的模板，里面存在证书密码
    


## 证书配置

### 复制证书
- 配置ES

    - 复制 root-ca.pem 到 elasticsearch的config目录下
    - 复制 [nodename].pem elasticsearch的config目录下
    - 复制 [nodename].key elasticsearch的config目录下
    - 复制 [nodename]_http.pem elasticsearch的config目录下
    - 复制 [nodename]_http.key elasticsearch的config目录下

        ```bash
        sudo cp out/[nodename]*.key /[espath]/config/
        sudo cp out/[nodename]*.pem /[espath]/config/
        sudo cp out/root-ca.pem /[espath]/config/

        ```

    - 复制 root-ca.pem 到 elasticsearch的plugins/search-guard-[version]/tools目录下 
    
        - 复制 [username].pem elasticsearch的config目录下
        - 复制 [username].key elasticsearch的config目录下

        ```
        sudo cp out/[username].key /opt/apps/elasticsearch-6.3.2/plugins/search-guard-6/tools/

        sudo cp out/[username].pem /opt/apps/elasticsearch-6.3.2/plugins/search-guard-6/tools/

        sudo cp out/root-ca.pem /opt/apps/elasticsearch-6.3.2/plugins/search-guard-6/tools/

        ```
- 配置ES(所有节点配置上)

    - 配置如下：  

        ```yml
        cluster.name: PPB-CLUSTER
        node.name: node-01
        network.host: 0.0.0.0
        http.cors.enabled: true
        http.cors.allow-origin: "*"
        http.cors.allow-headers: "Authorization,X-Requested-With,Content-Length,Content-Type"
        searchguard.ssl.transport.pemcert_filepath: node-01.pem
        searchguard.ssl.transport.pemkey_filepath: node-01.key
        searchguard.ssl.transport.pemkey_password: ******
        searchguard.ssl.transport.pemtrustedcas_filepath: root-ca.pem
        searchguard.ssl.transport.enforce_hostname_verification: false
        searchguard.ssl.transport.resolve_hostname: false
        searchguard.ssl.http.enabled: true
        searchguard.ssl.http.pemcert_filepath: node-01_http.pem
        searchguard.ssl.http.pemkey_filepath: node-01_http.key
        searchguard.ssl.http.pemkey_password: ********
        searchguard.ssl.http.pemtrustedcas_filepath: root-ca.pem
        searchguard.nodes_dn:
            - CN=node-01.xxx.com,OU=Ops,O=xxx Com\, Inc.,DC=xxx,DC=com
            - CN=node-02.xxx.com,OU=Ops,O=xxx Com\, Inc.,DC=xxx,DC=com
            - CN=node-03.xxx.com,OU=Ops,O=xxx Com\, Inc.,DC=xxx,DC=com
        searchguard.authcz.admin_dn:
            - CN=ppb.xxx.com,OU=Ops,O=xxx Com\, Inc.,DC=xxx,DC=com
        searchguard.allow_unsafe_democertificates: true
        xpack.security.enabled: false
        searchguard.enterprise_modules_enabled: false
        ```  

      - 配置提示(避坑)
        - network.host: 0.0.0.0 避免执行sgadmin命令时报错
        - xpack.security.enabled: false 禁用xpack
        - es-head支持
          - http.cors.enabled: true
          - http.cors.allow-origin: "*"
          - http.cors.allow-headers: "Authorization,X-Requested-With,- Content-Length,Content-Type"
- 启动ES节点

    ```bash
    /[espath]/bin/elasticsearch -d
    ```
- 激活searchguard
  - 进入/[espath]/plugins/search-guard-6/tools

    ```
        ./sgadmin.sh -cacert root-ca.pem -cert ppb.pem -key pbb.key -keypass gb1bbJEWO5lW -nhnv -icl -cd ../sgconfig
    ```  
  - ppb.pem ppb.key 即你后面在Spring里面集成时需要的客户端证书       
  - 如果提示没有权限则增加sgadmin.sh的权限重新执行即可

- 按如上方式把所有节点启动起来
- 验证：

    浏览器输入 https://es-node:9200，某一节点让输入用户名密码说明安装成功

## SpringBoot集成

### 通过TransportClient方式访问ES
#### 依赖pom.xml
```xml
          <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
        </dependency>
        <dependency>
            <groupId>com.floragunn</groupId>
            <artifactId>search-guard-ssl</artifactId>
            <version>5.6.4-23</version>
        </dependency>
        <dependency>
            <groupId> org.elasticsearch.plugin</groupId>
            <artifactId> transport-netty3-client</artifactId>
            <version> 5.1.1</version>
      </dependency>
```
#### 配置文件配置application.properties
```yml
search_guard.elasticsearch.nodes = node-01.xxx.com,node-02.xxx.com,node-03.xxx.com
search_guard.elasticsearch.transclient.port = 9300
search_guard.elasticsearch.clustername = PPB-CLUSTER
search_guard.ssl_transport_pemkey_password = ******
search_guard.ssl_transport_pemkey_filepath = /ssl/ppb.key
search_guard.ssl_transport_pemcert_filepath = /ssl/ppb.pem
search_guard.ssl_transport_pemtrustedcas_filepath = ssl/root-ca.pem
```

#### javaConfig

```java
@Configuration
@Profile("test")
public class ElasticSearchConfig {
    private static final Logger LOGGER = LoggerFactory.getLogger(ElasticSearchConfig.class);
    @Value("${search_guard.elasticsearch.clustername}")
    private String clusterName;
    @Value("${search_guard.ssl_transport_pemkey_filepath}")
    private String pemKeyFilePath;
    @Value("${search_guard.ssl_transport_pemcert_filepath}")
    private String pemCertFilePath;
    @Value("${search_guard.ssl_transport_pemtrustedcas_filepath}")
    private String pemTrustedCasFilePath;
    @Value("${search_guard.ssl_transport_pemkey_password}")
    private String pemkeyPassword;
    @Value("#{'${search_guard.elasticsearch.nodes}'.split(',')}")
    private List<String> clusterNodes;
    @Value("${search_guard.elasticsearch.transclient.port}")
    private int transportPort;

    @Bean
    public Client client() throws Exception {
        LOGGER.info("加载配置" + transportPort);
        Settings.Builder settingsBuilder =
                Settings.builder()
                        .put(SSLConfigConstants.SEARCHGUARD_SSL_TRANSPORT_PEMKEY_FILEPATH, pemKeyFilePath)
                        .put(SSLConfigConstants.SEARCHGUARD_SSL_TRANSPORT_PEMCERT_FILEPATH, pemCertFilePath)
                        .put(SSLConfigConstants.SEARCHGUARD_SSL_TRANSPORT_PEMTRUSTEDCAS_FILEPATH, pemTrustedCasFilePath)
                        .put(SSLConfigConstants.SEARCHGUARD_SSL_TRANSPORT_PEMKEY_PASSWORD, pemkeyPassword)
                        .put("cluster.name", clusterName)
                        .put("client.transport.sniff", false);
        Settings settings = settingsBuilder.build();
        TransportClient client = new PreBuiltTransportClient(settings, SearchGuardSSLPlugin.class);
        for (String node : clusterNodes) {
            LOGGER.info("加载配置 " + node);
            client.addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName(node), transportPort));
        }
        return client;
    }

    @Bean
    public ElasticsearchTemplate elasticsearchTemplate() throws Exception {
        return new ElasticsearchTemplate(client());
    }

}
```


### 避坑指南

- search_guard.elasticsearch.nodes 配置一定要与节点证书的dns一致node-01.xxx.com
- 在发布服务器上修改host文件，把es节点与对应的ip配置进去
- client.transport.sniff 配置成false。
- 证书文件放到对应的目录下 所需要的证书:
  - root-ca.pem
  - [username].key
  - [username].pem
  - 用户证书一定要使用你执行sgadmin命令时使用的证书

## es-head插件如何访问获取es信息
默认用户名密码是admin
```
http://[es-head]:9100/?base_uri=https://[es-node]:9200&auth_user=admin&auth_password=admin
```

## 总结
    最后就可以开启权限配置了。