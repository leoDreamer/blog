<!--
author: leo
date: 2017-08-08
title: 使用RSA实现前端公钥加密后端私钥解密
tags: crypto,rsa,javascript
category: frontend
status: publish
summary: 使用RSA实现前端公钥加密后端私钥解密
-->

# 项目中在用户登录时需要进行用户名和密码加密,这里选用了RSA非对称加密的方式.

- 公钥私钥:OpenSSL的公钥私钥(Node crypto模块限制)
- 前端: jsencrypt库加密
- 后端: Node crypto模块

使用openssl生成公钥私钥
---------------
linux生成公钥私钥命令:
```javascript
genrsa -out rsa_private_key.pem 1024 // 生成1024位私钥
pkcs8 -topk8 -inform PEM -in rsa_private_key.pem -outform PEM –nocrypt // 把RSA私钥转换成PKCS8格式
rsa -in rsa_private_key.pem -pubout -out rsa_public_key.pem // 生成对应公钥
```
这里已经生成了公钥和私钥,公钥私钥的使用可以分为两种方式
- 在使用时使用`fs.readFileSync()`来读取
- 直接将公私钥拷贝出来放到配置文件中(私钥要保证正确的换行格式,否则crypto不能正确的使用私钥,可以在每行末尾添加`\n\`来确保格式正确)

前端使用jsencrypt库加密
----------------
这个库可以使用模块加载的方式使用,下面的代码也是使用这种方式,当然也可以在页面中通过`<script>`标签引入,这种方式的使用请参考库的具体[文档][1]
```javascript
const JSEncrypt = require("jsencrypt"); // 引入模块
const str = "i will be encrypto";
const encrypt = new JSEncrypt.JSEncrypt(); // 实例化加密对象
encrypt.setPublicKey(publicKey); // 设置公钥
const encryptoPasswd = encrypt.encrypt(str) // 加密明文
```

服务端使用Node crypto模块解密
--------------------
解密函数实现
```javascript
function rsaDecrypt(text) {
    const privateKey = config.privateKey;
    let textBuffer, decryptText;
    try {
        textBuffer= new Buffer(text, "base64"); // jsencrypt 库在加密后使用了base64编码,所以这里要先将base64编码后的密文转成buffer
        decryptText= crypto.privateDecrypt({
            key: new Buffer(privateKey), // 如果通过文件方式读入就不必转成Buffer
            padding: constants.RSA_PKCS1_PADDING // 因为前端加密库使用的RSA_PKCS1_PADDING标准填充,所以这里也要使用RSA_PKCS1_PADDING 
        }, textBuffer).toString();
    } catch (err) {
        throw (new CommonError("errorMsg", errorCode));
    }
    return decryptText;
}
```

[1]: https://www.npmjs.com/package/jsencrypt