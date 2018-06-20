
# web3sdk使用指南

## （一）介绍
#### 1. 功能介绍

web3sdk是用来访问fisco-bcos节点的java API。<br />
主要由两部分组成：<br />
(1) AMOP（链上链下）系统旨在为联盟链提供一个安全高效的消息信道。<br />
(2) web3j(Web3 Java Ethereum Ðapp API),这个部分来源于[web3j](https://github.com/web3j/web3j)，并针对bcos的做了相应改动。主要改动点为：

① 交易结构中增加了randomid和blocklimit，这个改动对于sdk的使用者是透明的。<br /> ②为web3增加了AMOP消息信道。<br />

本文档主要说明如何使用AMOP消息信道调用web3j的API。AMOP（链上链下）的使用可以参考AMOP使用指南。web3j的使用可以参考[web3j](https://github.com/web3j/web3j)和[存证sample](https://github.com/FISCO-BCOS/evidenceSample)。



#### 2. 特性介绍

- **[部署合约](#23部署合约)**
- **[提供将合约代码转换成java代码的工具](#1合约编译及java-wrap代码生成)**
- **[提供系统合约部署和使用工具](#2系统合约相关接口说明)**
- **[使用国密算法发交易](doc/guomi_support_manual.md)**：[国密版FISCO BCOS](https://github.com/FISCO-BCOS/FISCO-BCOS/blob/master/doc/国密操作文档.md) 使用国密算法验证交易，客户端相应地需使用国密算法对交易进行签名，web3sdk客户端为国密版FISCO BCOS提供了支持，具体使用方法可参考文档[《web3sdk对国密版FISCO BCOS的支持》](doc/guomi_support_manual.md)



## （二）运行环境
1、需要首先搭建FISCO BCOS区块链环境（快速搭建可参考[FISCO BCOS一键快速安装部署](https://github.com/FISCO-BCOS/FISCO-BCOS/tree/master/sample)）。<br />
2、JDK1.8。

## （三） 配置
以下为SDK的配置案例<br />
1、SDK配置（Spring Bean）：<br />
``` 

	<?xml version="1.0" encoding="UTF-8" ?>
	<beans xmlns="http://www.springframework.org/schema/beans"
		xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"
		xmlns:tx="http://www.springframework.org/schema/tx" xmlns:aop="http://www.springframework.org/schema/aop"
		xmlns:context="http://www.springframework.org/schema/context"
		xsi:schemaLocation="http://www.springframework.org/schema/beans   
	    http://www.springframework.org/schema/beans/spring-beans-2.5.xsd  
	         http://www.springframework.org/schema/tx   
	    http://www.springframework.org/schema/tx/spring-tx-2.5.xsd  
	         http://www.springframework.org/schema/aop   
	    http://www.springframework.org/schema/aop/spring-aop-2.5.xsd">
	    
	<!-- AMOP消息处理线程池配置，根据实际需要配置 -->
	<bean id="pool" class="org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor">
		<property name="corePoolSize" value="50" />
		<property name="maxPoolSize" value="100" />
		<property name="queueCapacity" value="500" />
		<property name="keepAliveSeconds" value="60" />
		<property name="rejectedExecutionHandler">
			<bean class="java.util.concurrent.ThreadPoolExecutor.AbortPolicy" />
		</property>
	</bean>
	
	<!-- 系统合约地址配置，在使用./web3sdk SystemProxy|AuthorityFilter等系统合约工具时需要配置 -->
	<bean id="toolConf" class="org.bcos.contract.tools.ToolConf">
		<property name="systemProxyAddress" value="0x919868496524eedc26dbb81915fa1547a20f8998" />
		<!--GOD账户的私钥-->（注意去掉“0x”）
		<property name="privKey" value="bcec428d5205abe0f0cc8a734083908d9eb8563e31f943d760786edf42ad67dd" />
		<!--GOD账户-->
		<property name="account" value="0x776bd5cf9a88e9437dc783d6414bccc603015cf0" />
		<property name="outPutpath" value="./output/" />
	</bean>

	<!-- 区块链节点信息配置 -->
	<bean id="channelService" class="org.bcos.channel.client.Service">
		<property name="orgID" value="WB" /> <!-- 配置本机构名称 -->
			<property name="allChannelConnections">
				<map>
					<entry key="WB"> <!-- 配置本机构的区块链节点列表（如有DMZ，则为区块链前置）-->
						<bean class="org.bcos.channel.handler.ChannelConnections">
						    <property name="caCertPath" value="classpath:ca.crt" />
						    <property name="clientKeystorePath" value="classpath:client.keystore" />
						    <property name="keystorePassWord" value="123456" />
						    <property name="clientCertPassWord" value="123456" />
							<property name="connectionsStr">
								<list>
									<value>NodeA@127.0.0.1:30333</value><!-- 格式：节点名@IP地址:channelport，节点名可以为任意名称 -->
								</list>
							</property>
						</bean>
					</entry>
				</map>
			</property>
		</bean>
	</bean>
```
2、ca.crt(用来验证节点或者前置的CA证书)：

```
和节点的ca.crt保持一致。
```
3、client.keystore(用来做sdk的ssl身份证书)：
```
里面需要包含一个由节点CA证书颁发的，别名为client的身份证书。
```
## （四）SDK使用

### 1、代码案例：

```

	package org.bcos.channel.test;
	import org.bcos.web3j.abi.datatypes.generated.Uint256;
	import org.bcos.web3j.crypto.Credentials;
	import org.bcos.web3j.crypto.ECKeyPair;
	import org.bcos.web3j.crypto.Keys;
	import org.bcos.web3j.protocol.core.methods.response.EthBlockNumber;
	import org.bcos.web3j.protocol.core.methods.response.TransactionReceipt;
	import org.slf4j.Logger;
	import org.slf4j.LoggerFactory;
	import org.springframework.context.ApplicationContext;
	import org.springframework.context.support.ClassPathXmlApplicationContext;
	import org.bcos.channel.client.Service;
	import org.bcos.web3j.protocol.Web3j;
	import org.bcos.web3j.protocol.channel.ChannelEthereumService;

	import java.math.BigInteger;

	public class Ethereum {
		static Logger logger = LoggerFactory.getLogger(Ethereum.class);
		
		public static void main(String[] args) throws Exception {
			
			//初始化Service
			ApplicationContext context = new ClassPathXmlApplicationContext("classpath:applicationContext.xml");
			Service service = context.getBean(Service.class);
			service.run();
			
			System.out.println("开始测试...");			System.out.println("===================================================================");
			
			logger.info("初始化AOMP的ChannelEthereumService");
			ChannelEthereumService channelEthereumService = new ChannelEthereumService();
			channelEthereumService.setChannelService(service);
			
			//使用AMOP消息信道初始化web3j
			Web3j web3 = Web3j.build(channelEthereumService);

			logger.info("调用web3的getBlockNumber接口");
			EthBlockNumber ethBlockNumber = web3.ethBlockNumber().sendAsync().get();
			logger.info("获取ethBlockNumber:{}", ethBlockNumber);

			//初始化交易签名私钥
			ECKeyPair keyPair = Keys.createEcKeyPair();
			Credentials credentials = Credentials.create(keyPair);

			//初始化交易参数
			java.math.BigInteger gasPrice = new BigInteger("30000000");
			java.math.BigInteger gasLimit = new BigInteger("30000000");
			java.math.BigInteger initialWeiValue = new BigInteger("0");

			//部署合约
			Ok ok = Ok.deploy(web3,credentials,gasPrice,gasLimit,initialWeiValue).get();
			System.out.println("Ok getContractAddress " + ok.getContractAddress());
			
			//调用合约接口
			java.math.BigInteger Num = new BigInteger("999");
			Uint256 num = new Uint256(Num);
			TransactionReceipt receipt = ok.trans(num).get();
			System.out.println("receipt transactionHash" + receipt.getTransactionHash());

			//查询合约数据
			num = ok.get().get();
			System.out.println("ok.get() " + num.getValue());
		}
	}
```
### 2、代码说明：
这个代码演示了web3sdk和fisco-bcos的主要特性。
#### 2.1、初始化web3j对象，连接到fisco-bcos节点：
web3sdk使用AMOP（链上链下）连接fisco-bcos节点。
##### 2.1.1、初始化AMOP的service，初始化函数详见配置文件，sleep(3000)是确保AMOP网络连接初始化完成。注意：org.bcos.channel.client.Service在Java Client端须为单实例，否则与链上节点连接会有问题。
```
    Service service = context.getBean(Service.class);
    service.run();
    Thread.sleep(3000);
```
##### 2.1.2、初始化AMOP的ChannelEthereumService，ChannelEthereumService通过AMOP的网络连接支持web3j的Ethereum JSON RPC协议。（JSON RPC 参考[JSON RPC API](https://github.com/ethereum/wiki/blob/master/JSON-RPC.md)）
```
    ChannelEthereumService channelEthereumService = new ChannelEthereumService();
    channelEthereumService.setChannelService(service);
```
##### 2.1.3、使用ChannelEthereumService初始化web3j对象。（web3sdk也支持通过http和ipc来初始化web3j对象，但在fisc-bcos中推荐使用AMOP）
```
    Web3j web3 = Web3j.build(channelEthereumService);
```
##### 2.1.4、调用web3j的rpc接口。样例给出的是获取块高例子，web3j还支持sendRawTransaction、getCode、getTransactionReceipt等等。部分API参看5.3 web3j API说明（ 参考[JSON RPC API](https://github.com/ethereum/wiki/blob/master/JSON-RPC.md)）
```
    EthBlockNumber ethBlockNumber = web3.ethBlockNumber().sendAsync().get();
```
#### 2.2、初始化交易签名私钥（加载钱包）。样例给出的是新构建一个私钥文件。web3sdk也可以用证书来初始化交易签名私钥，只要是椭圆曲线加密算法(ECDSA-secp256k1)的私钥就能用来加载，对交易进行签可参考[存证sample](https://github.com/FISCO-BCOS/evidenceSample)。
```
    ECKeyPair keyPair = Keys.createEcKeyPair();
    Credentials credentials = Credentials.create(keyPair);
```
#### 2.3、部署合约。
##### 2.3.1、合约说明（Ok.sol）。
```
pragma solidity ^0.4.2;
contract Ok{
    
    struct Account{
        address account;
        uint balance;
    }
    
    struct  Translog {
        string time;
        address from;
        address to;
        uint amount;
    }
    
    Account from;
    Account to;
    
    Translog[] log;

    function Ok(){
        from.account=0x1;
        from.balance=10000000000;
        to.account=0x2;
        to.balance=0;

    }
    function get()constant returns(uint){
        return to.balance;
    }
    function trans(uint num){
    	from.balance=from.balance-num;
    	to.balance+=num;
    	log.push(Translog("20170413",from.account,to.account,num));
    }
}
```
##### 2.3.2、合约java类生产说明，详见（5.1）合约编译及java Wrap代码生成。
##### 2.3.3、初始化部署合约交易相关参数，正常情况采用demo给出值即可。
```
    java.math.BigInteger gasPrice = new BigInteger("30000000");
	java.math.BigInteger gasLimit = new BigInteger("30000000");
	java.math.BigInteger initialWeiValue = new BigInteger("0");
```
##### 2.3.3、部署合约。
```
    Ok ok = Ok.deploy(web3,credentials,gasPrice,gasLimit,initialWeiValue).get();
```
#### 2.4、更新智能合约成员的值。
```
    TransactionReceipt receipt = ok.trans(num).get();
```
#### 2.5、查询智能合约成员的值。
```
    num = ok.get().get();
```

### 2.6、 交易回调通知

交易回调通知是指针对合约的交易接口（非constant function和构造函数），通过web3j生成的java代码后，会在原来的java函数基础上，新增了同名的重载函数，与原来的区别在于多了一个TransactionSucCallback的参数。原有接口web3j在底层实现的机制是通过轮训机制来获取交易回执结果的，而TransactionSucCallback是通过服务端，也就是区块链节点来主动push通知，相比之下，会更有效率和时效性。当触发到onResponse的时候，代表这笔交易已经成功上链。使用例子：

```
ObjectMapper objectMapper = ObjectMapperFactory.getObjectMapper();
Uint256 num = new Uint256(Num);
ok.trans(num, new TransactionSucCallback() {
	@Override
    public void onResponse(EthereumResponse response) {
              TransactionReceipt transactionReceipt = objectMapper.readValue(ethereumResponse.getContent(), TransactionReceipt.class);
              //解析event log
   	}
});
```
## （五）SDK工具包功能说明
拉取源码，在根目录下执行gradle build，将生成SDK工具包dist.

### 1.合约编译及java Wrap代码生成

* 智能合约语法及细节参考 <a href="https://solidity.readthedocs.io/en/develop/solidity-in-depth.html">solidity官方文档</a>。

* 安装fisco-solc,fisco-solc为solidity编译器,[下载fisco-solc](https://github.com/FISCO-BCOS/fisco-solc)。

  将下载下来的fisco-solc，将fisco-solc拷贝到/usr/bin目录下，执行命令chmod +x fisco-solc。如此fisco-solc即安装完成。

* SDK执行gradle build 之后生成工具包(dist)。

* 工具包中/dist/bin文件夹下为合约编译的执行脚本，/dist/contracts为合约存放文件夹,将自己的合约复制进/dist/contracts文件夹中（建议删除文件夹中其他无关的合约)，/dist/apps为sdk jar包，/dist/lib为sdk依赖jar包，/dist/output（不需要新建，脚本会创建）为编译后输出的abi、bin及java文件目录。

* 在bin文件夹下compile.sh为编译合约的脚本，执行命令sh compile.sh [参数1：java包名]执行成功后将在output目录生成所有合约对应的abi,bin,java文件，其文件名类似：合约名字.[abi|bin|java]。compile.sh脚本执行步骤实际分为两步，1.首先将sol源文件编译成abi和bin文件，依赖solc工具；2.将bin和abi文件编译java Wrap代码，依赖web3sdk.

* 例如：在以上工作都以完成,在/dist/bin文件夹下输入命令
    sh compile.sh com ,在/dist/output目录下生成abi、bin文件，在/dist/output/com目录下生成java Wrap代码。

### 2.系统合约相关接口说明

在SDK工具包/dist/bin目录下，compile.sh为合约编译脚本，web3sdk为SDK的的执行脚本。web3sdk脚本中将系统合约接口进行暴露。系统合约介绍文档参考[FISCO BCOS系统合约介绍](https://github.com/FISCO-BCOS/Wiki/tree/master/FISCO-BCOS%E7%B3%BB%E7%BB%9F%E5%90%88%E7%BA%A6%E4%BB%8B%E7%BB%8D)

系统合约接口命令参考如下：

```
./web3sdk InitSystemContract   
./web3sdk SystemProxy          
./web3sdk AuthorityFilter
./web3sdk NodeAction all|registerNode|cancelNode
./web3sdk CAAction add|remove|all
./web3sdk ConfigAction get|set
./web3sdk ConsensusControl deploy|turnoff|list
```

InitSystemContract用来部署一套系统合约（用来做链的初始化和测试，生产环境请谨慎操作）。部署完成后需要将系统合约地址替换到各个节点的config.json和web3sdk工具的applicationContext.xml配置中，并重启节点。
SystemProxy|AuthorityFilter|......等其他工具applicationContext.xml配置中系统合约的地址。
系统合约接口代码参照：org.bcos.contract.tools.InitSystemContract,org.bcos.contract.tools.SystemContractTools

### 3.web3j API说明

在SDK工具包/dist/bin目录下，compile.sh为合约编译脚本，web3sdk为SDK的的执行脚本。web3sdk脚本中将web3j API接口进行暴露，使用命令执行调用打印API返回值（JSON RPC 参考[JSON RPC API](https://github.com/ethereum/wiki/blob/master/JSON-RPC.md)）

web3j API接口命令参考如下，--后为参数说明：

```
./web3sdk web3_clientVersion 
./web3sdk eth_accounts
./web3sdk eth_blockNumber
./web3sdk eth_pbftView
./web3sdk eth_getCode address blockNumber  --地址 存储位置整数
./web3sdk eth_getBlockTransactionCountByHash blockHash   --区块hash
./web3sdk eth_getTransactionCount address blockNumber   --区块号
./web3sdk eth_getBlockTransactionCountByNumber blockNumber  --区块号
./web3sdk eth_sendRawTransaction signTransactionData  --签名的交易数据
./web3sdk eth_getBlockByHash blockHash true|false   --区块hash true|false
./web3sdk eth_getBlockByNumber blockNumber  --区块号
./web3sdk eth_getTransactionByBlockNumberAndIndex blockNumber transactionPosition  --区块号 交易位置
./web3sdk eth_getTransactionByBlockHashAndIndex blockHash transactionPosition  --区块hash 交易位置
./web3sdk eth_getTransactionReceipt transactionHash  --交易hash
```

web3j API接口代码参照：

org.bcos.web3j.console.Web3RpcApi

### 4.权限控制说明

在SDK工具包/dist/bin目录下，compile.sh为合约编译脚本，web3sdk为SDK的的执行脚本。web3sdk脚本中将权限相关的接口进行暴露，使用相关命令执行即可。权限控制介绍文档请参看[FISCO BCOS权限模型（ARPI）介绍](https://github.com/FISCO-BCOS/Wiki/tree/master/FISCO%20BCOS%E6%9D%83%E9%99%90%E6%A8%A1%E5%9E%8B) 、[联盟链的权限体系](https://github.com/FISCO-BCOS/Wiki/tree/master/%E5%8C%BA%E5%9D%97%E9%93%BE%E7%9A%84%E6%9D%83%E9%99%90%E4%BD%93%E7%B3%BB) 。

```
org.bcos.contract.tools.ARPI_Model    #一键执行类
org.bcos.contract.tools.AuthorityManagerTools
```

在使用时，需要在applicationContext.xml文件中配置相关参数：

```
	<!-- 系统合约地址配置置-->
	<bean id="toolConf" class="org.bcos.contract.tools.ToolConf">
		<!--系统合约-->
		<property name="systemProxyAddress" value="0x919868496524eedc26dbb81915fa1547a20f8998" />
		<!--GOD账户的私钥-->（注意去掉“0x”）
		<property name="privKey" value="bcec428d5205abe0f0cc8a734083908d9eb8563e31f943d760786edf42ad67dd" />
		<!--God账户-->
		<property name="account" value="0x776bd5cf9a88e9437dc783d6414bccc603015cf0" />
		<property name="outPutpath" value="./output/" />
	</bean>
```

权限控制接口命令参考如下

```
./web3sdk ARPI_Model 
./web3sdk PermissionInfo 
./web3sdk FilterChain addFilter name1 version1 desc1 
./web3sdk FilterChain delFilter num 
./web3sdk FilterChain showFilter 
./web3sdk FilterChain resetFilter 
./web3sdk Filter getFilterStatus num 
./web3sdk Filter enableFilter num 
./web3sdk Filter disableFilter num 
./web3sdk Filter setUsertoNewGroup num account 
./web3sdk Filter setUsertoExistingGroup num account group 
./web3sdk Filter listUserGroup num account 
./web3sdk Group getBlackStatus num account 
./web3sdk Group enableBlack num account 
./web3sdk Group disableBlack num account 
./web3sdk Group getDeployStatus num account 
./web3sdk Group enableDeploy num account 
./web3sdk Group disableDeploy num account 
./web3sdk Group addPermission num account A.address fun(string) 
./web3sdk Group delPermission num account A.address fun(string) 
./web3sdk Group checkPermission num account A.address fun(string) 
./web3sdk Group listPermission num account 
```

## （六）FAQ

### 6.1、使用工具包生成合约Java Wrap代码时会报错。
    从 https://github.com/FISCO-BCOS/web3sdk 下载代码（Download ZIP），解压之后，目录名称将为：web3sdk-master，运行gradle命令后生成的工具包中，/dist/apps目录下生成的jar包名称为web3sdk-master.jar，导致和/dist/bin/web3sdk中配置的CLASSPATH中的配置项$APP_HOME/apps/web3sdk.jar名称不一致，从而导致使用工具包生成合约Java Wrap代码时会报错。
### 6.2、工具目录下，dist/bin/web3sdk运行会出错。
可能是权限问题，请尝试手动执行命令：chmod +x  dist/bin/web3sdk。
### 6.3、java.util.concurrent.ExecutionException: com.fasterxml.jackson.databind.JsonMappingException: No content to map due to end-of-input
出现此问题请按顺序尝试来排查：<br />
1、检查配置是否连接的是channelPort，需要配置成channelPort。<br />
2、节点listen ip是否正确，最好直接监听0.0.0.0。<br />
3、检查channelPort是否能telnet通，需要能telnet通。如果不通检查网络策略，检查服务是否启动。<br />
4、服务端和客户端ca.crt是否一致，需要一致。<br />
5、[FISCO-BCOS中client.keystore 的生成方法](https://github.com/FISCO-BCOS/web3sdk/issues/20)

