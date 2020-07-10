# NTLM认证协议与SSP(上)

内容参考原文链接：http://davenport.sourceforge.net/ntlm.html
翻译人：rootclay（香山）https://github.com/rootclay

# 说明 
本文是一篇NTLM中高级进阶进阶文章，文中大部分参考来自于[Sourceforge](http://davenport.sourceforge.net/ntlm.html)，原文中已经对NTLM讲解非常详细，在学习的过程中思考为何不翻译之，做为学习和后续回顾的文档，并在此基础上添加自己的思考，因此出现了这篇文章，在翻译的过程中会有部分注解与新加入的元素，后续我也会在Github对此文进行持续性的更新NTLM以及常见的协议中高级进阶并计划开源部分协议调试工具，望各位issue勘误。

# 摘要

本文旨在以中级到高级的详细级别描述NTLM身份验证协议(authentication protocol)和相关的安全支持提供程序功能(security support provider functionality)，作为参考。希望该文档能发展成为对NTLM的全面描述。目前，无论是在作者的知识还是在文档方面，都存在遗漏，而且几乎可以肯定的说本文是不准确的。但是，该文档至少应能够为进一步研究提供坚实的基础。本文提供的信息用作在开放源代码jCIFS库中实现NTLM身份验证的基础，该库可从 http://jcifs.samba.org获得。本文档基于作者的独立研究，并分析了[Samba](http://www.samba.org/)软件套件。

# 什么是NTLM？
NTLM是一套身份验证和会话安全协议，用于各种Microsoft网络协议的实现中（注：NTLM为嵌套协议，被嵌套在各种协议，如HTTP、SMB、SMTP等协议中），并由NTLM安全支持提供程序（"NTLMSSP"）支持。NTLM最初用于DCE/RPC的身份验证和协商（Negotiate），在整个Microsoft系统中也用作集成的单点登录机制（SSO）。可以认为NTLM是HTTP身份验证的技术栈的一部分。同时，它也用于SMTP，POP3，IMAP（Exchange的所有部分），CIFS/SMB，Telnet，SIP以及其他可能的Microsoft实现中。

NTLM Security Support Provider在Windows Security Support Provider（SSPI）框架内提供身份验证（Authentication），完整性（Signing）和机密性（Sealing）服务。SSPI定义了由支持提供程序(supporting providers)实现的一组核心安全功能集；NTLMSSP就是这样的提供程序(supporting providers)。SSPI定义并由NTLMSSP实现以下核心操作：

1. 身份验证（Authentication）-NTLM提供了质询响应(challenge-response)身份验证机制，在这种机制中，客户端无需向服务器发送密码即可证明其身份。(注：我们常说的PTH等等操作都发生在这里。)

2. 签名（Signing）-NTLMSSP提供了一种对消息应用数字"签名"的方法。这样可以确保已签名的消息未被（偶然或有意地）修改，并且确保签名方知道共享机密。NTLM实现了对称签名方案（消息身份验证码或MAC）；也就是说，有效的签名只能由拥有公共共享Key的各方生成和验证。

3. Sealing（注：找不到合适的词来翻译，可以理解为加密封装）-NTLMSSP实现了对称Key加密机制，该机制可提供消息机密性。对于NTLM，Sealing还意味着签名（已签名的消息不一定是已Sealing的，但是所有已Sealing的消息都已签名）。

Kerberos已取代NTLM成为基于域的方案的首选身份验证协议。但是，Kerberos是需要有受信任的第三方方案，不能在不存在受信任的第三方的情况下使用。例如，成员服务器（不属于域的服务器），本地帐户以及对不受信任域中资源的身份验证。在这种情况下，NTLM仍然是主要的身份验证机制（可能会持续很长时间）。

# NTLM通用术语
在开始深入研究之前，我们需要定义各种协议中使用的一些术语。
由于翻译的原因我们大概约定一些术语：
协商 = Negotiate
质询 = Challenge
响应 = Response
身份验证 = Authentication
签名 = Signing

NTLM身份验证是一种质询-响应方案，由三个消息组成，通常称为Type 1（协商），Type 2（质询）和Type 3（身份验证）。它基本上是这样的：

1. 客户端向服务器发送Type 1消息。它主要包含客户端支持和服务器请求的功能列表。
2. 服务器以Type 2消息响应。这包含服务器支持和同意的功能列表。但是，最重要的是，它包含服务器产生的challenge。
3. 客户用Type 3消息答复质询。其中包含有关客户端的几条信息，包括客户端用户的域和用户名。它还包含对Type 3 challenge 的一种或多种响应。
Type 3消息中的响应是最关键的部分，因为它们向服务器证明客户端用户已经知道帐户密码。

认证过程建立了两个参与方之间的共享上下文；这包括一个共享的Session Key，用于后续的签名和Sealing操作。

在本文档中，为避免混淆（无论如何，尽可能避免混淆），将遵循以下约定（除了“NTLM2会话响应”身份验证（NTLMv1身份验证的一种变体，与NTLM2会话安全性结合使用）的特殊情况外。）：

* 在讨论身份验证时，协议版本将使用"v编号"。例如" NTLM v1身份验证"。
* 在讨论会话安全性（签名和Sealing）时，"v"将被省略；例如" NTLM 1会话安全性"。 


" short "是一个低位（little-endian，即小端，这在实现协议库时会有小差别，不写代码可以忽略）字节的16位无符号值。例如，表示为short的十进制值"1234" 将以十六进制物理布局为" 0xd204 "。

" long "是32位无符号小尾数。以十六进制表示的长整数十进制值" 1234 "为" 0xd2040000 "。

Unicode字符串是一个字符串，其中每个字符都表示为一个16位的little-endian值（16位UCS-2转换格式，little-endian字节顺序，没有字节顺序标记，没有空终止符）。Unicode中的字符串" hello"将以十六进制表示为" 0x680065006c006c006f00 "。

OEM字符串是一个字符串，其中每个字符都表示为本地计算机的本机字符集（DOS代码页）中的8位值。没有空终止符。在NTLM消息中，OEM字符串通常以大写形式显示。OEM中的字符串" HELLO"将用十六进制表示为" 0x48454c4c4f "。

"安全缓冲区"（security buffer）是用于指向二进制数据缓冲区的结构。它包括：

1. 一个short的内容，包含缓冲区内容的长度（以字节为单位）（可以为零）。
2. 一个short的信息，包含为缓冲区分配的空间（以字节为单位）（大于或等于长度；通常与长度相同）。
3. 一个long，包含到缓冲区开头的偏移量（以字节为单位）（从NTLM消息的开头）。

因此，安全缓冲区" 0xd204d204e1100000 "将被读取为：
Length: 0xd204 (1234 bytes)
Allocated Space: 0xd204 (1234 bytes)
Offset: 0xe1100000 (4321 bytes)

比如下图表示的数据就是一个安全缓冲区：

![-w598](https://p1.ssl.qhimg.com/t0189cb3609b67c976a.jpg)


如果您从消息中的第一个字节开始，并且向前跳过了4321个字节，那么您将位于数据缓冲区的开头。您将读取1234个字节（这是缓冲区的长度）。由于为缓冲区分配的空间也是1234字节，因此您将位于缓冲区的末尾。

# NTLM Message Header Layout（NTLM消息头）
现在，我们准备看一下NTLM身份验证消息头的布局。

所有消息均以NTLMSSP签名开头，该签名（适当地）是以null终止的ASCII字符串" NTLMSSP"（十六进制的" 0x4e544c4d53535000 "）。

![-w572](https://p0.ssl.qhimg.com/t014edb771996e94f6e.jpg)


下一个是包含消息Type （1、2或3）的long。例如，Type 1消息的十六进制Type 为" 0x01000000 "。

![-w622](https://p2.ssl.qhimg.com/t01de539edb2c0528f0.jpg)


这之后是特定于消息的信息，通常由安全缓冲区和消息Flags组成。

## NTLM Flags
消息Flags包含在头的位域中。这是一个 long，其中每个位代表一个特定的Flags。这些内容中的大多数出现在特定消息中，但是我们将全部在这里介绍它们，可以为其余的讨论建立参考框架。下表中标记为"unidentified"或"unknown"的Flags暂时不在作者的知识范围之内。

注：下表第一次使用中文描述，之后不再使用英文描述。

| flag        | 名称           | 描述                                                                                                                                                                                 |
| ---------- | ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 0x00000001 | Negotiate Unicode    | 指示在安全缓冲区数据中支持使用Unicode字符串。                                                                                                                       |
| 0x00000002 | Negotiate OEM        | 表示支持在安全缓冲区数据中使用OEM字符串。                                                                                                                           |
| 0x00000004 | Request Target     | 请求将服务器的身份验证领域包含在Type 2消息中。                                                                                                                     |
| 0x00000008 | unknown           | 该Flags的用法尚未确定。                                                                                                                                                      |
| 0x00000010 | 	Negotiate Sign     | 指定客户端和服务器之间的经过身份验证的通信应带有数字签名（消息完整性）。                                                                           |
| 0x00000020 | Negotiate Seal     | 指定应该对客户机和服务器之间的已验证通信进行加密（消息机密性）。                                                                                       |
| 0x00000040 | 	Negotiate Datagram Style | 指示正在使用数据报认证。                                                                                                                                                   |
| 0x00000080 | Negotiate Lan Manager Key | 指示应使用Lan ManagerSession Key来签名和Sealing经过身份验证的通信。                                                                                                 |
| 0x00000100 | Negotiate Netware     | 该Flags的用法尚未确定。                                                                                                                                                      |
| 0x00000200 | Negotiate NTLM Key      | 指示正在使用NTLM身份验证。                                                                                                                                                  |
| 0x00000400 | 	Negotiate Only NT （unknown）           | 该Flags的用法尚未确定。                                                                                                                                                      |
| 0x00000800 | Negotiate Anonymous     | 客户端在Type 3消息中发送以指示已建立匿名上下文。这也会影响响应字段（如" 匿名响应 "部分中所述）。                                       |
| 0x00001000 | Negotiate OEM Domain Supplied  | 客户端在Type 1消息中发送的消息，指示该消息中包含客户端工作站具有成员资格的域的名称。服务器使用它来确定客户端是否符合本地身份验证的条件。 |
| 0x00002000 | Negotiate OEM Workstation Supplied | 客户端在"Type 1"消息中发送以指示该消息中包含客户端工作站的名称。服务器使用它来确定客户端是否符合本地身份验证的条件。        |
| 0x00004000 | Negotiate Local Call | 由服务器发送以指示服务器和客户端在同一台计算机上。表示客户端可以使用已建立的本地凭据进行身份验证，而不是计算对质询的响应。 |
| 0x00008000 | Negotiate Always Sign | 指示应使用"虚拟"签名对客户端和服务器之间的已验证通信进行签名。                                                                                       |
| 0x00010000 | Target Type Domain  | 服务器在Type 2消息中发送以指示目标身份验证领域是域。                                                                                                           |
| 0x00020000 | Target Type Server | 服务器在Type 2消息中发送的消息，指示目标身份验证领域是服务器。                                                                                            |
| 0x00040000 | Target Type Share | 服务器在Type 2消息中发送以指示目标身份验证领域是共享。大概是用于共享级别的身份验证。用法尚不清楚。                                      |
| 0x00080000 | Negotiate NTLM2 Key（Negotiate Extended Security） | 说明应使用NTLM2签名和Sealing方案来保护经过身份验证的通信。请注意，这是指特定的会话安全方案，与NTLMv2身份验证的使用无关。但是，此Flags可能会影响响应计算（如" NTLM2会话响应 "部分中所述）。 |
| 0x00100000 | Request Init Response（Negotiate Identify） | 该Flags的用法尚未确定。                                                                                                                                                      |
| 0x00200000 | Request Accept Response（Negotiate 0x00200000） | 该Flags的用法尚未确定。                                                                                                                                                      |
| 0x00400000 | Request Non-NT Session Key | 该Flags的用法尚未确定。                                                                                                                                                      |
| 0x00800000 | Negotiate Target Info | 服务器在Type 2消息中发送的消息，表明它在消息中包含目标信息块。目标信息块用于NTLMv2响应的计算。                                               |
| 0x01000000 | 未知           | 该Flags的用法尚未确定。                                                                                                                                                      |
| 0x02000000 | 未知           | 该Flags的用法尚未确定。                                                                                                                                                      |
| 0x04000000 | 未知           | 该Flags的用法尚未确定。                                                                                                                                                      |
| 0x08000000 | 未知           | 该Flags的用法尚未确定。                                                                                                                                                      |
| 0x10000000 | 未知           | 该Flags的用法尚未确定。                                                                                                                                                      |
| 0x20000000 | Negotiate 128        | 表示支持128位加密。                                                                                                                                                            |
| 0x40000000 | Negotiate Key Exchange | 指示客户端将在Type 3消息的"Session Key"字段中提供加密的Master Key。                                                                                            |
| 0x80000000 | Negotiate 56         | 表示支持56位加密。                                                                        

下面使用NTLM认证流程的数据包内容做示例参考学习研究：
NTLM Type1 Flag

![-w1202](https://p2.ssl.qhimg.com/t01c7d02d2dbaf15d91.jpg)

NTLM Type2 Flag

![-w1174](https://p0.ssl.qhimg.com/t01df76f3e9e760e056.jpg)


NTLM Type3 Flag

![-w1112](https://p1.ssl.qhimg.com/t01baf73be430bce4e8.jpg)


例如，考虑一条消息，该消息指定Flag：

Negotiate Unicode (0x00000001)
Request Target (0x00000004)
Negotiate NTLM (0x00000200)
Negotiate Always Sign (0x00008000)

结合以上flag位" 0x00008205 "。但是这在物理传输时上面的数据将被设置为" 0x05820000 "（因为它以小尾数字节顺序表示，这个非常重要，因为很多时候看到位置不对会产生疑问）。

# Type 1消息
我们来看看Type 1消息：

|     偏移量          | 描述                   | 内容                                               |
| ------------- | ------------------------ | -------------------------------------- |
| 0             | NTLMSSP签名            | Null-terminated ASCII "NTLMSSP" (0x4e544c4d53535000) |
| 8             | NTLM消息Type           | long (0x0000001)                       |
| 12            | Flags                    | long                                                 |
| （16）      | 提供的域（可选） | security buffer                                      |
| （24）      | 提供的工作站（可选） | security buffer                                      |
| （32）      | 操作系统版本结构（可选） | 8 Bytes                                              |
| （32） （40） | 数据块的开始（如果需要） | |

Type 1消息从客户端发送到服务器以启动NTLM身份验证。其主要目的是通过Flags指示受支持的选项，从而建立用于认证的"基本规则"。作为可选项，它还可以为服务器提供客户端的工作站名称和客户端工作站具有成员资格的域；服务器使用此信息来确定客户端是否符合本地身份验证的条件。

通常，Type 1消息包含来自以下集合的Flags：

| Flags                                      | 说明                                                                                                                                   |
| ------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Negotiate Unicode (0x00000001)              | The client sets this flag to indicate that it supports Unicode strings.                                                                  |
| Negotiate OEM (0x00000002)                  | This is set to indicate that the client supports OEM strings.                                                                            |
| Request Target (0x00000004)                 | This requests that the server send the authentication target with the Type 2 reply.                                                      |
| Negotiate NTLM (0x00000200)                 | Indicates that NTLM authentication is supported.                                                                                         |
| Negotiate Domain Supplied (0x00001000)      | When set, the client will send with the message the name of the domain in which the workstation has membership.                          |
| Negotiate Workstation Supplied (0x00002000) | Indicates that the client is sending its workstation name with the message.                                                              |
| Negotiate Always Sign (0x00008000)          | Indicates that communication between the client and server after authentication should carry a "dummy" signature.                        |
| Negotiate NTLM2 Key (0x00080000)            | Indicates that this client supports the NTLM2 signing and sealing scheme; if negotiated, this can also affect the response calculations. |
| Negotiate 128 (0x20000000)                  | Indicates that this client supports strong (128-bit) encryption.                                                                         |
| Negotiate 56 (0x80000000)                   | Indicates that this client supports medium (56-bit) encryption.                                                                          |

提供的域是一个安全缓冲区，其中包含客户机工作站具有其成员资格的域。即使客户端支持Unicode，也始终采用OEM格式。

提供的工作站是包含客户端工作站名称的安全缓冲区。这也是OEM而不是Unicode。


![-w590](https://p3.ssl.qhimg.com/t0117ea937e19c0dee9.jpg)


在Windows的最新更新中引入了OS版本结构。它标识主机的操作系统构建级别，其格式如下：

| Description | Content              |
| ----------- | -------------------- |
| 0           | Major Version Number |
| 1           | Minor Version Number |
| 2           | Build Number         |
| 4           | Unknown              |


![-w583](https://p0.ssl.qhimg.com/t01f3c900e5a6d6c8f2.jpg)


你可以通过CMD运行"winver.exe"来找到操作系统版本。它应该提供类似于以下内容的字符串：

请注意，操作系统版本结构和提供的域/工作站是可选字段。在所有消息类型中发现了三种Type 1型消息：

版本1-完全省略了提供的域和工作站安全缓冲区以及操作系统版本结构。在这种情况下，消息在Flags字段之后结束，并且是固定长度的16字节结构。这种形式通常出现在较旧的基于Win9x的系统中，并且在Open Group的ActiveX参考文档（[第11.2.2节](http://www.opengroup.org/comsource/techref2/NCH1222X.HTM#ntlm.2.2)）中有大致记录。

版本2-存在提供的域和工作站缓冲区，但没有操作系统版本结构。数据块在安全缓冲区标头之后的偏移量32处立即开始。在大多数Windows现成的出厂版本中都可以看到这种形式。

版本3-既提供了域/工作站缓冲区，也提供了OS版本结构。数据块从OS版本结构开始，偏移量为40。此格式是在相对较新的Service Pack中引入的，并且可以在Windows 2000，Windows XP和Windows 2003的当前修补版本中看到。

注：目前一般来说都是版本3了。

因此，"最最少的"格式正确的Type 1消息为：

    4e544c4d535350000100000002020000
这是"版本1"Type 1消息，仅包含NTLMSSP签名，NTLM消息Type 和最少的Flags集（NegotiateNTLM和NegotiateOEM）。

## Type 1消息示例
请考虑以下十六进制Type 1消息：

    4e544c4d53535000010000000732000006000600330000000b000b0028000000 
    050093080000000f574f524b53544154494f4e444f4d41494e
我们将其分解如下：

| 0  | 0x4e544c4d53535000       | NTLMSSP Signature                                                                                                                                                                                                             |
|----|--------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 8  | 0x01000000               | Type 1 Indicator                                                                                                                                                                                                              |
| 12 | 0x07320000               | Flags:<br>Negotiate Unicode (0x00000001)<br>Negotiate OEM (0x00000002)<br>Request Target (0x00000004)<br>Negotiate NTLM (0x00000200)<br>Negotiate Domain Supplied (0x00001000)<br>Negotiate Workstation Supplied (0x00002000) |
| 16 | 0x0600060033000000       | Supplied Domain Security Buffer:<br>Length: 6 bytes (0x0600)<br>Allocated Space: 6 bytes (0x0600)<br>Offset: 51 bytes (0x33000000)                                                                                            |
| 24 | 0x0b000b0028000000       | Supplied Workstation Security Buffer:<br>Length: 11 bytes (0x0b00)<br>Allocated Space: 11 bytes (0x0b00)<br>Offset: 40 bytes (0x28000000)                                                                                     |
| 32 | 0x050093080000000f       | OS Version Structure:<br>Major Version: 5 (0x05)<br>Minor Version: 0 (0x00)<br>Build Number: 2195 (0x9308)<br>Unknown/Reserved (0x0000000f)                                                                                   |
| 40 | 0x574f524b53544154494f4e | Supplied Workstation Data ("WORKSTATION")                                                                                                                                                                                     |
| 51 | 0x444f4d41494e           | Supplied Domain Data ("DOMAIN")                                                                                                                                                                                               |


分析这些信息，我们可以看到：

* 这是一条NTLM Type 1消息（来自NTLMSSP签名和Type 1指示符）。
* 此客户端可以支持Unicode或OEM字符串（同时设置了"Negotiate Unicode"和"Negotiate OEM"Flags）。
* 该客户端支持NTLM身份验证（Negotiate NTLM）。
* 客户端正在请求服务器发送有关身份验证目标的信息（设置了Request Target）。
* 客户端正在运行Windows 2000（5.0），内部版本2195（Windows 2000系统的生产内部版本号）。
* 该客户端正在发送其域" DOMAIN "（设置了"Negotiate Domain Supplied flag"Flags，并且该域名称存在于"提供的域安全性缓冲区"中）。
* 客户端正在发送其工作站名称，即" WORKSTATION "（已设置"Negotiate Workstation Supplied flag"Flags，并且工作站名称出现在"提供的工作站安全缓冲区"中）。

请注意，提供的工作站和域为OEM格式。此外，安全缓冲区数据块的布局顺序并不重要；在该示例中，工作站数据位于域数据之前。

创建Type 1消息后，客户端将其发送到服务器。服务器会像我们刚刚所做的那样分析消息，并创建答复。这将我们带入下一个主题，即Type 2消息。

# Type 2消息

|     偏移量         | Description                     | Content                                              |
| ------------ | ------------------------------- | ---------------------------------------------------- |
| 0            | NTLMSSP Signature               | Null-terminated ASCII "NTLMSSP" (0x4e544c4d53535000) |
| 8            | NTLM Message Type               | long (0x02000000)                                    |
| 12           | Target Name                     | security buffer                                      |
| 20           | Flags                           | long                                                 |
| 24           | Challenge                       | 8 bytes                                              |
| (32)         | Context (optional)              | 8 bytes (two consecutive longs)                      |
| (40)         | Target Information (optional)   | security buffer                                      |
| (48)         | OS Version Structure (Optional) | 8 bytes                                              |
| 32 (48) (56) | start of data block             |                                                      |
服务器将Type 2消息发送给客户端，以响应客户端的Type 1消息。它用于完成与客户端的选项Negotiate，也给客户端带来了challenge。它可以选择包含有关身份验证目标的信息。

典型的2类消息flags包括（前面已经翻译过，这里就不翻译了，正好也可以看看原文）：

| Flags    | 说明       |
| ---------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Negotiate Unicode (0x00000001)     | The server sets this flag to indicate that it will be using Unicode strings. This should only be set if the client indicates (in the Type 1 message) that it supports Unicode. Either this flag or Negotiate OEM should be set, but not both.        |
| Negotiate OEM (0x00000002)         | This flag is set to indicate that the server will be using OEM strings. This should only be set if the client indicates (in the Type 1 message) that it will support OEM strings. Either this flag or Negotiate Unicode should be set, but not both. |
| Request Target (0x00000004)        | This flag is often set in the Type 2 message; while it has a well-defined meaning within the Type 1 message, its semantics here are unclear.                                                                                                         |
| Negotiate NTLM (0x00000200)        | Indicates that NTLM authentication is supported.                                                                                                                                                                                                     |
| Negotiate Local Call (0x00004000)  | The server sets this flag to inform the client that the server and client are on the same machine. The server provides a local security context handle with the message.                                                                             |
| Negotiate Always Sign (0x00008000) | Indicates that communication between the client and server after authentication should carry a "dummy" signature.                                                                                                                                    |
| Target Type Domain (0x00010000)    | The server sets this flag to indicate that the authentication target is being sent with the message and represents a domain.                                                                                                                         |
| Target Type Server (0x00020000)    | The server sets this flag to indicate that the authentication target is being sent with the message and represents a server.                                                                                                                         |
| Target Type Share (0x00040000)     | The server apparently sets this flag to indicate that the authentication target is being sent with the message and represents a network share. This has not been confirmed.                                                                          |
| Negotiate NTLM2 Key (0x00080000)   | Indicates that this server supports the NTLM2 signing and sealing scheme; if negotiated, this can also affect the client's response calculations.                                                                                                    |
| Negotiate Target Info (0x00800000) | The server sets this flag to indicate that a Target Information block is being sent with the message.                                                                                                                                                |
| Negotiate 128 (0x20000000)         | Indicates that this server supports strong (128-bit) encryption.                                                                                                                                                                                     |
| Negotiate 56 (0x80000000)          | Indicates that this server supports medium (56-bit) encryption.                                                                                                                                                                                      |

Target Name是包含身份验证目标信息的安全缓冲区。 这通常是响应客户端请求目标而发送的（通过设置Type 1消息中的Request Target Flags）。 它可以包含域，服务器或（显然）网络共享。 通过Target Type Domain, Target Type Server, and Target Type Share flags指示目标类型。 Target Name可以是Unicode或OEM，如Type 2消息中存在适当的Flags所指示。

challenge是一个8字节的随机数据块。客户端将使用它来制定响应。

设置"Negotiate Local Call"时，通常会填充上下文字段。它包含一个SSPI上下文句柄，该句柄允许客户端"short-circuit"身份验证并有效规避对challenge的响应。从物理上讲，上下文是两个长值。稍后将在"Local Authentication "部分中对此进行详细介绍。

Target information是一个包含目标信息块的安全缓冲区，该缓冲区用于计算 NTLMv2响应（稍后讨论）。它由一系列子块组成，每个子块包括：

| Field   | Content        | Description                                                                                                                                                                                                    |
| ------- | -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Type    | short          | Indicates the type of data in this subblock:<br>1 (0x0100):	Server name<br>2 (0x0200):	Domain name<br>3 (0x0300):	Fully-qualified DNS host name (i.e., server.domain.com)<br>4 (0x0400):	DNS domain name (i.e., domain.com) |
| Length  | short          | Length in bytes of this subblock's content field                                                                                                                                                               |
| Content | Unicode string | Content as indicated by the type field. Always sent in Unicode, even when OEM is indicated by the message flags.                                                                                               |


前面已经描述了OS版本的结构。

与Type 1消息一样，已经观察到一些Type 2的版本：

版本1-上下文，目标信息和操作系统版本结构均被省略。数据块（仅包含Target Name安全缓冲区的内容）从偏移量32开始。这种形式在较旧的基于Win9x的系统中可见，并且在Open Group的ActiveX参考文档（第11.2.3节）中得到了大致记录。

版本2-存在Context 和 Target Information fields字段，但没有OS版本结构。数据块在目标信息标题之后的偏移量48处开始。在大多数现成的Windows发行版中都可以看到这种形式。

版本3-上下文，目标信息和操作系统版本结构均存在。数据块在OS版本结构之后开始，偏移量为56。同样，缓冲区可能为空（产生零长度的数据块）。这种形式是在相对较新的Service Pack中引入的，并且可以在Windows 2000，Windows XP和Windows 2003的当前修补版本中看到。

最小的Type 2消息如下所示：

    4e544c4d53535000020000000000000000000000020200000123456789abcdef
该消息包含NTLMSSP签名，NTLM消息Type ，空Target Name，最少Flags（NegotiateNTLM和NegotiateOEM）以及质询。

## Type 2消息示例
让我们看下面的十六进制Type 2消息：

    4e544c4d53535000020000000c000c003000000001028100 
    0123456789abcdef0000000000000000620062003c000000 
    44004f004d00410049004e0002000c0044004f004d004100 
    49004e0001000c0053004500520056004500520004001400 
    64006f006d00610069006e002e0063006f006d0003002200 
    7300650072007600650072002e0064006f006d0061006900 
    6e002e0063006f006d0000000000
将其分为几个组成部分可以得出：

| 偏移量  | 值                                                                                                                                                                                                                                                                         | 说明                                                                                                                                |
|----|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------|
| 0  | 0x4e544c4d53535000                                                                                                                                                                                                                                                                         | NTLMSSP Signature                                                                                                                                |
| 8  | 0x02000000                                                                                                                                                                                                                                                                                 | Type 2 Indicator                                                                                                                                 |
| 12 | 0x0c000c0030000000                                                                                                                                                                                                                                                                         | Target Name Security Buffer:<br>Length: 12 bytes (0x0c00)<br>Allocated Space: 12 bytes (0x0c00)<br>Offset: 48 bytes (0x30000000)                 |
| 20 | 0x01028100                                                                                                                                                                                                                                                                                 | Flags:<br>Negotiate Unicode (0x00000001)<br>Negotiate NTLM (0x00000200)<br>Target Type Domain (0x00010000)<br>Negotiate Target Info (0x00800000) |
| 24 | 0x0123456789abcdef                                                                                                                                                                                                                                                                         | Challenge                                                                                                                                        |
| 32 | 0x0000000000000000                                                                                                                                                                                                                                                                         | Context                                                                                                                                          |
| 40 | 0x620062003c000000                                                                                                                                                                                                                                                                         | Target Information Security Buffer:<br>Length: 98 bytes (0x6200)<br>Allocated Space: 98 bytes (0x6200)<br>Offset: 60 bytes (0x3c000000)          |
| 48 | 0x44004f004d004100   <br>49004e00                                                                                                                                                                                                                                                          | Target Name Data ("DOMAIN")                                                                                                                      |
| 60 | 0x02000c0044004f00   <br>4d00410049004e00   <br>01000c0053004500   <br>5200560045005200   <br>0400140064006f00   <br>6d00610069006e00   <br>2e0063006f006d00   <br>0300220073006500   <br>7200760065007200   <br>2e0064006f006d00   <br>610069006e002e00   <br>63006f006d000000   <br>0000 | Target Information Data:      接下一表格                                                                                                                   |

Target Information Data：

| 偏移量  | 值                                                                                                                                                                                                                                                                         | 
|--------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------|
| 0x02000c0044004f00<br>  4d00410049004e00                                                               | Domain name subblock:<br>Type: 2 (Domain name,<br> <br>0x0200<br>)<br>Length: 12 bytes (<br>0x0c00<br>)<br>Data: "<br>DOMAIN"                    |
| 0x01000c0053004500<br>  5200560045005200                                                               | Server name subblock:<br>Type: 1 (Server name,<br> <br>0x0100<br>)<br>Length: 12 bytes (<br>0x0c00<br>)<br>Data: "<br>SERVER"                    |
| 0x0400140064006f00<br>  6d00610069006e00<br>  2e0063006f006d00                                         | DNS domain name subblock:<br>Type: 4 (DNS domain name,<br> <br>0x0400<br>)<br>Length: 20 bytes (<br>0x1400<br>)<br>Data: "<br>domain.com"        |
| 0x0300220073006500<br>  7200760065007200<br>  2e0064006f006d00<br>  610069006e002e00<br>  63006f006d00 | DNS server name subblock:<br>Type: 3 (DNS server name,<br> <br>0x0300<br>)<br>Length: 34 bytes (<br>0x2200<br>)<br>Data: "<br>server.domain.com" |
| 0x00000000                                                                                             | Terminator subblock:<br>Type: 0 (terminator,<br> <br>0x0000<br>)<br>Length: 0 bytes (<br>0x0000)                                                 |

对此消息的分析显示：

* 这是一条NTLM 2类消息（来自NTLMSSP Signature and Type 2指示符）。
* 服务器已指示将使用Unicode编码字符串（已设置Negotiate UnicodeFlags）。
* 服务器支持NTLM身份验证（Negotiate NTLM）。
* 服务器提供的Target Name将被填充并代表一个域（"Target Type Domain"Flags已设置并且该域名存在于"Target Name Security Buffer"中）。
* 服务器正在提供目标信息结构（已设置Negotiate目标信息）。此结构存在于目标信息（Target Info）安全缓冲区（域名" DOMAIN "，服务器名称" SERVER "，DNS域名" domain.com "和DNS服务器名称" server.domain.com "）中。
* 服务器生成的challenge是" 0x0123456789abcdef "。
* 空上下文。

请注意，Target Name采用Unicode格式（由"NegotiateUnicode"Flags指定）。

服务器创建Type 2消息后，它将发送给客户端。客户端的Type 3消息中提供了对服务器质询的响应。

# Type 3消息

|              | Description                     | Content                                              |
| ------------ | ------------------------------- | ---------------------------------------------------- |
| 0            | NTLMSSP Signature               | Null-terminated ASCII "NTLMSSP" (0x4e544c4d53535000) |
| 8            | NTLM Message Type               | long (0x03000000)                                    |
| 12           | LM/LMv2 Response                | security buffer                                      |
| 20           | NTLM/NTLMv2 Response            | security buffer                                      |
| 28           | Target Name                     | security buffer                                      |
| 36           | User Name                       | security buffer                                      |
| 44           | Workstation Name                | security buffer                                      |
| (52)         | Session Key (optional)          | security buffer                                      |
| (60)         | Flags (optional)                | long                                                 |
| (64)         | OS Version Structure (Optional) | 8 bytes                                              |
| 52 (64) (72) | start of data block             |                                                      |

Type 3消息是身份验证的最后一步。此消息包含客户对Type 2质询的响应，这证明客户端无需直接发送密码进行认证而使用NTLM HASH认证。Type 3消息还指示身份验证目标（域或服务器名称）和身份验证帐户的用户名，以及客户端工作站名称。

请注意，Type 3消息中的Flags是可选的；较旧的客户端在消息中既不包含Session Key也不包含Flags。通过实验确定，Type 3 Flags（如果包含）在面向连接的身份验证中不带有任何其他语义。它们似乎对身份验证或会话安全性都没有任何明显的影响。发送Flags的客户端通常会非常紧密地镜像已建立的Type 2设置。可能会将Flags作为已建立选项的"提醒"发送，以允许服务器避免缓存Negotiate的设置。但是，Type 3 Flags在数据报样式身份验证期间是有意义的的 。

LM/LMv2和NTLM/NTLMv2响应是安全缓冲区，其中包含根据用户的密码响应Type 2质询而创建的答复。下一节概述了生成这些响应的过程。

target name是一个安全缓冲区，其中包含身份验证领域，其中身份验证帐户具有成员身份（域帐户的域名，或本地计算机帐户的服务器名）。根据Negotiate的编码，它可以是Unicode或OEM。

user name是包含身份验证帐户名的安全缓冲区。根据Negotiate的编码，它可以是Unicode或OEM。

workstation name是包含客户端工作站名称的安全缓冲区。根据Negotiate的编码，它可以是Unicode或OEM。

session key值在Key交换期间由会话安全性机制使用；"会话安全性"部分将对此进行详细讨论 。

当在Type 2消息中建立了"Negotiate Local Call"时，Type 3消息中的安全缓冲区通常都为空（零长度）。客户端"采用"在Type 2消息中发送的SSPI上下文，从而有效地避免了计算适当响应的需求。

OS版本结构与前面描述的格式相同。

同样，Type 3消息有一些变体：

版本1-Session Key，Flags和OS版本结构被省略。在这种情况下，数据块在"工作站名称"安全缓冲区标头之后的偏移量52处开始。此格式在基于Win9x的较旧系统中可见。

版本2-包含Session Key和Flags，但不包含OS版本结构。在这种情况下，数据块在Flags字段之后的偏移量64处开始。这种形式在大多数现成的Windows版本中都可以看到，并且在Open Group的ActiveX参考文档（第11.2.4节）中有大致记录。。

版本3-Session Key，Flags和OS版本结构均存在。数据块从OS版本结构开始，偏移量为72。此格式是在相对较新的Service Pack中引入的，并且可以在Windows 2000，Windows XP和Windows 2003的当前修补版本中看到。

## 名称可变性（Name Variations）
除了消息的布局变化之外，user和Target Name还可以在Type 3消息中以几种不同的格式显示。通常情况下，"User Name"字段将使用Windows帐户名称填充，而"Target Name"将使用NT域名填充。但是，用户名和/或域也可以采用Kerberos样式的" user@domain.com"格式进行多种组合。已经支持多种变体，并具有一些可能的含义如下表：

| Format          | Type 3 Field Content                                                                                                                                                                                                                                                            | Notes                                                                                                                                                                                                                                                                                                                                                                             |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| DOMAIN\user     | User Name = "user"<br>Target Name = "DOMAIN"                                                                                                                                                                                                                                        | Target Name=“ DOMAIN”这是“常规”格式； 用户名字段包含Windows用户名，目标名包含NT样式的NetBIOS域或服务器名。                                                                                                                                                                                                                        |
| domain.com\user | User Name = "user"<br>Target Name = "domain.com"                                                                                                                                                                                                                                    | Target Name=“ domain.com”在此，Type 3消息中的“Target Name”字段填充有DNS域名/领域名称（对于本地计算机帐户，则为标准DNS主机名）。                                                                                                                                                                                                          |
| user@DOMAIN     | User Name = "user@DOMAIN"<br>Target Name is empty                                                                                                                                                                                                                                   | Target Name为空在这种情况下，“Target Name”字段为空（零长度），而“用户名”字段使用Kerberos样式的“ user @ realm”格式。 但是，使用NetBIOS域名代替DNS域。 已经观察到，本地计算机帐户不支持此格式。 此外，NTLMv2 / LMv2身份验证似乎不支持此格式。 |
| user@domain.com | User Name = "user@domain.com"<br>Target Name is empty | 类型3消息中的Target Name字段为空； “用户名”字段包含Kerberos样式的“ user @ realm”格式以及DNS域。本地计算机帐户似乎不支持此方式。                                                                                                                                                                                                                                                                                                                                                                 |

## 应答challenge（Responding to the Challenge）
客户端创建一个或多个对Type 2质询的响应，并将响应以Type 3消息发送给服务器。有六种Type 的响应：

* LM（LAN管理器）响应-由大多数较旧的客户端发送，这是"原始"响应Type 。
* NTLM响应-这是由基于NT的客户端（包括Windows 2000和XP）发送的。
* NTLMv2响应-一种较新的返回类型，Windows NT Service Pack 4中引入更新的响应Type。它替换了启用了NTLMv2的系统上的NTLM响应。
* LMv2响应-替换NTLMv2系统上的LM响应。
* NTLM2会话响应-在未经NTLMv2身份验证的情况下Negotiate NTLM2会话安全性时使用，此方案会更改LM和NTLM响应的语义。
* 匿名响应(Anonymous Response)-建立匿名上下文时使用；不会显示实际凭据，也不会进行真正的身份验证。"Stub"字段显示在Type 3消息中。

有关这些方案的更多信息，强烈建议您阅读Christopher Hertel的[《实现CIFS》](http://ubiqx.org/cifs)，尤其是有关[身份验证的部分](http://ubiqx.org/cifs/SMB.html#SMB.8)。

响应(responses)用作客户端拥有密码口令的间接证明。客户端使用密码导出LM和/或NTLM哈希（在下一节中讨论）；这些值依次用于计算对challenge的适当响应。域控制器（或本地计算机帐户的服务器）存储LM和NTLM哈希作为密码；当从客户端收到响应时，这些存储的值将用于计算适当的响应值，并将其与客户端发送的响应值进行比较。匹配会成功验证用户。

请注意，与Unix密码哈希不同，LM和NTLM哈希在响应计算的上下文中是与密码等效的。它们必须受到保护，因为即使不知道实际密码本身，也可以使用它们来通过网络对用户进行身份验证。（注：也就是说知道密码和指导哈希是同样的都可以用于认证）

### LM响应（The LM Response）
LM响应是由大多数客户端发送的。此方案比NTLM响应要旧，并且安全性较低。虽然较新的客户端支持NTLM响应，但它们通常会同时发送这两个响应以与旧服务器兼容。因此，在支持NTLM响应的许多客户端中，仍然存在LM响应中存在的安全漏洞。

LM响应的计算方式如下（有关 Java中的示例实现，请参阅附录D）：

1. 首先将用户密码（作为OEM字符串）转换为大写。
2. 将其填充为14个字节，不足则用0填充。
3. 将此固定长度的密码分为两个7字节。
4. 这些值做为两个DESKey（每个7字节为一个）。
5. 这些Key中的每一个都用于对特定的ASCII字符串"KGS!@#$%" 进行DES加密（产生两个8字节密文值）。
6. 将这两个密文值连接起来以形成一个16字节的值-LM哈希。
7. 16字节的LM哈希被空填充为21个字节。
8. 该值分为三个7字节。
9. 这些值用于创建三个DESKey。
10. 这些Key中的每一个都用于对来自Type 2消息的质询进行DES加密（产生三个8字节密文值）。
11. 将这三个密文值连接起来形成一个24字节的值。这就是LM的回应。
如果用户的密码长度超过15个字符，则主机或域控制器将不会为该用户存储LM哈希。在这种情况下，LM响应不能用于认证用户。仍会生成一个响应并将其放置在LM响应字段中，并使用16字节的空值（0x00000000000000000000000000000000000000）作为计算中的LM哈希。该值将被目标忽略。（注：平时在利用工具时LM字段可以放置空值即可）

最好用一个详细的例子说明响应计算过程。考虑一个密码为" SecREt01 " 的用户，它响应Type 2质询" 0x0123456789abcdef "。

1. 密码（作为OEM字符串）将转换为大写，并以十六进制的形式给出" SECRET01 "（或" 0x5345435245543031 "）。
2. 将其填充为14个字节，为" 0x534543524554303031000000000000 "。
3. 此值分为两个7字节，即" 0x53454352455430 "和" 0x31000000000000 "。
4. 这两个值用于创建两个DESKey。DESKey的长度为8个字节；每个字节包含7位Key材料和1个奇偶校验位（根据基础DES的实现，可以检查或不检查奇偶校验位）。我们的第一个7字节值" 0x53454352455430 "将以二进制形式表示为：
    01010011 01000101 01000011 01010010 01000101 01010100 00110000
    
    此值的未经奇偶校验调整的DESKey为：
    
    0101001 0 1010001 0 0101000 0 0110101 0 0010010 0 0010101 0 0101000 0 0110000 0
    
    （奇偶校验位在上方以最后一位显示）。十六进制为" 0x52a2506a242a5060 "。应用奇数奇偶校验以确保每个八位位组中的总置位位数为奇数可得出：
    
    0101001 0 1010001 0 0101000 1 0110101 1 0010010 1 0010101 0 0101000 1 0110000 1
    
    这是第一个DESKey（十六进制为" 0x52a2516b252a5161 "）。然后，我们对第二个7字节值" 0x31000000000000 " 应用相同的过程，以二进制表示：
    
    00110001 00000000 00000000 00000000 00000000 00000000 00000000
    
    创建一个非奇偶校验的DESKey可以得到：
    
    0011000 0 1000000 0 0000000 0 0000000 0 0000000 0 0000000 0 0000000 0 0000000 0
    
    （十六进制为" 0x3080000000000000 "）。调整奇偶校验位可得出：
    
    0011000 1 1000000 0 0000000 1 0000000 1 0000000 1 0000000 1 0000000 1 0000000 1
    
    这是我们的第二个DESKey，十六进制为" 0x3180010101010101 "。请注意，如果我们的特定DES实现不强制执行奇偶校验（很多都不强制），则可以跳过奇偶校验调整步骤；然后，将非奇偶校验调整后的值用作DESKey。在任何情况下，奇偶校验位都不会影响加密过程。

5. 我们的每个Key都用于DES加密常量ASCII字符串"KGS!@#$% "（十六进制为" 0x4b47532140232425 "）。这使我们获得" 0xff3750bcc2b22412 "（使用第一个Key）和" 0xc2265b23734e0dac "（使用第二个Key）。
6. 这些密文值被连接起来以形成我们的16字节LM哈希-" 0xff3750bcc2b22412c2265b23734e0dac "。
7. 将其空填充到21个字节，得到" 0xff3750bcc2b22412c2265b23734e0dac0000000000 "。
8. 该值分为三个7字节的三分之三：" 0xff3750bcc2b224 "，" 0x12c2265b23734e "和" 0x0dac0000000000 "。
9. 这三个值用于创建三个DESKey。使用前面概述的过程，我们的第一个价值是：
    11111111 00110111 01010000 10111100 11000010 10110010 00100100
    
    给我们提供经过奇偶校验调整的DESKey：
    
    1111111 0 1001101 1 1101010 1 0001011 0 1100110 1 0001010 1 1100100 0 0100100 1
    
    （十六进制为" 0xfe9bd516cd15c849 "）。第二个值：
    
    00010010 11000010 00100110 01011011 00100011 01110011 01001110
    
    结果的关键：
    
    0001001 1 0110000 1 1000100 1 1100101 1 1011001 1 0001101 0 1100110 1 1001110 1
    
    （" 0x136189cbb31acd9d "）。最后，第三个值：
    
    00001101 10101100 00000000 00000000 00000000 00000000 00000000
    
    给我们：
    
    0000110 1 1101011 0 0000000 1 0000000 1 0000000 1 0000000 1 0000000 1 0000000 1
    
    这是第三个DESKey（" 0x0dd6010101010101 "）。

10. 三个Key中的每个Key都用于对来自Type 2消息的质询进行DES加密（在我们的示例中为" 0x0123456789abcdef "）。这给出结果" 0xc337cd5cbd44fc97 "（使用第一个键），" 0x82a667af6d427c6d "（使用第二个键）和" 0xe67c20c2d3e77c56 "（使用第三个键）。
11. 将这三个密文值连接起来以形成24字节LM响应：
0xc337cd5cbd44fc9782a667af6d427c6de67c20c2d3e77c56

该算法有几个弱点，使其容易受到攻击。尽管在Hertel文本中详细介绍了这些内容，但最突出的问题是：在计算响应之前，密码将转换为大写。

### NTLM响应(The NTLM Response)
NTLM响应由较新的客户端发送。该方案解决了LM响应中的一些缺陷。但是，它仍然被认为是相当薄弱的。此外，NTLM响应几乎总是与LM响应一起发送。可以利用该算法的弱点来获得不区分大小写的密码，并通过反复试验来找到NTLM响应所采用的区分大小写的密码。

NTLM响应的计算方式如下（有关示例Java实现，请参阅附录D）：

1. MD4消息摘要算法（在RFC 1320中描述 ）应用于Unicode大小写混合密码。结果为16字节的值-NTLM哈希。
2. 16字节的NTLM哈希值被空填充为21个字节。
3. 该值分为三个7字节的三分之二。
4. 这些值用于创建三个DESKey（每个7字节的三分之一）。
5. 这些Key中的每一个都用于对来自Type 2消息的质询进行DES加密（产生三个8字节密文值）。
6. 将这三个密文值连接起来形成一个24字节的值。这是NTLM响应。

注意，只有哈希值的计算与LM方案有所不同；响应计算是相同的。为了说明此过程，我们将其应用到之前的示例（密码为"SecREt01" 的用户，响应Type 2质询" 0x0123456789abcdef "）。

1. Unicode混合大小写密码为" 0x53006500630052004500740030003100 "（十六进制）；计算出该值的MD4哈希，结果为" 0xcd06ca7c7e10c99b1d33b7485a2ed808 "。这是NTLM哈希。
2. 将其空填充到21个字节，得到" 0xcd06ca7c7e10c99b1d33b7485a2ed8080000000000 "。
3. 该值分为三个7字节的三分之三：" 0xcd06ca7c7e10c9 "，" 0x9b1d33b7485a2e "和" 0xd8080000000000 "。
4. 这三个值用于创建三个DESKey。我们的第一个值：
    11001101 00000110 11001010 01111100 01111110 00010000 11001001
    
    得出奇偶校验调整后的Key：
    
    1100110 1 1000001 1 1011001 1 0100111 1 1100011 1 1111000 1 0100001 1 1001001 0
    
    （十六进制为" 0xcd83b34fc7f14392 "）。第二个值：
    
    10011011 00011101 00110011 10110111 01001000 01011010 00101110
    
    给出Key：
    
    1001101 1 1000111 1 0100110 0 0111011 0 0111010 1 0100001 1 0110100 0 0101110 1
    
    （" 0x9b8f4c767543685d "）。我们的第三个价值：
    
    11011000 00001000 00000000 00000000 00000000 00000000 00000000
    
    产生我们的第三个关键：
    
    1101100 1 0000010 0 0000000 1 0000000 1 0000000 1 0000000 1 0000000 1 0000000 1
    
    （十六进制为" 0xd904010101010101 "）。

1. 这三个Key中的每一个都用于对来自Type 2消息（" 0x0123456789abcdef "）的质询进行DES加密。这将产生结果" 0x25a98c1c31e81847 "（使用第一个Key），" 0x466b29b2df4680f3 "（使用第二个Key）和" 0x9958fb8c213a9cc6 "（使用第三个Key）。
2. 这三个密文值连接起来形成24字节NTLM响应：
0x25a98c1c31e81847466b29b2df4680f39958fb8c213a9cc6

### NTLMv2响应
NTLM版本2（"NTLMv2"）专门用于解决NTLM中存在的安全问题。启用NTLMv2时，NTLM响应将替换为NTLMv2响应，而LM响应将替换为LMv2响应（我们将在下面讨论）。

NTLMv2响应的计算方式如下（有关 Java中的示例实现，请参阅附录D）：

1. 获取NTLM密码哈希（如前所述，这是Unicode混合大小写密码的MD4摘要）。
2. Unicode大写用户名与Unicode身份验证目标（在Type 3消息的"Target Name"字段中指定的域或服务器名称）串联在一起。请注意，即使已协商使用OEM编码，此计算也始终使用Unicode表示形式。还请注意，用户名将转换为大写，而身份验证目标（authentication target）区分大小写，并且必须与"Target Name"字段中显示的大小写匹配。使用16字节NTLM哈希作为Key，将HMAC-MD5消息认证代码算法（[RFC 2104](http://www.ietf.org/rfc/rfc2104.txt)）应用于该值。结果为16字节的值-NTLMv2哈希。
3. 构造一个称为"Blob"的数据块。Hertel text更详细地讨论了这种结构的格式。简要说明如下：

    |   offset         | Description        | Content                                                                                                      |
    | ---------- | ------------------ | ------------------------------------------------------------------------------------------------------------ |
    | 0          | Blob Signature     | 0x01010000                                                                                                   |
    | 4          | Reserved           | long (0x00000000)                                                                                            |
    | 8          | Timestamp          | Little-endian, 64-bit signed value representing the number of tenths of a microsecond since January 1, 1601. |
    | 16         | Client Nonce       | 8 bytes                                                                                                      |
    | 24         | Unknown            | 4 bytes                                                                                                      |
    | 28         | Target Information | Target Information block (from the Type 2 message).                                                          |
    | (variable) | Unknown            | 4 bytes                                                                                                      |
4. 来自Type 2消息的质询与Blob连接在一起。使用16字节NTLMv2哈希（在步骤2中计算）作为Key，将HMAC-MD5消息认证代码算法应用于此值。结果是一个16字节的输出值。
5. 该值与Blob连接起来形成NTLMv2响应。

让我们来看一个例子。由于我们需要更多信息来计算NTLMv2响应，因此我们将使用前面提供的示例中的以下值：

|名称             |      值                                                                                                                                                     
------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Target:             | DOMAIN                                                                                                                                                                                                 |
| Username:           | user                                                                                                                                                                                                   |
| Password:           | SecREt01                                                                                                                                                                                               |
| Challenge:          | 0x0123456789abcdef                                                                                                                                                                                     |
| Target Information: | 0x02000c0044004f004d00410049004e0001000c005300450052005600450052000400140064006f006d00610069006e002e0063006f006d00030022007300650072007600650072002e0064006f006d00610069006e002e0063006f006d0000000000 |

1. Unicode混合大小写密码为" 0x53006500630052004500740030003100 "（十六进制）；计算出该值的MD4哈希，结果为" 0xcd06ca7c7e10c99b1d33b7485a2ed808 "。这是NTLM哈希。
2. Unicode大写的用户名与Unicode身份验证目标连接在一起，并提供" USERDOMAIN "（或十六进制的" 0x55005300450052004200444f004d00410049004e00 "）。使用上一步中的16字节NTLM哈希作为Key，将HMAC-MD5应用于此值，这将产生" 0x04b8e0ba74289cc540826bab1dee63ae "。这是NTLMv2哈希。
3. 接下来，构建Blob。时间戳是其中最繁琐的部分，添加11644473600将使我们在1601年1月1日之后获得秒数（12700317600）。乘以10的7次方（10000000）将得到十分之一微秒（127003176000000000）。作为小端64位值，它是" 0x0090d336b734c301 "（十六进制）。
    我们还需要生成一个8字节的随机"客户随机数"；我们将使用不太随机的" 0xffffff0011223344 "。构造其余的Blob很容易；我们只是串联：
    
    0x01010000	（blob签名）
    
    0x00000000	（保留值）
    
    0x0090d336b734c301	（我们的时间戳）
    
    0xffffff0011223344	（随机的客户随机数）
    
    0x00000000	（未知，但零将起作用）
    
    0x02000c0044004f00 （我们的目标信息块）
      4d00410049004e00 
      01000c0053004500 
      5200560045005200 
      0400140064006f00 
      6d00610069006e00 
      2e0063006f006d00 
      0300220073006500 
      7200760065007200 
      2e0064006f006d00 
      610069006e002e00 
      63006f006d000000 
      0000
    
    0x00000000	（未知，但零会起作用）

4. 我们将Type 2challenge与Blob连接起来：
   
    0x0123456789abcdef0101000000000000 
      0090d336b734c301ffffff0011223344 
      0000000002000c0044004f004d004100 
      49004e0001000c005300450052005600 
      450052000400140064006f006d006100 
      69006e002e0063006f006d0003002200 
      7300650072007600650072002e006400 
      6f006d00610069006e002e0063006f00 
      6d000000000000000000
    使用第2步中的NTLMv2哈希作为Key，将HMAC-MD5应用于该值，即可得到16个字节的值" 0xcbabbca713eb795d04c97abc01ee4983 "。

5. 此值与Blob串联以获得NTLMv2响应：
    
    0xcbabbca713eb795d04c97abc01ee4983 
      01010000000000000090d336b734c301 
      ffffff00112233440000000002000c00 
      44004f004d00410049004e0001000c00 
      53004500520056004500520004001400 
      64006f006d00610069006e002e006300 
      6f006d00030022007300650072007600 
      650072002e0064006f006d0061006900 
      6e002e0063006f006d00000000000000 
      0000

### LMv2响应(The LMv2 Response)
LMv2响应用于提供与旧服务器的直通身份验证兼容性。与客户端通信的服务器很可能不会实际执行身份验证；而是将响应传递到域控制器进行验证。较旧的服务器仅传递LM响应，并且期望它恰好是24个字节。LMv2响应旨在使此类服务器正常运行。它实际上是一个"微型" NTLMv2响应，如下所示（有关示例Java实现，请参阅附录D）：

1. 将计算NTLM密码哈希（Unicode大小写混合的密码的MD4摘要）。
2. Unicode大写用户名与Type 3消息的"Target Name"字段中显示的Unicode身份验证目标（域或服务器名称）串联。使用16字节NTLM哈希作为Key，将HMAC-MD5消息认证代码算法应用于该值。结果为16字节的值-NTLMv2哈希。
3. 创建一个随机的8字节client nonce（这与NTLMv2 blob中使用的client nonce相同）。
4. 来自Type 2消息的质询与client nonce串联在一起。使用16字节NTLMv2哈希（在步骤2中计算）作为Key，将HMAC-MD5消息认证代码算法应用于此值。结果是一个16字节的输出值。
5. 该值与8字节client nonce串联在一起，以形成24字节LMv2响应。

我们将使用一个久经考验的样本值通过一个简短的示例来说明此过程：
目标：	域
用户名：	用户
密码：	SecREt01
challenge：	0x0123456789abcdef

1. Unicode混合大小写密码为" 0x53006500630052004500740030003100 "（十六进制）；计算出该值的MD4哈希，结果为" 0xcd06ca7c7e10c99b1d33b7485a2ed808 "。这是NTLM哈希。
2. Unicode大写的用户名与Unicode身份验证目标连接在一起，并提供" USERDOMAIN "（或十六进制的" 0x55005300450052004200444f004d00410049004e00 "）。使用上一步中的16字节NTLM哈希作为Key，将HMAC-MD5应用于此值，这将产生" 0x04b8e0ba74289cc540826bab1dee63ae "。这是NTLMv2哈希。
3. 创建一个随机的8字节client nonce。在我们的NTLMv2示例中，我们将使用" 0xffffff0011223344 "。
4. 然后，我们将Type 2challenge与客户现时串联起来：
    0x0123456789abcdefffffff0011223344
    
    使用第2步中的NTLMv2哈希作为Key，将HMAC-MD5应用于该值，即可得到16字节的值" 0xd6e6152ea25d03b7c6ba6629c2d6aaf0 "。

5. 此值与client nonce连接在一起，以获得24字节LMv2响应：
0xd6e6152ea25d03b7c6ba6629c2d6aaf0ffffff0011223344

### NTLM2会话响应(The NTLM2 Session Response)
NTLM2会话响应可以与NTLM2会话安全性（session security）结合使用（可通过"Negotiate NTLM2 Key"Flags使用）。这用于在不支持完整NTLMv2身份验证的环境中提供增强的保护，以抵御预计算的字典攻击（尤其是基于Rainbow Table的攻击）。

NTLM2会话响应将替换LM和NTLM响应字段，如下所示（有关 Java中的示例实现，请参阅附录D）：

1. 创建一个随机的8字节client nonce。
2. client nonce被空填充为24个字节。此值放在Type 3消息的LM响应字段中。
3. 来自Type 2消息的质询（challenge）与8字节的client nonce串联在一起以形成session nonce。
4. 将MD5消息摘要算法（[RFC 1321](http://www.ietf.org/rfc/rfc1321.txt) ）应用于session nonce，产生16字节的值。
5. 该值将被截断为8个字节，以形成NTLM2会话哈希。
6. 获得NTLM密码哈希（如所讨论的，这是Unicode混合大小写密码的MD4摘要）。
7. 16字节的NTLM哈希值被空填充为21个字节。
8. 该值分为三个7字节。
9. 这些值用于创建三个DESKey（每个7字节）。
10. 这些Key中的每一个都用于对NTLM2会话散列进行DES加密（产生三个8字节密文值）。
11. 将这三个密文值连接起来形成一个24字节的值。这是NTLM2会话响应，放置在Type 3消息的NTLM响应字段中。

 
为了用我们先前的示例值（用户密码为" SecREt01 "，响应Type 2质询" 0x0123456789abcdef "）进行演示：

1. 创建一个随机的8字节client nonce；与前面的示例一样，我们将使用" 0xffffff0011223344 "。
2. challenge是将空值填充为24个字节：
    
    0xffffff001122334400000000000000000000000000000000000000
    
    此值放在Type 3消息的LM响应字段中。

5. 来自Type 2消息的质询与client nonce串联在一起，形成session nonce（" 0x0123456789abcdefffffff0011223344 "）。
6. 将MD5摘要应用于该随机数将产生16字节的值" 0xbeac9a1bc5a9867c15192b3105d5beb1 "。
7. 它被截断为8个字节，以获得NTLM2会话哈希（" 0xbeac9a1bc5a9867c "）。
8. Unicode大小写混合密码为" 0x53006500630052004500740030003100 "；将MD4摘要应用于此值将为我们提供NTLM哈希（" 0xcd06ca7c7e10c99b1d33b7485a2ed808 "）。
9. 将其空填充到21个字节，得到" 0xcd06ca7c7e10c99b1d33b7485a2ed8080000000000 "。
10. 该值分为三个7字节的三分之三：" 0xcd06ca7c7e10c9 "，" 0x9b1d33b7485a2e "和" 0xd8080000000000 "。
11. 这些值用于创建三个DESKey（如在我们之前的NTLM响应示例中计算的" 0xcd83b34fc7f14392 "，" 0x9b8f4c767543685d "和" 0xd904010101010101 "）。
12. 这三个Key中的每一个都用于对NTLM2会话哈希（" 0xbeac9a1bc5a9867c "）进行DES加密。这将产生结果" 0x10d550832d12b2cc "（使用第一个Key），" 0xb79d5ad1f4eed3df "（使用第二个Key）和" 0x82aca4c3681dd455 "（使用第三个Key）。
13. 这三个密文值被连接起来以形成24字节的NTLM2会话响应：
    
    0x10d550832d12b2ccb79d5ad1f4eed3df82aca4c3681dd455

    放置在Type 3消息的NTLM响应字段中。

### 匿名响应(The Anonymous Response)
当客户端建立匿名上下文而非真正的基于用户的上下文时，将看到“匿名响应”。 当不需要“已验证”用户的操作需要“占位符”时，通常会看到这种情况。 匿名连接与Windows“来宾”用户不同（后者是实际用户帐户，而匿名连接则根本没有帐户关联）。

在匿名的Type 3消息中，客户端指示“ Negotiate Anonymous”标志。 NTLM响应字段为空（零长度）； LM响应字段包含单个空字节（“ 0x00”）。

## Type 3 消息示例
现在我们已经熟悉了类型3的响应，现在可以检查类型3的消息了：

    4e544c4d5353500003000000180018006a00000018001800
    820000000c000c0040000000080008004c00000016001600
    54000000000000009a0000000102000044004f004d004100
    49004e00750073006500720057004f0052004b0053005400
    4100540049004f004e00c337cd5cbd44fc9782a667af6d42
    7c6de67c20c2d3e77c5625a98c1c31e81847466b29b2df46
    80f39958fb8c213a9cc6

此消息被分解为：

| 偏移量 | 值                                                       | 说明                                                                                                                                  |
|--------|----------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| 0      | 0x4e544c4d53535000                                       | NTLMSSP Signature                                                                                                                     |
| 8      | 0x03000000                                               | Type 3 Indicator                                                                                                                      |
| 12     | 0x180018006a000000                                       | LM Response Security Buffer:<br>Length: 24 bytes (0x1800)<br>Allocated Space: 24 bytes (0x1800)<br>Offset: 106 bytes (0x6a000000)     |
| 20     | 0x1800180082000000                                       | NTLM Response Security Buffer:<br>Length: 24 bytes (0x1800)<br>Allocated Space: 24 bytes (0x1800)<br>Offset: 130 bytes (0x82000000)   |
| 28     | 0x0c000c0040000000                                       | Target Name Security Buffer:<br>Length: 12 bytes (0x0c00)<br>Allocated Space: 12 bytes (0x0c00)<br>Offset: 64 bytes (0x40000000)      |
| 36     | 0x080008004c000000                                       | User Name Security Buffer:<br>Length: 8 bytes (0x0800)<br>Allocated Space: 8 bytes (0x0800)<br>Offset: 76 bytes (0x4c000000)          |
| 44     | 0x1600160054000000                                       | Workstation Name Security Buffer:<br>Length: 22 bytes (0x1600)<br>Allocated Space: 22 bytes (0x1600)<br>Offset: 84 bytes (0x54000000) |
| 52     | 0x000000009a000000                                       | Session Key Security Buffer:<br>Length: 0 bytes (0x0000)<br>Allocated Space: 0 bytes (0x0000)<br>Offset: 154 bytes (0x9a000000)       |
| 60     | 0x01020000                                               | Flags:<br>Negotiate Unicode (0x00000001)<br>Negotiate NTLM (0x00000200)                                                               |
| 64     | 0x44004f004d004100   49004e00                            | Target Name Data ("DOMAIN")                                                                                                           |
| 76     | 0x7500730065007200                                       | User Name Data ("user")                                                                                                               |
| 84     | 0x57004f0052004b00   5300540041005400   49004f004e00     | Workstation Name Data ("WORKSTATION")                                                                                                 |
| 106    | 0xc337cd5cbd44fc97   82a667af6d427c6d   e67c20c2d3e77c56 | LM Response Data                                                                                                                      |
| 130    | 0x25a98c1c31e81847   466b29b2df4680f3   9958fb8c213a9cc6 | NTLM Response Data                                                                                                                    |

分析表明：

1. 这是一条NTLM Type 3消息（来自NTLMSSP签名和3类指示符）。
2. 客户端已指示使用Unicode编码字符串（已设置"Negotiate Unicode"Flags）。
3. 客户端支持NTLM身份验证（NegotiateNTLM）。
4. 客户的域是" DOMAIN "。
5. 客户端的用户名是" user "。
6. 客户的工作站是" WORKSTATION "。
7. 客户端的LM响应为" 0xc337cd5cbd44fc9782a667af6d427c6de67c20c2d3e77c56 "。
8. 客户端的NTLM响应为" 0x25a98c1c31e81847466b29b2df4680f39958fb8c213a9cc6 "。
9. 空的Session Key已发送。
10. 收到Type 3消息后，服务器将计算LM和NTLM响应，并将它们与客户端提供的值进行比较。如果它们匹配，则用户已成功通过身份验证。
