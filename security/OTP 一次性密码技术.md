
## 编写目的

本文档不涉及细节的技术原理，只是系统性地介绍OTP，给大家在做多因子身份认证或授权等功能的设计上提供些参考

## OTP简介

什么是OTP，参考百度百科： [一次性密码_百度百科
(baidu.com)](https://baike.baidu.com/item/%E4%B8%80%E6%AC%A1%E6%80%A7%E5%AF%86%E7%A0%81/10650649?fromtitle=OTP&fromid=1406545&fr=aladdin) 

一次性密码（英语：One Time Password，简称OTP），又称动态密码或单次有效密码，是指计算器系统或其他数字设备上只能使用一次的密码，有效期为只有一次登录会话或交易。OTP 避免了一些与传统基于（静态）密码认证相关系的缺点；一些实现还纳入了[双因素认证](https://baike.baidu.com/item/%E5%8F%8C%E5%9B%A0%E7%B4%A0%E8%AE%A4%E8%AF%81/1612350?fromModule=lemma_inlink)，确保单次有效密码需要访问一个人有的某件事物（如内置 OTP 计算器的小钥匙挂件设备）以及一个人知道的某件事物（如 PIN）。 [1]

相对于静态密码，OTP 最重要的优点是它们不容易受到[重放攻击](https://baike.baidu.com/item/%E9%87%8D%E6%94%BE%E6%94%BB%E5%87%BB/2229240?fromModule=lemma_inlink)(replay attack)。这意味着管理记录已用于登录到服务或进行交易的 OTP 的潜在入侵者将无法滥用它，因为它将不再有效。第二个主要优点是，使用多个系统相同（或类似）密码的用户，如果其中一个密码被攻击者获得，不是对所有的系统都容易变得脆弱。许多 OTP 系统也旨在确保会话不能轻易被截获或没有前一个会话期间产生不可预测数据的知识模拟，从而进一步减少攻击面。

OTP 已被作为传统密码一个可能的替代以及增强方式讨论。不利的是，OTP 人类难以记忆。因此，它们需要额外的技术来运作。

一般的静态密码在安全性上容易因为[木马](https://baike.baidu.com/item/%E6%9C%A8%E9%A9%AC/0?fromModule=lemma_inlink)与键盘侧录程序等而被窃取，而只要花上相当程度的时间，也有可能被[暴力破解](https://baike.baidu.com/item/%E6%9A%B4%E5%8A%9B%E7%A0%B4%E8%A7%A3/0?fromModule=lemma_inlink)。为了解决一般密码容易遭到破解情况，因此开发出一次性密码的解决方案。

## OTP体验

看过OTP的简介，我们已经有概念上的认识，现在通过一个简单的Demo感受下OTP是怎么使用的

1.  APP安装：

android版安装地址： <https://f-droid.org/repo/org.fedorahosted.freeotp_17.apk>

iOS 直接appstore里搜索即可: [FreeOTP Authenticator on the App Store
(apple.com)](https://apps.apple.com/us/app/freeotp-authenticator/id872559395)

2.  演示Demo: [OTPDemo.exe](https://wiki.mchz.com.cn/download/attachments/31857999/OTPDemo.exe?version=1&modificationDate=1643251391552&api=v2)

3.  Demo是个windows程序，是个基本的注册和用OTP登录的例子

- 注册：输入用户名，颁发机构，和密码，
  颁发机构填hzmc。注册后会弹出一个二维码，用FreeOTP扫描，扫描后相当于把注册的身份信息和APP绑定了（软件绑定）。这时候在FreeOTP会增加一个token， 二维码相当于传统的密码，非常重要，不可泄漏

- 登录：输入注册的用户名和FreeOTP的6位数字密码。你会发现这个数字30秒会更新一次，代表数字密码的时效性最多只有30秒

## OTP验证原理

A端：服务器端； B端：APP端

A端生成一个随机密钥，把该密钥和某个账号做关联，并生成一个二维码，二维码包含了账号和密钥信息

B端扫描A端生成的二维码，B端获取了账号和密钥信息

A，B端采用相同的算法，相同的密钥，在同一个时间段内会生成一个相同的随机数字，因此可以根据这个随机数字来做身份认证

 

## OTP常用场景

授权：B用户做某操作需要A用户授权，A只需要将动态数字密码告知B，而不需要将密码告知B，降低了密码管理和泄漏的风险

双重因子身份认证：用户做某操作需要输入密码，还需要输入动态数字密码，大大加强了安全性

 
## 附录

TOTP - 基于时间的一次性密码方案

前提和限制条件：

1\. 时间服务器，时间正确。

The prover (e.g., token, soft token) and verifier (authentication or
validation server) MUST know or be able to derive the current Unix time
(i.e., the number of seconds elapsed since midnight UTC of January 1,
1970) for OTP generation.

The prover and verifier MUST use the same time-step value X.

the HOTP algorithm is based on the HMAC-SHA-1 algorithm (as specified in
\[[RFC2104](https://datatracker.ietf.org/doc/html/rfc2104)\]) and
applied to an increasing counter value representing the message in the
HMAC computation. Basically, the output of the HMAC-SHA-1 calculation is
truncated to obtain user-friendly values: HOTP(K,C) =
Truncate(HMAC-SHA-1(K,C)) where Truncate represents the function that
can convert an HMAC-SHA-1 value into an HOTP value. K and C represent
the shared secret and counter value; see
\[[RFC4226](https://datatracker.ietf.org/doc/html/rfc4226)\] for
detailed definitions.

The implementation of this algorithm MUST support a time value T larger
than a 32-bit integer when it is beyond the year 2038.

参考内容：

RFC6238 TOTP: Time-Based One-Time Password Algorithm

<https://datatracker.ietf.org/doc/html/rfc6238>

RFC4226  HOTP: An HMAC-Based One-Time Password Algorithm

<https://datatracker.ietf.org/doc/html/rfc4226>
