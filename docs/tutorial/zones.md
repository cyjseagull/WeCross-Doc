## 跨链组网

如果已完成[接入区块链](./chain.md)的体验，接下来将教您如何搭建一个WeCross跨链组网，实现不同的跨链路由之间的相互调用。请求发送至任意的跨链路由，都能够访问到正确的跨链资源。

```eval_rst
.. important::
    - WeCross跨链路由之间采用TLS协议实现安全通讯，只有持有相同ca证书的跨链路由才能建立连接。
```

### 构建多个WeCross跨链路由

```bash
cd ~/wecross
vi ipfile
```

本举例中将构造两个跨链路由，构建一个ipfile文件，将需要构建的两个跨链路由信息内容保存到文件中（按行区分：`ip地址:rpc端口:p2p端口`）：

```text
127.0.0.1:8251:25501
127.0.0.1:8252:25502
```
生成跨链路由

```bash
# -f 表示以文件为输入
bash ./WeCross/build_wecross.sh -n bill -o routers-bill -f ipfile

# 成功输出如下信息
[INFO] Create routers/127.0.0.1-8251-25501 successfully
[INFO] Create routers/127.0.0.1-8252-25502 successfully
[INFO] All completed. WeCross routers are generated in: routers-bill/
```

在routers-bill目录下生成了两个跨链路由

``` bash
tree routers-bill/ -L 1
routers-bill/
├── 127.0.0.1-8251-25501
├── 127.0.0.1-8252-25502
└── cert
```

### 跨链路由接入区块链

两个跨链路由配置同一条FISCO BCOS链的不同合约资源。

#### 配置127.0.0.1-8251-25501

让此跨链路由操作`HelloWorld`合约。

- 部署合约
```bash
cd ~/fisco/console && bash start.sh

# 部署HelloWorld合约，请将合约地址记录下来后续步骤使用
[group:1]> deploy HelloWorld
contract address: 0xe51eb006c96345f8f0d431f100f0bf619f6145d4

# 部署HelloWeCross合约，请将合约地址记录下来后续步骤使用
[group:1]> deploy HelloWeCross
contract address: 0x5854394d40e60b203e3807d6218b36d4bc0f3437

# 退出控制台
[group:1]> q
```

- 配置stub.toml

```bash
cd ~/wecross/routers-bill/127.0.0.1-8251-25501 
bash create_bcos_stub_config.sh -n bcos1
vi conf/stubs/bcos1/stub.toml
# 配置通过哪些节点接入此链
connectionsStr = ['127.0.0.1:20200','127.0.0.1:20201','127.0.0.1:20202','127.0.0.1:20203']

# 配置资源：HelloWorld合约（同时删除其他资源示例）
[[resources]]
    # name must be unique
    name = 'HelloWorld'
    type = 'BCOS_CONTRACT'
    contractAddress = '0xe51eb006c96345f8f0d431f100f0bf619f6145d4'
```

- 拷贝证书
```bash
cp ~/fisco/nodes/127.0.0.1/sdk/* ~/wecross/routers-bill/127.0.0.1-8251-25501/conf/stubs/bcos1
```

#### 配置127.0.0.1-8252-25502

让此跨链路由操作`HelloWeCross`合约

- 配置stub.toml
```bash
cd ~/wecross/routers-bill/127.0.0.1-8252-25502 
bash create_bcos_stub_config.sh -n bcos2
vi conf/stubs/bcos2/stub.toml
# 配置通过哪些节点接入此链
connectionsStr = ['127.0.0.1:20200','127.0.0.1:20201','127.0.0.1:20202','127.0.0.1:20203']

# 配置资源：HelloWeCross合约（同时删除其他资源示例）
[[resources]]
    # name must be unique
    name = 'HelloWeCross'
    type = 'BCOS_CONTRACT'
    contractAddress = '0x5854394d40e60b203e3807d6218b36d4bc0f3437'
```

- 拷贝证书

```bash
cp ~/fisco/nodes/127.0.0.1/sdk/* ~/wecross/routers-bill/127.0.0.1-8252-25502/conf/stubs/bcos2
```

#### 启动跨链路由

```bash
cd ~/wecross/routers-bill/127.0.0.1-8251-25501 
bash start.sh

cd ~/wecross/routers-bill/127.0.0.1-8252-25502 
bash start.sh
```

### 调用跨链资源

用控制台调用跨链资源，请求发到任意的跨链路由上，都可访问到跨链分区中的任意资源。

#### 配置跨链控制台

在控制台配置文件中增加新的跨链路由的信息。

```bash
cd ~/wecross/WeCross-Console
vi conf/console.xml 
# 配置map信息
<map>
    <entry key="server1" value="127.0.0.1:8251"/>
    <entry key="server2" value="127.0.0.1:8252"/>
</map>

```

#### 启动跨链控制台

``` bash
cd ~/wecross/WeCross-Console
bash start.sh
```

#### 测试请求路由功能

跨链路由能够进行请求转发，将调用请求正确地路由至相应的跨链资源。无论请求发至哪个跨链路由，都可正确路由至相应跨链路由配置的链上资源。

```bash
# 查看配置的所有跨链路由
[server1]> listServers 
{server1=127.0.0.1:8251, server2=127.0.0.1:8252}

# 查看server1的资源列表，可以看到server2所连接的跨链路由的资源（bill.bcos2.HelloWeCross）也在列表中
[server1]> listResources
Resources{
    errorCode=0,
    errorMessage='',
    resourceList=[
        WeCrossResource{
            checksum='0x7027331008fe69a87ec703a006461a14a652f5380071c8868cc08d3c7247d608',
            type='REMOTE_RESOURCE',
            distance=1,
            path='bill.bcos2.HelloWeCross'
        },
        WeCrossResource{
            checksum='0x3d58f2c92c9894abbcff5192c78f9ea823c39b089c44f976bb53e63d9285b0b0',
            type='BCOS_CONTRACT',
            distance=0,
            path='bill.bcos1.HelloWorld'
        }
    ]
}

# 在server1中调用server2连接的资源
[server1]> call bill.bcos2.HelloWeCross Int getNumber
Receipt{
    errorCode=0,
    errorMessage='success',
    hash='null',
    result=[
        2019
    ]
}

# 切换server2
[server1]> switch server2

# server2调用server1的资源
[server2]> call bill.bcos1.HelloWorld String get
Receipt{
    errorCode=0,
    errorMessage='success',
    hash='null',
    result=[
        Hello,
        World!
    ]
}

# 退出控制台
[server2]> q
````

停止跨链路由

```shell
cd ~/wecross/routers-bill/127.0.0.1-8251-25501
bash stop.sh
cd ~/wecross/routers-bill/127.0.0.1-8252-25502
bash stop.sh
```

