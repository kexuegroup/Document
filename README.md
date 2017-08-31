# Document
科学集团接口文档
1. 概述

1.1 关于科学集团

科学集团（kexuegroup.com）2015年9月9日正式上线运营，是国内专业的IT业务一站式技术服务商，致力于软件开发和产品设计。

1.2 设计背景

科学集团与优质的互联网公司合作，通过技术手段对接，为企业一站式技术服务。

1.3 文档范围

此文档面向接口开发者介绍科学集团的接口协议、如何实现、相应示例，以及相关注意事项。

1.4 阅读对象

科学集团平台、项目经理、产品经理、开发人员、测试人员、运维人员。

2. 总体说明

2.1 接口调用

科学集团通过HTTP POST的方式调用平台实现的接口

平台提供单一入口的URL(除了bindUser和loginUser接口),用来接收调用请求

调用方式

curl -x POST -d '<data>' '(http|https)://<url>'
2.2 数据格式

无论是请求还是响应，数据格式都是JSON，且数据都须满足一定的Schema（模式）,Schema定义如下

{
  "data": "string, required",
  "timestamp": "int64, required",
  "nonce": "string, required",
  "signature": "string, required"
}
data原始数据模式定义

{
	"service": "string, required, 用于区分不同的接口调用",
	"body": "object, optional, 请求数据，JSON格式，具体参考接口定义章节"
}
Schema各字段解释

字段名	类型	必填	描述
data	string	Yes	请求或响应数据，该字段为加密后的数据（加密算法请参考数据安全性章节），加密前的原始数据是JSON格式（参考data原始数据模式定义）
timestamp	int64	Yes	时间戳，数据发送的当前时间，在数据加密和校验时有重要作用
nonce	string	Yes	随机字符串，用于data的签名，如果data为空，该字段也为空
signature	string	Yes	对data的签名，如果data为空，该字段也为空，具体签名算法见2.3.4章节
错误响应

如果程序逻辑出错，需要返回错误的响应，错误响应的HTTP Status Code定义为500，错误的响应内容要有统一的格式，定义如下

{
  "code": "int, 错误代码，具体见接口定义章节",
  "message": "错误描述"
}
注意：错误响应内容无需加密

其他说明

字符编码使用UTF-8
协议中所有时间参数（如registerAt, investAt等等）格式均为2006-01-02 15:04:05
2.3 数据安全

为保证数据安全性，对每个接口的调用（无论是请求还是响应）都需要加密，下面具体的描述了加密的细则

2.3.1 时间戳

这里指2.2章节中提到的协议中的时间戳。

程序逻辑在处理接收到的请求或响应时，需要根据该时间戳校验协议的时效性，如果

当前时间戳 - 协议时间戳 > 5分钟
则需要拒绝该请求或者响应。

2.3.2 密钥

科学集团会为每个平台分配一个密钥（Secret），为了保证密钥的安全性，在每次请求时都会生成一个请求密钥，使用请求密钥对数据进行加密。

// 请求密钥的生成方法为
ReqKey = MD5(Secret + Timestamp)
变量名	描述
ReqKey	请求密钥，用于加密请求或响应数据
Secret	分配密钥，谷米财富为每个平台分配
Timestamp	时间戳，单位秒，数据发送的当前时间
2.3.3 加解密

默认使用AES方式对数据进行加密，数据加密前会对数据做一些简单的处理。

使用Data表示真实的明文数据

// 加密算法
RawData = RandomStr + DataLength + Data + PlatId;
EncryptData = AESEncrypt(RawData, ReqKey);
变量名	描述
RandomStr	16个字节的随机字符串
DatadLength	数据长度，固定4个字节
Data	明文数据
PlatId	科学集团分配给平台的ID
RawData	加密前的数据
EncryptData	加密后的数据
ReqKey	请求密钥
2.3.4 数据签名

为了防止数据被篡改，对加密后的数据进行签名，接收到数据后会需要校验签名。

// 数据签名算法
Signature = SHA1(sort(EncryptData, Token, Timestamp, Nonce));
变量名	描述
EncryptData	加密后的数据
Token	科学集团分配给平台的Token
Timestamp	时间戳，对应2.2中的timestamp
Nonce	随机字符串，对应2.2中的timestamp
Signature	签名

3. 接口描述

3.1 创建新账户

科学集团通过该接口，为科学集团用户在合作平台创建一个新的帐号。

Service

service=createUser

Request

{
  "username": "string, 科学集团用户名",
  "mobile": "string, 手机",
  "email": "string, 电子邮箱(可选)",
  "idCard": {
    "number": "string, 身份证号码",
    "name": "string, 实名"
  }, 
  "bankCard": {
	  "number": "string, 卡号",
	  "bank": "string, 银行名称",
	  "branch": "string, 支行名",
	  "province": "string, 省份",
	  "city": "string, 城市",
  },
  "tags": "array, 标签 (wap,pc)"
}
Response

{
  "username": "string, required, 科学集团用户名",
  "usernamep": "string, required, 平台用户名",
  "registerAt": "datetime, required, 平台注册时间",
  "bindAt": "datetime, required, 绑定科学集团时间",
  "bindType": "enum, required, 0:表示科学集团带来的新用户",
  "salt": "string, 用于鉴权校验,该账户的8位长度密钥",
  "tags": "array, 标签"
}
如果科学集团用户重复创建，同样视为成功，返回对应的绑定信息。
新注册的用户如果在平台未发现重复用户名，则默认为kxjt+手机号。
如果平台用户名usernamep是可以做修改的，则回传的usernamep改成kxjt+入库的绑定关系主键id,确保唯一性。
Errors

code	message
1001	手机号已占用
1002	邮箱已占用(目前邮箱不是必传项，可不做验证)
1003	身份证已占用
1004	用户名已占用
3.2 关联老账户

科学集团通过该接口，将用户在科学集团的帐号跟在合作平台的帐号关联起来。

Service

service=bindUser

Request

{
  "username": "string, 科学集团用户名",
  "telephone": "string, 手机",
  "email": "string, 电子邮箱(可选)",
  "idCard": {
    "number": "string, 身份证号码",
    "name": "string, 实名"
  },
  "bankCard": {
	  "number": "string, 卡号",
	  "bank": "string, 银行名称",
	  "branch": "string, 支行名",
	  "province": "string, 省份",
	  "city": "string, 城市",
  },
  "tags": "array, 标签 (wap,pc)"
}
Response

合作平台接收到该请求后，需要将用户带到科学集团与合作平台的专属绑定页面，验证用户身份。验证成功后，完成绑定。 用户授权绑定成功后平台需同步回调科学集团的接口URL.

 线上地址：http://www.kexuegroup.com
 测试地址：http://test.kexuegroup.com:8080/callback
此时请求的URL 所需参数为：

Service

service=bindUser

Request

{
  "username": "string, required, 科学集团用户名",
  "usernamep": "string, required, 平台用户名",
  "registerAt": "datetime, required, 平台注册时间",
  "bindAt": "datetime, required, 绑定科学集团时间",
  "bindType": "enum, required, 1:表示平台已有用户",
  "salt": "string, 用于鉴权校验,该账户的8位长度密钥",
  "tags": "array, 标签"
}
请求回调的Method 为 POST 参数为 data=xxx&nonce=xxx&signature=xxx&timestamp=12345643&appId=xxxx

Response

科学集团会输出提示用户绑定成功的页面

3.3 单点登录

科学集团与合作平台帐户关联的用户，通过该接口登录到合作平台。

Service

service=login

Request

{
  "username": "string, 科学集团用户名",
  "usernamep": "string, 合作平台用户名",
  "salt": "string, 用于鉴权校验,该账户的8位长度密钥",
  "bid": "string, 标的ID，跳转到标的购买页，home为首页，account为个人中心",
  "type": "登录类型，0:PC，1:WAP"
}
Response

该接口为非应答接口，而是平台设置该用户的登录态，并进行浏览器跳转
