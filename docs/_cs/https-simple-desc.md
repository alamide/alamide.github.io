---
layout: post
title: Https 的简单理解
categories: network
tags: network https
date: 2023-04-27
---
最近在学习 `Nginx` 时，学到了 `Https` 的配置，简单学习了 `Https` 出现的原因，及加密的原理，这里用自己的理解简单描述一下。
<!--more-->

网络明文传输的问题？

由于网络传输的过程成会经过很多路由器、交换机，传输的信息很可能会被拦截、偷窥，明文传输会造成信息泄漏，甚至被恶意篡改。例如：
用户 A 通过网络向服务器提交向 B 用户转账 5000 元的请求，这个信息很可能会被篡改为向 C 用户转账 500000 元，这个过程是不安全的。

![明文传输](/assets/imgs/https-01.jpg)


我们可能会想到进行数据加密，让拦截者即使拦截到请求，也破解不出传输的具体内容。如何加密呢？这个加密算法最基础的要求是什么？是我们加密的内容，服务器能正确的解密。
服务器要知晓我们的具体加密规则。难吗？很难，难在这个加密规则如何传递，明文传递肯定是行不通的。加密传递？又回到原点了，加密的规则如何传递？明文传递肯定是不行的
，加密传递…… 彻底进入死循环了，无解。

传统的加密方式是对称加密，也即加密和解密的规则一致，稍微的区别的是解密方和加密方执行的是相反的操作。最常见的对称加密是密码本通信，
早期战争中的通信，就是这类加密方式。通信双方拥有同样的密码本，发送信息时，首先将要传输的报文每个字先在密码本上查询到具体位置，如“是”这个字在密码本的第 18 页 18 行 18 列，就编码为 `181818`。接收方再依据 `181818` 去密码本解密出传输的内容为 “是”。这就完成了数据的加密解密，加密解密算法一致，解密加密双方互为逆操作。

整个信息传输的过程中，密码本是最为重要的部分。通信双方的密码本如何指定呢？先是通信双方会面，现场指定密码本，这是最安全的方式。不能通过报文的方式指定密码本，
因为不能保证报文在传输过程中没有被敌对势力偷窥。当然通信双方为了避免密码被破解也会定期修改密码本，可能通过加密的报文，也可能事先约定。
但是使用加密的报文修改密码本的前提还是双方通信之初会过面，指定过密码本。这前提无法避免，否则通信不可信。

这种加密方式适合网络通信吗？很遗憾不适合。上面通信可信的前提是什么？是双方会过面，现场指定密码本。很显然我们不可能和每个服务提供商会面。当然也有特例，有的服务商会给一个实体密码器，密码器有电子显示屏，显示屏上会显示一串数字，定期变更，那就是我们双方通信的密码本。涉及需要加密的操作时，输入密码器给定的数字，然后使用这串数字作为因子进行通信加密。这种方式成本高、繁琐、低效，无法普及。我们使用的网络服务有很多，不可能用一家服务就去服务提供商处申请一份密码本，不现实。怎么办，如果解决不了加密的问题，互联网很难大规模推广，陷入绝境了吗？

1976年，`W.Diffie` 和 `M.Hellman` 两位科学家提出了“非对称密码体制即公开密钥密码体制”的概念，之后研究者们研究出了具体的非对称加密算法。非对称加密使用配对的密钥，公钥和私钥，公钥加密的内容只有私钥能解密，公钥无法解密，私钥加密的内容公钥可以解密。用户拥有公钥，服务器拥有私钥。

类似于，一个只有锁却没有钥匙的箱子，服务器给用户发送一个箱子，箱子有锁但没有锁上，用户拿到箱子后，把要发送的信件放在箱子里，把锁锁上，再把箱子发送给服务器。只要锁上，除了拥有钥匙的人，任何人都无法打开，包括用户自己。用户知道箱子里是什么，但就是打不开，很神奇的加密方式。有了这种加密方式，我们的通信安全可性就可以提升了。用户正式通信之前，先去服务器下载公钥，然后用公钥加密要传输的内容，只要使用公钥加密后，除了私钥拥有者，任何人都无法解密传输的具体内容，很强。

那么这就足够安全了吗？不。这个过程看似安全，但有一个漏洞。

还是用上面的没加锁的箱子来举例吧。没加锁的箱子，想法是不错，但不能保证服务器发送的和用户收到的是同一个箱子。中间人可以自己伪造一个箱子，再把伪造的箱子发给用户，用户无法分辨真伪，仍然按照流程，写信件，上锁。中间人收到解锁，篡改内容，再把篡改的信息放在原版箱子里上锁，整个过程神不知鬼不觉。问题出在哪了？问题出在用户无法辨别箱子的真伪。

怎么办？找第三方专业鉴定机构鉴定比较靠谱。如何鉴定？把箱子邮寄到鉴定机构吗？显然不行，返回的鉴定结果同样会被篡改，只能亲自去机构鉴定。所以这里问题的关键就在鉴定机构和亲临了。

同样网络通信中公钥也可能会被替换，需要一个专门的鉴定机构来鉴定我们的公钥是否合法，这个机构叫做 `CA(Certificate Authority)` 。为了防止鉴定的结果被篡改，操作系统中会内置一些权威的 `CA` 证书，这些证书是内置的，不会经由网络传递。

当与服务器通信时，服务器会下发权威机构颁发的证书，证书含有公钥及域名等信息，证书无法被篡改（如果被篡改，本地认证系统会解析失败）。浏览器收到证书后会使用操作系统内置的认证系统认证。认证的过程可能是这样，浏览器输入 `https://www.alamide.com` ，服务器下发认证过的证书，内置认证系统解析出证书含有的公钥及公钥绑定的域名信息，解析的域名为 `www.alamide.com` ，与浏览器输入域名一致，证书合法，允许进一步请求。如果解析的域名与浏览器输入的域名不一致，那么返回的内容被篡改，请求非法。

上面使用的非对称算法有漏洞吗？有。虽然公钥加密的内容无法被其他人解密，即提交给服务器的内容无法被解密和篡改，但是服务返回的即被私钥加密的内容，任何拥有
公钥的用户都可以破解。虽然返回的内容不可篡改，但是可以被偷窥。而且整个加密过程很消耗资源，比对称加密消耗的资源多的多。

继续前进吧，目前为止我们可信的通信是什么？用户向服务器发送的信息，这条单向通信是绝对可信的。用户向服务器发送公钥加密的信息，除了拥有配对私钥的服务器，任何人都无法破解。好了，这就足够了，还记得之前提到的对称算法吗？网络通信无法使用对称算法的原因是什么？是无法通过网络传递双方使用的加解密规则。现在有了非对称算法，我们可以解决这个问题了。具体的对称加密规则由用户制定，用户制定加密规则，将规则使用公钥加密，发送给服务器，服务器使用私钥解密，得到具体的规则，整个流程安全可信。后续双方通信使用指定的对称加密规则通信，高效、安全、可靠、完美。

这就是整个 `Https` 的理解了，最后说一句，非对称算法真是个伟大的发明，正是有了它，才有了今天繁荣的互联网。 
