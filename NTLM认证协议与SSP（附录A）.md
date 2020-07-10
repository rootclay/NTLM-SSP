# NTLM认证协议与SSP（附录A）
请注意，由于Web具有高度动态性和瞬态性，因此这些功能可能可用或可能不可用。

jCIFS项目主页
http://jcifs.samba.org/
jCIFS是CIFS / SMB的开源Java实现。本文中提供的信息用作jCIFS NTLM身份验证实现的基础。jCIFS为NTLM HTTP身份验证方案的客户端和服务器端以及非协议特定的NTLM实用程序类提供支持。
Samba主页
http://www.samba.org/
Samba是一个开源CIFS / SMB服务器和客户端。实现NTLM身份验证和会话安全性，以及本文档大部分内容的参考。
实施CIFS：通用Internet文件系统
http://ubiqx.org/cifs/
Christopher R. Hertel撰写的，内容丰富的在线图书。与该讨论特别相关的是有关 身份验证的部分。
Open Group ActiveX核心技术参考（第11章，" NTLM"）
http://www.opengroup.org/comsource/techref2/NCH1222X.HTM
与NTLM上"官方"参考最接近的东西。不幸的是，它还很旧并且不够准确。
安全支持提供者界面
http://www.microsoft.com/windows2000/techinfo/howitworks/security/sspi2000.asp
白皮书，讨论使用SSPI进行应用程序开发。
HTTP的NTLM身份验证方案
http://www.innovation.ch/java/ntlm.html
有关NTLM HTTP身份验证机制的内容丰富的讨论。
Squid NTLM认证项目
http://squid.sourceforge.net/ntlm/
为Squid代理服务器提供NTLM HTTP身份验证的项目。
Jakarta Commons HttpClient
http://jakarta.apache.org/commons/httpclient/
一个开放源Java HTTP客户端，它提供对NTLM HTTP身份验证方案的支持。
GNU加密项目
http://www.gnu.org/software/gnu-crypto/
一个开放源代码的Java密码学扩展提供程序，提供了MD4消息摘要算法的实现。
RFC 1320-MD4消息摘要算法
http://www.ietf.org/rfc/rfc1320.txt
MD4摘要的规范和参考实现（用于计算NTLM密码哈希）。
RFC 1321-MD5消息摘要算法
http://www.ietf.org/rfc/rfc1321.txt
MD5摘要的规范和参考实现（用于计算NTLM2会话响应）。
RFC 2104-HMAC：消息身份验证的键哈希
http://www.ietf.org/rfc/rfc2104.txt
HMAC-MD5算法的规范和参考实现（用于NTLMv2 / LMv2响应的计算）。
如何启用NTLM 2身份验证
http://support.microsoft.com/default.aspx?scid=KB;zh-cn;239869
描述了如何启用NTLMv2身份验证的Negotiate并强制执行NTLM安全Flags。
Microsoft SSPI功能文档
http://windowssdk.msdn.microsoft.com/en-us/library/ms717571.aspx#sspi_functions
概述了安全支持提供程序接口（SSPI）和相关功能。