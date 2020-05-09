---
title: 反序列化漏洞 
description: 总结一下序列化漏洞利用、绕过及防御姿势。
categories:
 - web vul
tags:
 - serialization
---

## 概述

1. 序列化就是把对象的状态信息转换为字节序列(即可以存储或传输的形式)过程，反序列化即逆过程，由字节流还原成对象。3种格式：
* 默认（二进制）
* xml（ XStream）
* json（ gson(google维护)、jackson、fastjson（阿里巴巴））
2. 一个类的对象要想序列化成功，必须满足两个条件：
* 该类必须实现 java.io.Serializable 或Externalizable接口(Externalizable接口继承自 Serializable接口)
* 该类的所有属性必须是可序列化的
3. 特殊变量
* transient变量不会被序列化。
* static静态变量不是对象状态的一部分，因此它不参与序列化。
* final变量将直接通过值参与序列化。
4. 其他
* 重写序列化类的readObject方法，该方法在反序列化时会被自动调用
* JAVA序列化的机制是通过判断类的serialVersionUID来验证的版本一致的


## 漏洞利用
* **信息泄露**

被序列化的对象包含敏感数据的话，存在信息泄露的风险。

* **伪造**

因为反序列化过程中没有对数据进行校验,可以篡改被系列化的二进制流，来进行数据伪造。

* **拒绝服务**

当篡改的数据不符合序列化对象的格式要求时候，可能会导致在反序列化对象的过程中抛出异常，从而拒绝服务。

* **命令执行**

当反序列化对象时的运行环境中存在有漏洞的 jar 包（比如 commons Collections 低版本），攻击者通过构造恶意数据流可以达到命令执行的效果。


## 漏洞挖掘

### 黑盒

* **报文分析**

HTTP请求中的参数，cookies以及Parameters（json，xml，表单）。在HTTP报文中，大多以base64传输（ rO0AB ）
在TCP报文中，一般二进制流方式传输（ aced0005 ）

* **RMI协议**

Java RMI的传输100%基于反序列化，Java RMI的默认端口是1099端口（jboss 1090则是RMI服务端口）
RMI命令执行有两个条件：
1) RMI 对外开放；
2) 系统环境中存在有漏洞 Jar 包。

* **JMX 同样用于处理序列化对象**

* **自定义协议 用来接收与发送原始的java对象**

### 白盒
* 观察实现了Serializable接口的类是否存在问题
* 观察重写了readObject方法的函数逻辑是否存在问题
* 查看项目工程中是否引入可利用的commons-collections 3.1、commons-fileupload 1.3.1等第三方库
* 查找Java RMI服务
1. LocateRegistry.getRegistry （默认端口 1099）
2. LocateRegistry.createRegistry

```
ObjectInputStream.readObject
ObjectInputStream.readUnshared
XMLDecoder.readObject
Yaml.load
XStream.fromXML
ObjectMapper.readValue
JSON.parseObject
```

## 漏洞防御
* **通用做法**
1. 对序列化的流数据进行加密
2. 在传输过程中使用 TLS 加密传输
3. 对序列化数据进行完整性校验
* **针对信息泄露**
1. 使用 transient 标记敏感字段，这样敏感字段将不进行序列化。
* **针对序列化对象属性的篡改**
1. 可以通过实现 validateObject 方法来进行对象属性值的校验。
* **类黑白名单校验**
1. 在反序列化过程开始前，对 className 进行白名单校验，不符合白名单的类不进行反序列化操作（比如Runtime ）。校验类名方法参见[Java反序列化漏洞之殇](https://www.cnblogs.com/studyskill/p/9207117.html)
* **禁止 JVM 执行外部命令 Runtime.exec**
* **针对RMI**
1. 使 RMI 只开放在内网
2. 升级本地的 jar 包
* **升级漏洞版本**

## 0day应急方案

1. 手动添加类名校验（黑白名单）----Fastjson autoType安全黑名单，手动添加
2. 在危险jar包中直接删除危险class ---- 需谨慎
3. NG上进行拦截（基于特征）----性能可能会受影响
4. 禁止 JVM 执行外部命令 Runtime.exec


