# NTLM认证协议与SSP(下)
内容参考原文链接：http://davenport.sourceforge.net/ntlm.html
翻译人：rootclay（香山）https://github.com/rootclay

# 说明 
本文是一篇NTLM中高级进阶进阶文章，文中大部分参考来自于[Sourceforge](http://davenport.sourceforge.net/ntlm.html)，原文中已经对NTLM讲解非常详细，在学习的过程中思考为何不翻译之，做为学习和后续回顾的文档，并在此基础上添加自己的思考，因此出现了这篇文章，在翻译的过程中会有部分注解与新加入的元素，后续我也会在Github对此文进行持续性的更新NTLM以及常见的协议中高级进阶并计划开源部分协议调试工具，望各位issue勘误。


# 摘要

本文旨在以中级到高级的详细级别描述NTLM身份验证协议(authentication protocol)和相关的安全支持提供程序功能(security support provider functionality)，作为参考。希望该文档能发展成为对NTLM的全面描述。目前，无论是在作者的知识还是在文档方面，都存在遗漏，而且几乎可以肯定的说本文是不准确的。但是，该文档至少应能够为进一步研究提供坚实的基础。本文提供的信息用作在开放源代码jCIFS库中实现NTLM身份验证的基础，该库可从 http://jcifs.samba.org获得。本文档基于作者的独立研究，并分析了[Samba](http://www.samba.org/)软件套件。

# NTLM版本2(NTLM Version 2)
NTLM版本2包含三种新的响应算法（NTLMv2，LMv2和NTLM2会话响应，如前所述）和新的签名和Sealing方案（NTLM2会话安全性）。NTLM2会话安全性是通过"Negotiate NTLM2 Key"FlagsNegotiate的；但是，可以通过修改注册表来启用NTLMv2身份验证。此外，客户端和域控制器上的注册表设置必须兼容才能成功进行身份验证（尽管NTLMv2身份验证有可能通过较旧的服务器传递到NTLMv2域控制器）。部署NTLMv2所需的配置和计划的结果是，许多主机仅使用默认设置（NTLMv1），而不怎么使用NTLMv2进行身份验证。

Microsoft知识库文章239869 中详细介绍了启用NTLM版本2的说明 。简要地，对注册表值进行了修改：
`HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\LSA\LMCompatibilityLevel`
（基于Win9x的系统上的LMCompatibility）。这是一个 REG_DWORD条目，可以设置为以下值之一：

| Level | Sent by Client |      Accepted by Server      |
|:-----|:--------------|:----------------------------|
| 0     | LM<br>NTLM     | LM<br>NTLM<br>LMv2<br>NTLMv2 |
| 1     | LM<br>NTLM     | LM<br>NTLM<br>LMv2<br>NTLMv2 |
| 2     | NTLM           | LM<br>NTLM<br>LMv2<br>NTLMv2 |
| 3     | LMv2<br>NTLMv2 | LM<br>NTLM<br>LMv2<br>NTLMv2 |
| 4     | LMv2<br>NTLMv2 | NTLM<br>LMv2<br>NTLMv2       |
| 5     | LMv2<br>NTLMv2 | LMv2<br>NTLMv2               |

在所有级别中，都支持NTLM2会话安全性并在可用时进行Negotiate（大多数可用文档表明NTLM2会话安全性仅在级别1和更高级别上启用，但实际上在级别0上也可以看到）。默认情况下，在Windows 95和Windows 98平台上仅支持LM响应。安装Directory Services之后的客户端使NTLMv2也可以在这些主机上使用（并启用LMCompatibility 设置，尽管仅级别0和3可用）。

在级别2中，客户端两次发送NTLM响应（在LM和NTLM响应字段中）。在级别3和更高级别，LMv2和NTLMv2响应分别替换LM和NTLM响应。

协商了NTLM2会话安全性后（由"Negotiate NTLM2 Key"Flags指示），可以在级别0、1和2中使用NTLM2会话响应来代替较弱的LM和NTLM响应。与NTLMv1相比，这可以提供针对基于服务器的预先计算的字典攻击的增强保护。通过向计算中添加随机的客户随机数，可以使客户对给定challenge的响应变得可变。

NTLM2 session response很有趣，因为它可以在支持较新方案的客户端和服务器之间进行Negotiate，即使存在不支持较旧域控制器的情况也是如此。在通常情况下，身份验证事务中的服务器实际上并不拥有用户的密码哈希；而是保留在域控制器中。将计算机加入使用NT风格认证的域时，它会建立到域控制器（俗称" NetLogon pipe"）的经过加密，相互认证的通道。当客户端使用"原始" NTLMv1握手向服务器进行身份验证时，在后台进行以下事务：

1. 客户端发送Type 1消息，其中包含Flags和其他信息，如前所述。
2. 服务器为客户端生成一个质询，并发送包含Negotiate Flags集的Type 2消息。
3. 客户响应challenge，提供LM / NTLM响应。
4. 服务器通过NetLogon管道将质询和客户端响应发送到域控制器。
5. 域控制器使用存储的哈希值和服务器给出的质询来重现身份验证计算。如果它们与响应匹配，则认证成功。
6. 域控制器计算Session Key并将其发送到服务器，该Session Key可用于服务器和客户端之间的后续签名和Sealing操作。

在NTLM2会话响应的情况下，可能已升级了客户端和服务器以允许较新的协议，而域控制器却没有。为了考虑到这种情况，对上述握手进行了如下修改：

1. 客户端发送Type 1消息，在这种情况下，该消息指示"Negotiate NTLM2 Key"Flags。
2. 服务器为客户端生成质询，并发送包含NegotiateFlags集（还包括"Negotiate NTLM2 Key"Flags）的Type 2消息。
3. 客户端响应challenge，在LM字段中提供client nonce，并在NTLM字段中提供NTLM2会话响应（NTLM2 Session Response）。请注意，后者的计算与NTLM响应完全相同，只是客户端没有对服务器质询进行加密，而是对与client nonce连接的服务器质询的MD5哈希进行了加密。
4. 服务器不是将服务器质询直接通过NetLogon管道直接发送到域控制器，而是将服务器质询的MD5哈希与client nonce连接在一起（从LM响应字段中提取）。此外，它还发送客户端响应（照常）。
5. 域控制器使用存储的哈希作为Key对服务器发送的质询字段进行加密，并验证它与NTLM响应字段匹配；因此，客户端已成功通过身份验证。
6. 域控制器计算正常的NTLM用户Session Key并将其发送到服务器；服务器在次要计算中使用它来获取NTLM2会话响应用户Session Key（在后续部分中讨论 ）
本质上，这允许已升级的客户端和服务器在尚未将域控制器升级到NTLMv2（或者网络管理员尚未将LMCompatibilityLevel注册表设置配置为使用NTLMv2）的网络中使用NTLM2会话响应。

与LMCompatibilityLevel设置相关的是 NtlmMinClientSec和NtlmMinServerSec设置。这些规定了由NTLMSSP建立的NTLM上下文的最低要求。两者都是 REG_WORD条目，并且是指定以下NTLMFlags组合的位域：

1. Negotiate Sign（0x00000010）-指示必须在支持消息完整性（签名）的情况下建立上下文。
2. Negotiate Seal（0x00000020）-指示必须在支持消息机密性（Sealing）的情况下建立上下文。
3. Negotiate NTLM2 Key（0x00080000）-指示必须使用NTLM2会话安全性来建立上下文。
4. Negotiate 128（0x20000000）-指示上下文必须至少支持128位签名/SealingKey。
5. Negotiate 56（0x80000000）-指示上下文必须至少支持56位签名/SealingKey。

尽管其中大多数都更适用于NTLM2签名和Sealing，但"Negotiate NTLM2 Key"对于身份验证很重要，因为它可以防止与无法NegotiateNTLM2会话安全性的主机建立会话。这用于确保不发送LM和NTLM响应（要求认证在所有情况下至少将使用NTLM2会话响应）。

# NTLMSSP和SSPI
在这一点上，我们将开始研究NTLM如何适应"大局"（big picture）。关于SSPI内容也可以查看本链接中的简单说明[SSPI](https://daiker.gitbook.io/windows-protocol/ntlm-pian/4#0x04-ssp-and-sspi)

Windows提供了一个称为SSPI的安全框架-安全支持提供程序接口。这与GSS-API（通用安全服务应用程序接口，RFC 2743 ）在Microsoft中等效。 ），并允许应用认证，完整性和机密性原语的非常高级的机制无关的方法。SSPI支持多个基础提供程序（Kerberos、Cred SSP、Digest SSP、Negotiate SSP、Schannel SSP、Negotiate Extensions SSP、PKU2U SSP）。其中之一就是NTLMSSP（NTLM安全支持提供程序），它提供了到目前为止我们一直在讨论的NTLM身份验证机制。SSPI提供了一个灵活的API，用于处理不透明的，特定于提供程序的身份验证令牌。NTLM Type 1，Type 2和Type 3消息就是此类令牌，专用于NTLMSSP并由其处理。SSPI提供的API几乎抽象了NTLM的所有细节。应用程序开发人员甚至不必知道正在使用NTLM，并且可以交换另一种身份验证机制（例如Kerberos），而在应用程序级别进行的更改很少或没有更改。

在系统层面，SSP就是一个dll，来实现身份验证等安全功能，实现的身份验证机制是不一样的。比如 NTLM SSP 实现的就是一种 Challenge/Response 验证机制。而 Kerberos 实现的就是基于 ticket 的身份验证机制。我们可以编写自己的 SSP，然后注册到操作系统中，让操作系统支持更多的自定义的身份验证方法。


我们不会对SSPI框架进行深入研究，但这是研究应用于NTLM的SSPI身份验证握手的好方法：

1. 客户端通过SSPI AcquireCredentialsHandle函数为用户获取证书集的表示。
2. 客户端调用SSPI InitializeSecurityContext函数以获得身份验证请求令牌（在我们的示例中为Type 1消息）。客户端将此令牌发送到服务器。该函数的返回值表明身份验证将需要多个步骤。
3. 服务器从客户端接收令牌，并将其用作AcceptSecurityContext SSPI函数的输入 。这将在服务器上创建一个表示客户端的本地安全上下文，并生成一个身份验证响应令牌（Type 2消息），该令牌将发送到客户端。该函数的返回值指示需要客户端提供更多信息。
4. 客户端从服务器接收响应令牌，然后再次调用 InitializeSecurityContext，并将服务器的令牌作为输入传递。这为我们提供了另一个身份验证请求令牌（Type 3消息）。返回值指示安全上下文已成功初始化；令牌已发送到服务器。
5. 服务器从客户端接收令牌，并使用Type 3消息作为输入再次调用 AcceptSecurityContext。返回值指示上下文已成功接受；没有令牌产生，并且认证完成。

## 本地认证（Local Authentication）
我们在讨论的各个阶段都提到了本地身份验证序列。对SSPI有基本的了解后，我们可以更详细地研究这种情况。

基于NTLM消息中的信息，客户端和服务器通过一系列决策来协商本地身份验证。其工作方式如下：

1. 客户端调用AcquireCredentialsHandle函数，通过将null传递给"pAuthData"参数来指定默认凭据。这将获得用于单点登录的登录用户凭据的句柄。
2. 客户端调用SSPI InitializeSecurityContext函数来创建Type 1消息。提供默认凭据句柄时，Type 1消息包含客户端的工作站和域名。这由"Negotiate Domain Supplied"和"Negotiate Workstation Supplied"Flags的存在以及消息中包含已填充的"已提供的域（Supplied Domain）"和"工作站的安全性（Supplied Workstation security）"标记来表明。
3. 服务器从客户端接收Type 1消息，并调用 AcceptSecurityContext。这将在服务器上创建一个代表客户端的本地安全上下文。服务器检查客户端发送的域和工作站信息，以确定客户端和服务器是否在同一台计算机上。如果是这样，则服务器通过在结果2类消息中设置"Negotiate Local Call"Flags来启动本地身份验证。Type 2消息的Context字段中的第一个long填充了新获得的SSPI上下文句柄的"upper"部分（特别是SSPI CtxtHandle结构的" dwUpper"字段）。第二个long在所有情况下，"上下文"字段中的"空白"都为空。（尽管从逻辑上讲，它会假定它应包含上下文句柄的"下部"部分）。
4. 客户端从服务器接收Type 2消息，并将其传递给 InitializeSecurityContext。注意了"Negotiate Local Call"Flags的存在之后，客户端检查服务器上下文句柄以确定它是否代表有效的本地安全上下文。如果无法验证上下文，则身份验证将照常进行-计算适当的响应，并将其包含在Type 3消息中的域，工作站和用户名中。如果来自Type 2消息的安全上下文句柄可以验证，但是，没有准备任何答复。而是，默认凭据在内部与服务器上下文相关联。生成的Type 3消息完全为空，其中包含响应长度为零的安全缓冲区以及用户名，域和工作站。
5. 服务器收到Type 3消息，并将其用作AcceptSecurityContext函数的输入 。服务器验证安全上下文已与用户关联；如果是这样，则认证已成功完成。如果上下文尚未绑定到用户，则身份验证失败。

## 数据报认证（Datagram Authentication）（面向无连接）
数据报样式验证用于通过无连接传输Negotiate NTLM。尽管消息周围的许多语义保持不变，但仍存在一些重大差异：

1. 在第一次调用InitializeSecurityContext的过程中，SSPI不会创建Type 1消息 。
2. 身份验证选项由服务器提供，而不是由客户端请求。
3. Type 3消息中的Flags将会有用（如在面向连接的身份验证中）。

在"normal"（面向连接）身份验证期间，在交换Type 1和Type 2消息期间，所有选项都在客户端和服务器之间的第一个事务中Negotiate。Negotiate的设置由服务器"remembered"，并应用于客户端的Type 3消息。尽管大多数客户端发送带有Type 3消息的Negotiate一致的Flags，但它们未用于连接身份验证。（注：也就是Type3消息的Flag是没有用的）

但是，在数据报身份验证中，规则发生了一些变化。为了减轻服务器跟踪Negotiate选项的需要（如果没有持久连接，这将变得困难），将Type 1消息完全删除。服务器生成包含所有受支持Flags的Type 2消息（当然还有质询）。然后，客户端决定它将支持哪些选项，并以Type 3消息进行答复，其中包含对质询的响应和一组选定Flags。数据报认证的SSPI握手序列如下：

1. 客户端调用AcquireCredentialsHandle以获得用户证书集的表示。
2. 客户端调用InitializeSecurityContext，并通过fContextReq参数将 ISC_REQ_DATAGRAMFlags作为上下文要求传递。这将启动客户端的安全上下文的建设，但并没有产生令牌的请求（Type 1的消息）。
3. 服务器调用AcceptSecurityContext函数，指定 ASC_REQ_DATAGRAM上下文要求Flags并传入空输入令牌。这将创建本地安全上下文，并生成身份验证响应令牌（Type 2消息）。此Type 2消息将包含"Negotiate数据报样式"Flags，以及服务器支持的所有Flags。照常发送给客户端。
4. 客户端收到Type 2消息，并将其传递给 InitializeSecurityContext。客户端从服务器提供的选项中选择适当的选项（包括必须设置的"Negotiate数据报样式"），创建对质询的响应，并填充Type 3消息。然后，该消息将中继到服务器。
5. 服务器将Type 3消息传递到AcceptSecurityContext 函数中。根据客户端选择的Flags来处理消息，并且上下文被成功接受。

与SSPI一起使用时，显然无法产生数据报样式的Type 1消息。但是，有趣的是，我们可以通过巧妙地操纵NTLMSSP令牌来产生我们自己的数据报Type 1令牌，从而在较低级别上"诱导"数据报语义。

这可以通过在将令牌传递到服务器之前，在面向连接的SSPI握手中在第一个InitializeSecurityContext调用产生的Type 1消息上设置"NegotiateNegotiate Datagram Style"Flags来实现。当将修改后的Type 1消息传递到 AcceptSecurityContext函数中时，服务器将采用数据报语义（即使未指定ASC_REQ_DATAGRAM）。这将产生设置了"Negotiate Datagram Style"Flags的2类消息，但与通常会生成的面向连接的消息相同；也就是说，在构造Type 2消息时会考虑客户端发送的Type 1Flags，而不是简单地提供所有受支持的选项。

然后，客户端可以使用此Type 2令牌调用InitializeSecurityContext。请注意，客户端仍处于面向连接的模式。生成的Type 3消息将忽略应用于Type 2消息的"Negotiate Datagram Style"Flags。但是，服务器正在执行数据报语义，并且现在将要求正确设置Type 3Flags。在将"Negotiate Datagram Style"Flags添加到Type 3消息之前，将其手动发送到服务器之前，可以使服务器使用修改后的令牌成功调用 AcceptSecurityContext。

这样可以成功进行身份验证；"篡改"Type 1消息有效地将服务器切换到数据报式身份验证，其中将观察并强制使用Type 3Flags。目前没有已知的实际用途，但是它确实演示了可以通过策略性地处理NTLM消息来观察到的一些有趣和意外的行为。

# 会话安全性-签名和盖章概念（Session Security - Signing & Sealing Concepts）
除了SSPI身份验证服务，还提供了消息完整性和机密性功能。这也由NTLM安全支持提供程序实现。"签名"由SSPI MakeSignature函数执行，该函数将消息验证码（MAC）应用于消息（message）。收件人可以对此进行验证，并且可以强有力地确保消息在传输过程中没有被修改。签名是使用发送方和接收方已知的Key生成的；MAC只能由拥有Key的一方来验证（这反过来可以确保签名是由发送方创建的）。"Sealing"由SSPI EncryptMessage执行功能。这会对消息应用加密，以防止传输中的第三方查看它（类似HTPPS）；NTLMSSP使用多种对称加密机制（使用相同的Key进行解密和加密）。

NTLM身份验证过程的同时会建立用于签名和Sealing的Key。除了验证客户端的身份外，身份验证握手还在客户端和服务器之间建立了一个上下文，其中包括在各方之间签名和Sealing消息所需的Key。我们将讨论这些Key的产生以及NTLMSSP用于签名和Sealing的机制。

在签名和盖章过程中采用了许多关键方案。我们将首先概述不同Type的Key和核心会话安全性概念。

## The User Session Key
这是会话安全中使用的基本Key Type。有很多变体：

* LM User Session Key
* NTLM User Session Key
* LMv2 User Session Key
* NTLMv2 User Session Key
* NTLM2 Session Response User Session Key

所使用的推导方法取决于Type 3消息中发送的响应。这些变体及其计算概述如下。

### LM User Session Key
仅在提供LM响应时（即，对于Win9x客户端）使用。LM用户Session Key的得出如下：

1. 16字节LM哈希（先前计算）被截断为8字节。
2. 将其空填充为16个字节。该值是LM用户Session Key。

与LM哈希本身一样，此Key仅响应于用户更改密码而更改。还要注意，只有前7个密码字符输入了Key（请参阅LM响应的计算过程 ； LM用户Session Key是LM哈希的前半部分）。此外，Key空间实际上要小得多，因为LM哈希本身基于大写密码。所有这些因素加在一起使得LM用户Session Key非常难以抵抗攻击。

### NTLM User Session Key
客户端发送NTLM响应时，将使用此变体。Key的计算非常简单：

1. 获得NTLM哈希（Unicode大小写混合的密码的MD4摘要，先前已计算）。
2. MD4消息摘要算法应用于NTLM哈希，结果为16字节。这是NTLM用户Session Key。

NTLM用户Session Key比LM用户Session Key有了很大的改进。密码空间更大（区分大小写，而不是将密码转换为大写）；此外，所有密码字符都已输入到Key生成中。但是，它仍然仅在用户更改其密码时才更改。这使得离线攻击变得更加容易。

### LMv2 User Session Key
发送LMv2响应（但不发送NTLMv2响应）时使用。派生此Key有点复杂，但并不十分复杂：

1. 获得NTLMv2哈希（如先前计算的那样）。
2. 获得LMv2client nonce（用于LMv2响应）。
3. 来自Type 2消息的质询与client nonce串联在一起。使用NTLMv2哈希作为Key，将HMAC-MD5消息认证代码算法应用于此值，从而得到16字节的输出值。
4. 再次使用NTLMv2哈希作为Key，将HMAC-MD5算法应用于该值。结果为16个字节的值是LMv2 User Session Key。

LMv2 User Session Key相对于基于NTLMv1的Key提供了一些改进。它是从NTLMv2哈希派生而来的（它本身是从NTLM哈希派生的），它特定于用户名和域/服务器。此外，服务器质询和client nonce都为Key计算提供输入。Key计算也可以简单地表示为LMv2响应的前16个字节的HMAC-MD5摘要（使用NTLMv2哈希作为Key）。

### NTLMv2 User Session Key
发送NTLMv2响应时使用。该Key的计算与LMv2用户Session Key非常相似：

1. 获得NTLMv2哈希（如先前计算的那样）。
2. 获得NTLMv2"blob"（与NTLMv2响应中使用的一样）。
3. 来自Type 2消息的challenge与Blob连接在一起作为待加密值。使用NTLMv2哈希作为Keykey，将HMAC-MD5消息认证代码算法应用于此值，从而得到16字节的输出值。
4. 再次使用NTLMv2哈希作为Key，将HMAC-MD5算法应用于第三步的值。结果为16个字节的值是NTLMv2用户Session Key。

NTLMv2 User Session Key在密码上与LMv2 User Session Key非常相似。可以说是NTLMv2响应的前16个字节的HMAC-MD5摘要（使用NTLMv2哈希作为关键字）。

### NTLM2 Session Response User Session Key
当NTLMv1身份验证与NTLM2会话安全性一起使用时使用。该Key是从NTLM2会话响应信息中派生的，如下所示：

1. 如前所述，将获得NTLM User Session Key。
2. 获得session nonce（先前已讨论过，这是Type 2质询和NTLM2会话响应中的随机数的串联）。
3. 使用NTLM User Session Key作为Key，将HMAC-MD5算法应用于session nonce。结果为16个字节的值是NTLM2会话响应用户Session Key。

NTLM2会话响应用户Session Key的显着之处在于它是在客户端和服务器之间而不是在域控制器上计算的。域控制器像以前一样导出NTLM用户Session Key，并将其提供给服务器。如果已经与客户端Negotiate了NTLM2会话安全性，则服务器将使用NTLM用户Session Key作为MACKey来获取session nonce的HMAC-MD5摘要。

### 空用户Session Key（The Null User Session Key）
当执行匿名身份验证时，将使用Null用户Session Key。这很简单；它只有16个空字节（" 0x000000000000000000000000000000000000 "）。

## Lan Manager Session Key
Lan Manager Session Key是User Session Key的替代方法，用于在设置"Negotiate Lan Manager Key" NTLM Flags时派生NTLM1签名和Sealing中的Key。Lan ManagerSession Key的计算如下：

1. 16字节LM哈希（先前计算）被截断为8字节。
2. 这将填充为14个字节，其值为"0xbdbdbdbdbdbdbd "。
3. 该值分为两个7字节的一半。
4. 这些值用于创建两个DESKey（每个7字节的一半为一个）。
5. 这些Key中的每一个都用于对LM响应的前8个字节进行DES加密（导致两个8字节密文值）。
6. 这两个密文值连接在一起形成一个16字节的值-Lan ManagerSession Key。

请注意，Lan ManagerSession Key基于LM响应（而不是简单的LM哈希），这意味着它将响应于不同的服务器challenge而更改。与仅基于密码哈希的LM和NTLM用户Session Key相比，这是一个优势。Lan ManagerSession Key会针对每个身份验证操作进行更改，而LM / NTLM用户Session Key将保持不变，直到用户更改其密码为止。因此，Lan ManagerSession Key比LM用户Session Key（两者具有相似的Key强度，但Lan ManagerSession Key可以防止重放攻击）要强得多。NTLM用户Session Key具有完整的128位Key空间，但与LM用户Session Key一样，在每次身份验证时也不相同。

## Key Exchange（密钥交换）
当设置"Negotiate Key Exchange"Flags时，客户端和服务器将会就"secondary"Key达成共识，该Key用于代替Session Key进行签名和Sealing。这样做如下：

1. 客户端选择一个随机的16字节Key（辅助Key，也就是py-ntlm中的exported_session_key）。
2. Session Key（User Session Key或Lan Manager Session Key，取决于"Negotiate Lan  ManagerKey"Flags的状态）用于RC4加密辅助Key。结果是一个16字节的密文值（注：也就是py-ntlm中的encrypted_random_session_key）。
3. 此值在Type 3消息的"Session Key"字段中发送到服务器。
4. 服务器接收Type 3消息并解密客户端发送的值（使用带有用户Session Key或Lan ManagerSession Key的RC4）。
5. 结果值是恢复的辅助Key，并代替Session Key进行签名和Sealing。

此外，密钥交换过程巧妙地更改了NTLM2会话安全性中的签名协议（在后续部分中讨论）。

## 弱化Key（Key Weakening）
根据加密输出限制，用于签名和Sealing的Key已被弱化（"weakened"）（注：可能是由于加密性能原因）。Key强度由"Negotiate128"和"Negotiate56"Flags确定。使用的最终Key的强度是客户端和服务器都支持的最大强度。如果两个Flags都未设置，则使用默认的Key长度40位。NTLM1签名和Sealing支持40位和56位Key；NTLM2会话安全性支持40位，56位和不变的128位Key。

# NTLM1会话安全
NTLM1是"原始" NTLMSSP签名和Sealing方案，在未协商"Negotiate NTLM2 Key"Flags时使用。此方案中的Key派生由以下NTLM的Flags驱动：

| Flag                      | 说明                                                                                                                                                                                                                                |
|---------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Negotiate Lan Manager Key | 设置后，Lan Manager会话密钥将用作签名和密封密钥（而不是用户会话密钥）的基础。 如果未建立，则用户会话密钥将用于密钥派生。（When set, the Lan Manager Session Key is used as the basis for the signing and sealing keys (rather than the User Session Key). If not established, the User Session Key will be used for key derivation. ）                          |
| Negotiate 56              | 表示支持56位密钥。 如果未协商，将使用40位密钥。 这仅适用于与“协商Lan Manager密钥”结合使用； 在NTLM1下，用户会话密钥不会减弱（因为它们已经很弱）。（Indicates support for 56-bit keys. If not negotiated, 40-bit keys will be used. This is only applicable in combination with "Negotiate Lan Manager Key"; User Session Keys are not weakened under NTLM1 (as they are already weak).） |
| Negotiate Key Exchange    | 表示将执行密钥交换以协商用于签名和密封的辅助密钥。（Indicates that key exchange will be performed to negotiate a secondary key for signing and sealing.）                                                                                                                                 |

## NTLM1Key派生
要产生或是派生NTLM1Key本质上是一个三步过程：

1. Master key negotiation
2. Key exchange
3. Key weakening

### Master key negotiation
第一步是Negotiate128位"Master Key"，从中将得出最终的签名和Sealing的Key。这是由NTLMFlags"Negotiate Lan Manager Key"驱动的；如果设置，则Lan ManagerSession Key将用作Master Key。否则，将使用适当的用户Session Key。

例如，考虑我们的示例用户，其密码为" SecREt01"。如果未设置"Negotiate Lan Manager"Key，并且在Type 3消息中提供了NTLM响应，则将选择NTLM用户Session Key作为Master Key。这是通过获取NTLM哈希的MD4摘要（本身就是Unicode密码的MD4哈希）来计算的：

0x3f373ea8e4af954f14faa506f8eebdc4


### 密钥交换（Key exchange）
如果设置了"Negotiate Key exchange"Flags，则客户端将使用新的Master Key（使用先前选择的Master Key进行RC4加密）填充Type 3消息中的"Session Key"字段。服务器将解密该值以接收新的Master Key。

例如，假定客户端选择随机Master Key" 0xf0f0aabb00112233445566778899aabb "。客户端将使用先前Negotiate的Master Key（" 0x3f373ea8e4af954f14faa506f8eebdc4 "）做为Key使用RC4加密此随机Master Key，以获取该值：

0x1d3355eb71c82850a9a2d65c2952e6f3

它在Type 3消息的"Session Key"字段中发送到服务器。服务器RC4-使用旧的Master Key对该值解密，以恢复客户端选择的新的Master Key（" 0xf0f0aabb00112233445566778899aabb "）。

### 弱化Key（Key Weakening）
最后，关键是要弱化以遵守出口限制。NTLM1支持40位和56位Key。如果设置了" Negotiate 56" NTLMFlags，则128位Master Key将减弱为56位；如果不设置，它将被削弱到40位。请注意，仅在使用Lan Manager Session Key（设置了"NegotiateLan ManagerKey"）时，才在NTLM1下采用Key弱化功能。LM和NTLM 的 User Session Key基于密码散列，而不是响应。给定的密码将始终导致NTLM1下具有相同的用户Session Key。显然不需要弱化，因为给定用户的密码哈希可以轻松恢复User Session Key。

NTLM1下的Key弱化过程如下：

* 要生成56位Key，Master Key将被截断为7个字节（56位），并附加字节值" 0xa0 "。
* 要生成40位Key，Master Key将被截断为5个字节（40位），并附加三个字节的值" 0xe538b0 "。

以Master Key" 0x0102030405060708090a0b0c0d0e0f00 "为例，用于签名和Sealing的40位Key为" 0x0102030405e538b0 "。如果Negotiate了56位Key，则最终Key将为" 0x01020304050607a0 "。

## 签名
一旦协商了Key，就可以使用它来生成数字签名，从而提供消息完整性。通过存在"Negotiate Flags" NTLMFlags来指示对签名的支持。

NTLM1签名（由SSPI MakeSignature函数完成）如下：

1. 使用先前Negotiate的Key初始化RC4密码。只需执行一次（在第一次签名操作之前），并且Key流永远不会重置。
2. 计算消息的CRC32校验和；它表示为长整数（32位Little-Endian值）。
3. 获得序列号；它从零开始，并在每条消息签名后递增。该数字表示为长号。
4. 将四个零字节与CRC32值和序列号连接起来，以获得一个12字节的值（" 0x00000000 " + CRC32（message）+ sequenceNumber）。
5. 使用先前初始化的RC4密码对该值进行加密。
6. 密文结果的前四个字节被伪随机计数器值覆盖（使用的实际值无关紧要）。
7. 将版本号（" 0x01000000 "）与上一步的结果并置以形成签名。

例如，假设我们使用上一个示例中的40位Key对消息" jCIFS "（十六进制" 0x6a43494653 "）进行签名：

1. 计算CRC32校验和（使用小端十六进制" 0xa0310宝宝7 "）。
2. 获得序列号。由于这是我们签名的第一条消息，因此序列号为零（" 0x00000000 "）。
3. 将四个零字节与CRC32值和序列号连接起来，以获得一个12字节的值（" 0x00000000a0310宝宝700000000 "）。
4. 使用我们的Key（" 0x0102030405e538b0 "）对这个值进行RC4加密；这将产生密文" 0xecbf1ced397420fe0e5a0f89 "。
5. 前四个字节被计数器值覆盖；使用" 0x78010900 "给出" 0x78010900397420fe0e5a0f89 "。
6. 将版本图章与结果连接起来以形成最终签名：
    0x0100000078010900397420fe0e5a0f89

下一条签名的消息将接收序列号1；同样，再次注意，用第一个签名初始化的RC4Key流不会为后续签名重置。

## Sealing
除了消息完整性之外，还通过Sealing来提供消息机密性。"Negotiate Sealing" NTLMFlags表示支持Sealing。在具有NTLM提供程序的SSPI下，Sealing总是与签名结合进行（Sealing消息会同时生成签名）。相同的RC4Key流用于签名和Sealing。

NTLM1Sealing（由SSPI EncryptMessage函数完成）如下：

1. 使用先前Negotiate的Key初始化RC4密码。只需执行一次（在第一次Sealing操作之前），并且Key流永远不会重置。
2. 使用RC4密码对消息进行加密；这将产生Sealing的密文。
3. 如前所述，将生成消息的签名，并将其放置在安全尾部缓冲区中。

例如，考虑使用40位Key" 0x0102030405e538b0 " 对消息" jCIFS "（" 0x6a43494653 "）进行Sealing：

1. 使用我们的Key（" 0x0102030405e538b0 "）初始化RC4密码。
2. 我们的消息通过RC4密码传递，并产生密文" 0x86fc55abca "。这是Sealing消息。
3. 我们计算出消息的CRC32校验和（使用小尾数十六进制" 0xa0310宝宝7 "）。
4. 获得序列号。由于这是第一个签名，因此序列号为零（" 0x00000000 "）。
5. 将四个零字节与CRC32值和序列号连接起来，以获得一个12字节的值（" 0x00000000a0310宝宝700000000 "）。
6. 该值是使用来自密码的Key流进行RC4加密的；这将产生密文" 0x452b490efa3e828bcc8affc3 "。
7. 前四个字节被计数器值覆盖；使用" 0x78010900 "给出" 0x78010900fa3e828bcc8affc3 "。
8. 版本标记与结果串联在一起，以形成最终签名，并将其放置在安全尾部缓冲区中：
    0x0100000078010900fa3e828bcc8affc3

    整个Sealing结构的十六进制转储为：

    0x86fc55abca0100000078010900fa3e828bcc8affc3

# NTLM2会话安全
NTLM2是更新的签名和Sealing方案，在建立"NegotiateNTLM2Key"Flags时使用。此方案中的Key派生由以下NTLMFlags驱动：

| Flags  | 说明                                                     |
|----------------------|-------------------------------------------------------------------------------|
| Negotiate NTLM2 Key  | 表示支持NTLM2会话安全性。                                                     |
| Negotiate56          | 表示支持56位Key。如果既未指定此Flags也未指定"Negotiate128"，则将使用40位Key。 |
| Negotiate128         | 表示支持128位Key。如果既未指定此Flags也未指定"Negotiate56"，则将使用40位Key。 |
| NegotiateKey交换     | 表示将执行Key交换以Negotiate用于签名和Sealing的辅助基本Key。                  |

## NTLM2Key派生
NTLM2中的Key派生分为四个步骤：

1. Master Key Negotiate
2. Key exchange
3. Key weakening
4. Subkey generation

### Master Key Negotiate
用户Session Key在NTLM2签名和Sealing中始终用作基本Master Key。使用NTLMv2身份验证时，LMv2或NTLMv2用户Session Key将用作Master Key。当NTLMv1身份验证与NTLM2会话安全一起使用时，NTLM2会话响应用户Session Key将用作Master Key。请注意，NTLM2中使用的用户Session Key比NTLM1对应的用户Session Key或Lan Manager Session Key要强得多，因为它们同时包含服务器质询和client nonce。

### Key交换
如先前针对NTLM1所讨论的那样执行Key交换。客户端选择一个辅助Master Key，RC4用基本Master Key对其进行加密，然后在Type 3"Session Key"字段中将密文值发送到服务器。这由"Negotiate Key exchange"Flags的存在指示。

### 弱化Key
NTLM2中的Key弱化仅通过将Master Key（或辅助Master Key，如果执行了Key交换）截短到适当的长度即可完成；例如，Master Key" 0xf0f0aabb00112233445566778899aabb "将减弱为40位，如" 0xf0f0aabb00 "和56位为" 0xf0f0aabb001122 "。请注意，NTLM2支持128位Key。在这种情况下，Master Key直接用于生成子Key（不执行弱化操作）。

仅当生成Sealing子Key时，Master Key才会在NTLM2下减弱。完整的128位Master Key始终用于生成签名Key。

### 子项生成
在NTLM2下，最多可以建立四个子项。Master Key实际上从未用于Signing或Sealing消息。子项生成如下：

1. 128位（无弱点）Master Key与以空值终止的ASCII常量字符串连接：
    客户端到服务器签名的Session Key魔术常数
    
    以十六进制表示，此常数是：
    
    0x73657373696f6e206b657920746f2063 
      6c69656e742d746f2d73657276657220 
      7369676e696e67206b6579206d616769 
      6320636f6e7374616e7400
    上面的换行符仅用于显示目的。将MD5消息摘要算法应用于此算法，从而得到一个16字节的值。这是客户端Signing Key，客户端使用它来为消息创建签名。

2. 原生的128位Master Key与以空值终止的ASCII常量字符串连接：
    服务器到客户端签名的Session Key魔术常数
    
    以十六进制表示，此常数是：
    
    0x73657373696f6e206b657920746f2073 
      65727665722d746f2d636c69656e7420 
      7369676e696e67206b6579206d616769 
      6320636f6e7374616e7400
    将使用此内容的MD5摘要，从而获得16字节的服务器Signing Key。服务器使用它来创建消息的签名。

3. 弱化的Master Key（取决于Negotiate的是40位，56位还是128位加密）与以空值结尾的ASCII常量字符串连接：
    客户端到服务器的Session KeySealingKey魔术常数
    
    以十六进制表示，此常数是：
    
    0x73657373696f6e206b657920746f2063 
      6c69656e742d746f2d73657276657220 
      7365616c696e67206b6579206d616769 
      6320636f6e7374616e7400
    使用MD5摘要来获取16字节的客户端Sealing Key。客户端使用它来加密消息。

4. 弱化的主键与以空值终止的ASCII常量字符串连接：
    服务器到客户端的Session KeySealingKey魔术常数
    
    以十六进制表示，此常数是：
    
    0x73657373696f6e206b657920746f2073 
      65727665722d746f2d636c69656e7420 
      7365616c696e67206b6579206d616769 
      6320636f6e7374616e7400
    应用MD5摘要算法，产生16字节的服务器Sealing Key。服务器使用此Key来加密消息。

### 签名（Signing）
签名支持再次由"Negotiate Signing" NTLMFlags指示。客户端签名是使用客户端签名Key完成的；服务器使用服务器签名Key对消息进行签名。签名Key是从无损的Master Key生成的（如前所述）。

NTLM2签名（由SSPI MakeSignature函数完成）如下：

1. 获得序列号；它从零开始，并在每条消息签名后递增。该数字表示为长整数（32位Little-endian值）。
2. 序列号与消息串联在一起。HMAC-MD5消息认证代码算法使用适当的签名Key应用于此值。这将产生一个16字节的值。
3. 如果已NegotiateKey交换，则使用适当的SealingKey初始化RC4密码。这一次完成（在第一次操作期间），并且Key流永远不会重置。HMAC结果的前八个字节使用此RC4密码加密。如果Key交换还没有经过谈判，省略这个Sealing操作。
4. 将版本号（" 0x01000000 "）与上一步的结果和序列号连接起来以形成签名。

例如，假设我们使用Master Key" 0x0102030405060708090a0b0c0d0e0f00 " 在客户端上对消息" jCIFS "（十六进制" 0x6a43494653 "）进行签名。这与客户端到服务器的签名常量连接在一起，并应用MD5生成客户端签名Key（" 0xf7f97a82ec390f9c903dac4f6aceb132 "）和客户端SealingKey（" 0x2785f595293f3e2813439d73a223810d "）；这些用于签名消息如下：

1. 获得序列号。由于这是我们签名的第一条消息，因此序列号为零（" 0x00000000 "）。
2. 序列号与消息串联在一起：
    0x000000006a43494653

    使用客户端签名Key（" 0xf7f97a82ec390f9c903dac4f6aceb132 "）应用HMAC-MD5 。结果是16字节的值" 0x0a003602317a759a720dc9c7a2a95257 "。

3. 使用我们的SealingKey（" 0x2785f595293f3e2813439d73a223810d "）初始化RC4密码。先前结果的前八个字节通过密码传递，产生密文" 0xe37f97f2544f4d7e "。
4. 将版本标记与上一步的结果和序列号连接起来，以形成最终签名：
    0x01000000e37f97f2544f4d7e00000000

### Sealing
"Negotiate Sealing" NTLMFlags再次表明支持NTLM2中的消息机密性。NTLM2Sealing（由SSPI EncryptMessage函数完成）如下：

1. RC4密码使用适当的SealingKey初始化（取决于客户端还是服务器正在执行Sealing）。只需执行一次（在第一次Sealing操作之前），并且Key流永远不会重置。
2. 使用RC4密码对消息进行加密；这将产生Sealing的密文。
3. 如前所述，将生成消息的签名，并将其放置在安全尾部缓冲区中。请注意，签名操作中使用的RC4密码已经初始化（在前面的步骤中）；它不会为签名操作重置。

例如，假设我们使用Master Key" 0x0102030405060060090090a0b0c0d0e0f00 " 在客户端上Sealing消息" jCIFS "（十六进制" 0x6a43494653 "）。与前面的示例一样，我们使用未减弱的Master Key生成客户端签名Key（" 0xf7f97a82ec390f9c903dac4f6aceb132 "）。我们还需要生成客户SealingKey；我们将假定已经Negotiate了40位弱化。我们将弱化的Master Key（" 0x0102030405 "）与客户端到服务器的Sealing常数连接起来，并应用MD5产生客户端SealingKey（" 0x6f0d9953503333cbe499cd1914fe9ee "）。以下过程用于Sealing消息：

1. RC4密码使用我们的客户SealingKey（" 0x6f0d99535033951cbe499cd1914fe9ee "）初始化。
2. 我们的消息通过RC4密码传递，并产生密文" 0xcf0eb0a939 "。这是Sealing消息。
3. 获得序列号。由于这是第一个签名，因此序列号为零（" 0x00000000 "）。
4. 序列号与消息串联在一起：
    0x000000006a43494653

    使用客户端签名Key（" 0xf7f97a82ec390f9c903dac4f6aceb132 "）应用HMAC-MD5 。结果是16字节的值" 0x0a003602317a759a720dc9c7a2a95257 "。

5. 该值的前八个字节通过Sealing密码，得到的密文为" 0x884b14809e53bfe7 "。
6. 将版本标记与结果和序列号连接起来以形成最终签名，该最终签名被放置在安全性尾部缓冲区中：
    0x01000000884b14809e53bfe700000000
    
    整个Sealing结构的十六进制转储为：
    
    0xcf0eb0a93901000000884b14809e53bfe700000000

# 会话安全主题（Miscellaneous Session Security Topics）
还有其他几个会话安全性主题，这些主题实际上并不适合其他任何地方：

* 数据报的签名和Sealing
* "虚拟"的签名

## 数据报签名和Sealing
在建立数据报上下文时使用此方法（由数据报身份验证握手和"Negotiate Datagram Style"Flags的存在指示）。关于数据报会话安全性的语义有些不同；首次调用SSPI InitializeSecurityContext函数之后（即，在与服务器进行任何通信之前），签名可以立即在客户端上开始。这意味着需要预先安排的签名和Sealing方案（因为可以在与服务器Negotiate任何选项之前创建签名）。数据报会话安全性基于具有密钥交换的40位Lan Manager Session Key NTLM1（尽管可能有一些方法可以通过注册表预先确定更强大的方案）。

在数据报模式下，序列号不递增；它固定为零，每个签名都反映了这一点。同样，每次签名或Sealing操作都会重置RC4Key流。这很重要，因为消息可能容易受到已知的明文攻击。

## "虚拟"签名
如果初始化SSPI上下文而未指定对消息完整性的支持，则使用此方法。如果建立了"始终NegotiateNegotiate" NTLMFlags，则对MakeSignature的调用将成功，并返回常量" signature"：

0x01000000000000000000000000000000

对EncryptMessage的调用通常会成功（包括安全性尾部缓冲区中的"真实"签名）。如果未Negotiate"Negotiate始终签名"，则签名和Sealing均将失败。