# NTLM认证协议与SSP（附录B）
# 附录B：NTLM的应用协议用法
本节研究了Microsoft的某些网络协议实现中NTLM身份验证的使用。
## NTLM HTTP身份验证
Microsoft已经为HTTP建立了专有的" NTLM"身份验证方案，以向IIS Web服务器提供集成身份验证。此身份验证机制允许客户端使用其Windows凭据访问资源，通常用于公司环境中，以向Intranet站点提供单点登录功能。从历史上看，Internet Explorer仅支持NTLM身份验证。但是，最近，已经向其他各种用户代理添加了支持。

NTLM HTTP身份验证机制的工作方式如下：

1. 客户端从服务器请求受保护的资源：
    GET /index.html HTTP / 1.1
2. 服务器以401状态响应，指示客户端必须进行身份验证。通过" WWW-Authenticate "标头将" NTLM"表示为受支持的身份验证机制。通常，服务器此时会关闭连接：
    HTTP / 1.1 401未经授权的
    WWW身份验证：NTLM 
    连接：关闭
请注意，如果Internet Explorer是第一个提供的机制，它将仅选择NTLM。这与RFC 2616不一致，RFC 2616指出客户端必须选择支持最强的身份验证方案。

3. 客户端使用包含Type 1消息参数的" Authorization "标头重新提交请求。Type 1消息经过Base-64编码以进行传输。从这一点开始，连接保持打开状态。关闭连接需要重新验证后续请求。这意味着服务器和客户端必须通过HTTP 1.0样式的" Keep-Alive"标头或HTTP 1.1（默认情况下采用持久连接）来支持持久连接。相关的请求标头显示如下（下面的" Authorization "标头中的换行符仅用于显示目的，在实际消息中不存在）：
    GET /index.html HTTP / 1.1 
    授权：NTLM TlRMTVNTUAABAAAABzIAAAYABgArAAAACwALACAAAABXT1 
    JLU1RBVElPTkRPTUFJTg ==
4. 服务器以401状态答复，该状态在" WWW-Authenticate "标头中包含Type 2消息（再次，以Base-64编码）。如下所示（" WWW-Authenticate "标头中的换行符仅出于编辑目的，在实际标头中不存在）。
    HTTP / 1.1 401未授权
    WWW验证：NTLM TlRMTVNTUAACAAAADAAMADAAAAABAoEAASNFZ4mrze8 
    AAAAAAAAAAGIAYgA8AAAARABPAE0AQQBJAE4AAgAMAEQATwBNAEEASQBOAAEADABTA 
    EUAUgBWAEUAUgAEABQAZABvAG0AYQBpAG4ALgBjAG8AbQADACIAcwBlAHIAdgBlAHI 
    ALgBkAG8AbQBhAGkAbgAuAGMAbwBtAAAAAAA =
5. 客户端通过使用包含包含Base-64编码的Type 3消息的" Authorization "标头重新提交请求来响应Type 2消息（同样，下面的" Authorization "标头中的换行符仅用于显示目的）：
    GET /index.html HTTP / 1.1 
    授权：NTLM TlRMTVNTUAADAAAAGAAYAGoAAAAYABgAggAAAAwADABAAA 
    AACAAIAEwAAAAWABYAVAAAAAAAAACaAAAAAQIAAEQATwBNAEEASQBOAHUAcwBlAHIA 
    VwBPAFIASwBTAFQAQQBUAEkATwBOAMM3zVy9RPyXgqZnr21CfG3mfCDC0 + d8ViWpjB 
    wx6BhHRmspst9GgPOZWPuMITqcxg ==
6. 最后，服务器验证客户端的Type 3消息中的响应，并允许访问资源。
    HTTP / 1.1 200 OK

此方案与大多数"常规" HTTP身份验证机制不同，因为通过身份验证的连接发出的后续请求本身不会被身份验证；NTLM是面向连接的，而不是面向请求的。因此，对" /index.html " 的第二个请求将不携带任何身份验证信息，并且服务器将不请求任何身份验证信息。如果服务器检测到与客户端的连接已断开，则对" /index.html " 的请求将导致服务器重新启动NTLM握手。

上面的一个显着例外是客户端在提交POST请求时的行为（通常在客户端向服务器发送表单数据时使用）。如果客户端确定服务器不是本地主机，则客户端将通过活动连接启动POST请求的重新认证。客户端将首先提交一个空的POST请求，并在" Authorization "标头中带有Type 1消息。服务器以Type 2消息作为响应（如上所示，在" WWW-Authenticate "标头中）。然后，客户端使用Type 3消息重新提交POST，并随请求发送表单数据。

NTLM HTTP机制也可以用于HTTP代理身份验证。该过程类似，除了：

* 服务器使用407响应代码（指示需要进行代理身份验证）而不是401。
* 客户端的1类和3类消息是在" 代理授权 "请求标头中发送的，而不是在" 授权 "标头中发送的。
* 服务器的Type 2质询在" Proxy-Authenticate "响应头中发送（而不是" WWW-Authenticate "）。

在Windows 2000中，Microsoft引入了"Negotiate" HTTP身份验证机制。虽然其主要目的是提供一种通过Kerberos通过Active Directory验证用户身份的方法，但它与NTLM方案向后兼容。当在"传统"模式下使用Negotiate机制时，在客户端和服务器之间传递的标头是相同的，只是将"Negotiate"（而不是" NTLM"）指定为机制名称。

## NTLM POP3身份验证
Microsoft的Exchange服务器为POP3协议提供了NTLM身份验证机制。这是RFC 1734中记录的与POP3 AUTH命令 一起使用的专有扩展 。在客户端，Outlook和Outlook Express支持此机制，称为"安全密码身份验证"。

POP3 NTLM身份验证握手在POP3"授权"状态期间发生，其工作方式如下：

1. 客户端可以通过发送不带参数的AUTH命令来请求支持的身份验证机制的列表：
    AUTH
2. 服务器以成功消息响应，然后是受支持机制的列表；此列表应包含" NTLM "，并以包含单个句点（" 。 "）的行结尾。
    +OK The operation completed successfully.
    NTLM
    .
3. 客户端通过发送一个将NTLM指定为身份验证机制的AUTH命令来启动NTLM身份验证：
    AUTH NTLM
4. 服务器将显示一条成功消息，如下所示。注意，" + "和" OK " 之间有一个空格；RFC 1734指出服务器应以质询进行答复，但是NTLM要求来自客户端的Type 1消息。因此，服务器发送"非challenge"消息，基本上是消息" OK "。
    +OK
5. 然后，客户端发送Type 1消息，以Base-64编码进行传输：
    TlRMTVNTUAABAAAABzIAAAYABgArAAAACwALACAAAABXT1JLU1RBVElPTkRPTUFJTg ==
6. 服务器回复Type 2质询消息（再次，以Base-64编码）。它以RFC 1734指定的质询格式发送（" + "，后跟一个空格，后跟质询消息）。如下所示；换行符是出于编辑目的，不出现在服务器的答复中：
    + TlRMTVNTUAACAAAADAAMADAAAAABAoEAASNFZ4mrze8AAAAAAAAAAGIAYgA8AAAA 
    RABPAE0AQQBJAE4AAgAMAEQATwBNAEEASQBOAAEADABTAEUAUgBWAEUAUgAEABQAZA 
    BvAG0AYQBpAG4ALgBjAG8AbQADACIAcwBlAHIAdgBlAHIALgBkAG8AbQBhAGkAbgAu 
    AGMAbwBtAAAAAAA =
7. 客户端计算并发送Base-64编码的Type 3响应（下面的换行符仅用于显示目的）：
    TlRMTVNTUAADAAAAGAAYAGoAAAAYABgAggAAAAwADABAAAAACAAIAEwAAAAWABYAVA 
    AAAAAAAACaAAAAAQIAAEQATwBNAEEASQBOAHUAcwBlAHIAVwBPAFIASwBTAFQAQQBU 
    AEkATwBOAMM3zVy9RPyXgqZnr21CfG3mfCDC0 + d8ViWpjBwx6BhHRmspst9GgPOZWP 
    uMITqcxg ==
8. 服务器验证响应并指示认证结果：
    +OK User successfully logged on

成功进行身份验证后，POP3会话进入"事务"状态，从而允许客户端检索消息。

## NTLM IMAP身份验证
Exchange提供了一种IMAP身份验证机制，其形式类似于前面讨论的POP3机制。RFC 1730中记录了IMAP身份验证 ；NTLM机制是Exchange提供的专有扩展，并由Outlook客户端家族支持。

握手序列类似于POP3机制：

1. 服务器可以在能力响应中指示对NTLM身份验证机制的支持。连接到IMAP服务器后，客户端将请求服务器功能列表：
    0000 CAPABILITY
2. 服务器以支持的功能列表作为响应；服务器回复中字符串" AUTH = NTLM " 的存在指示了NTLM身份验证扩展名：
    * CAPABILITY IMAP4 IMAP4rev1 IDLE LITERAL+ AUTH=NTLM
    0000 OK CAPABILITY completed.
3. 客户端通过发送 将NTLM指定为身份验证机制的AUTHENTICATE命令来启动NTLM身份验证：
    0001授权NTLM
4. 服务器以一个空的质询作为响应，该质询仅由" + "组成：
    +
5. 然后，客户端发送Type 1消息，以Base-64编码进行传输：
    TlRMTVNTUAABAAAABzIAAAYABgArAAAACwALACAAAABXT1JLU1RBVElPTkRPTUFJTg ==
6. 服务器回复Type 2质询消息（再次，以Base-64编码）。它以RFC 1730指定的质询格式发送（" + "，后跟一个空格，后跟质询消息）。如下所示；换行符是出于编辑目的，不出现在服务器的答复中：
    + TlRMTVNTUAACAAAADAAMADAAAAABAoEAASNFZ4mrze8AAAAAAAAAAGIAYgA8AAAA 
    RABPAE0AQQBJAE4AAgAMAEQATwBNAEEASQBOAAEADABTAEUAUgBWAEUAUgAEABQAZA 
    BvAG0AYQBpAG4ALgBjAG8AbQADACIAcwBlAHIAdgBlAHIALgBkAG8AbQBhAGkAbgAu 
    AGMAbwBtAAAAAAA =
7. 客户端计算并发送Base-64编码的Type 3响应（下面的换行符仅用于显示目的）：
    TlRMTVNTUAADAAAAGAAYAGoAAAAYABgAggAAAAwADABAAAAACAAIAEwAAAAWABYAVA 
    AAAAAAAACaAAAAAQIAAEQATwBNAEEASQBOAHUAcwBlAHIAVwBPAFIASwBTAFQAQQBU 
    AEkATwBOAMM3zVy9RPyXgqZnr21CfG3mfCDC0 + d8ViWpjBwx6BhHRmspst9GgPOZWP 
    uMITqcxg ==
8. 服务器验证响应并指示认证结果：
    0001 OK AUTHENTICATE NTLM completed.

身份验证完成后，IMAP会话将进入身份验证状态。

## NTLM SMTP身份验证
除了为POP3和IMAP提供的NTLM身份验证机制外，Exchange还为SMTP协议提供了类似的功能。这样可以对发送外发邮件的用户进行NTLM身份验证。这是与SMTP AUTH命令一起使用的专有扩展（在 RFC 2554中记录）。

SMTP NTLM身份验证握手的操作如下：

1. 服务器可以在EHLO答复中指示支持NTLM作为身份验证机制。连接到SMTP服务器后，客户端将发送初始EHLO消息：
    EHLO client.example.com
2. 服务器以支持的扩展列表进行响应。NTLM身份验证扩展由其在AUTH机制列表中的存在指示，如下所示。请注意，AUTH 列表发送了两次（一次带有" = "，一次没有）。显然在RFC草案中指定了" AUTH = "形式。发送两种表格都可以确保支持针对该草案实施的客户。
    250-server.example.com Hello [10.10.2.20]
    250-HELP
    250-AUTH LOGIN NTLM
    250-AUTH=LOGIN NTLM
    250 SIZE 10240000
3. 客户端通过发送一个AUTH 命令来启动NTLM身份验证，该命令将NTLM指定为身份验证机制，并提供Base-64编码的Type 1消息作为参数：
    
    AUTH NTLM TlRMTVNTUAABAAAABzIAAAYABgArAAAACwALACAAAABXT1JLU1RBVElPTkRPTUFJTg ==

根据RFC 2554，客户端可以选择不发送初始响应参数（而是仅发送" AUTH NTLM "并等待空服务器质询，然后以Type 1消息答复）。但是，在针对Exchange测试时，这似乎无法正常工作。

4. 服务器回复334响应，其中包含Type 2质询消息（同样是Base-64编码）。如下所示；换行符是出于编辑目的，不出现在服务器的答复中：
    334 TlRMTVNTUAACAAAADAAMADAAAAABAoEAASNFZ4mrze8AAAAAAAAAAGIAYgA8AAAA 
    RABPAE0AQQBJAE4AAgAMAEQATwBNAEEASQBOAAEADABTAEUAUgBWAEUAUgAEABQAZA 
    BvAG0AYQBpAG4ALgBjAG8AbQADACIAcwBlAHIAdgBlAHIALgBkAG8AbQBhAGkAbgAu 
    AGMAbwBtAAAAAAA =
5. 客户端计算并发送Base-64编码的Type 3响应（下面的换行符仅用于显示目的）：
    TlRMTVNTUAADAAAAGAAYAGoAAAAYABgAggAAAAwADABAAAAACAAIAEwAAAAWABYAVA 
    AAAAAAAACaAAAAAQIAAEQATwBNAEEASQBOAHUAcwBlAHIAVwBPAFIASwBTAFQAQQBU 
    AEkATwBOAMM3zVy9RPyXgqZnr21CfG3mfCDC0 + d8ViWpjBwx6BhHRmspst9GgPOZWP 
    uMITqcxg ==
6. 服务器验证响应并指示认证结果：

    235 NTLM authentication successful.

验证后，客户端可以正常发送消息。
