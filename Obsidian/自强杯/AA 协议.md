# AA 协议

## 1 电子护照的 AA 协议详解

### 1.1 关键要点

-   AA 协议即 Active Authentication 协议，用于防止电子护照芯片被克隆。
-   终端生成随机数，芯片用私钥签名，终端验证签名以确认芯片真实性。
-   使用 INTERNAL AUTHENTICATE 命令，APDU 格式为 CLA 0x00, INS 0x88, P1, P2 为 0x00 0x00。

### 1.2 什么是 AA 协议？

Active Authentication（AA）协议是电子护照安全机制的一部分，旨在确保护照芯片不是克隆的。通过证明芯片拥有与其公开密钥对应的私有密钥，AA 协议防止了未经授权的复制。

### 1.3 工作流程

1. **生成随机数**：终端生成一个随机数（通常 8 字节）作为挑战。
2. **发送命令**：终端通过 INTERNAL AUTHENTICATE 命令（APDU 格式：CLA 0x00, INS 0x88, P1, P2 为 0x00 0x00）将随机数发送给芯片。
3. **签名验证**：芯片用其私有密钥对随机数签名，并返回签名。
4. **验证真实性**：终端使用芯片的公开密钥（通过被动认证已验证）验证签名。如果签名有效，确认芯片真实。

### 1.4 令人惊讶的细节

令人惊讶的是，AA 协议是可选的，并非所有电子护照都支持这一机制，这可能导致某些护照更容易被克隆。

---

## 2 电子护照 AA 协议的详细调研报告

电子护照（e-passport）的安全机制是现代国际旅行的重要保障，其中 Active Authentication（AA）协议作为防止芯片克隆的关键技术，受到广泛关注。本报告将详细探讨 AA 协议的定义、工作流程、技术细节以及相关实施考量，旨在为用户提供全面的信息。

### 2.1 背景与定义

电子护照通过嵌入 RFID 芯片存储旅行者信息，包括生物识别数据（如面部识别、指纹），以提高安全性并防止伪造。根据国际民航组织（ICAO）的 Doc 9303 标准，AA 协议被定义为一种可选的安全机制，旨在防止护照芯片被克隆。其核心是通过证明芯片拥有与其公开密钥对应的私有密钥，确保芯片的真实性。

从搜索结果来看，AA 协议在电子护照中的作用是通过挑战-响应机制验证芯片的真实性，具体涉及终端与芯片之间的通信。相比之下，扩展访问控制（EAC）中的芯片认证虽然也有类似功能，但其实现方式更复杂，涉及密钥协商协议。

### 2.2 协议工作流程

AA 协议的详细步骤如下，基于标准智能卡通信协议 ISO/IEC 7816-4 的 INTERNAL AUTHENTICATE 命令：

1. **随机数生成**：

    - 终端生成一个随机数，通常为 8 字节，作为挑战数据。这一步确保验证过程具有随机性，难以预测。
    - 例如，终端可能生成随机数如“12345678”（实际为二进制数据）。

2. **发送 INTERNAL AUTHENTICATE 命令**：

    - 终端通过 Application Protocol Data Unit（APDU）命令将随机数发送给芯片。
    - APDU 命令格式如下：
        - CLA（类字节）：通常为 0x00，表示无安全消息传递。
        - INS（指令字节）：0x88，表示 INTERNAL AUTHENTICATE。
        - P1, P2（参数字节）：通常为 0x00 0x00，无特定参数。
        - Lc（数据长度）：表示随机数的长度，例如 8。
        - Data（数据字段）：随机数本身。
    - 示例 APDU 命令：`00 88 00 00 08 12345678`。

3. **芯片签名**：

    - 芯片接收随机数后，使用其存储的私有密钥对该随机数进行签名。
    - 签名算法通常为 RSA 或 ECDSA，具体取决于护照的实现。
    - 签名生成后，芯片通过响应 APDU 返回签名数据。

4. **终端验证**：
    - 终端接收签名后，使用芯片的公开密钥进行验证。
    - 芯片的公开密钥需通过被动认证（Passive Authentication）预先获取并验证其真实性。
    - 验证过程涉及检查签名的有效性，若有效，则确认芯片非克隆。

### 2.3 技术细节与实施考量

-   **签名算法**：AA 协议支持的签名算法包括 RSA 和 ECDSA，具体选择由发行国决定。RSA 签名通常涉及较大的密钥长度（如 2048 位），而 ECDSA 则更高效但对硬件要求较高。
-   **APDU 响应**：芯片返回的响应 APDU 包括签名数据和状态字节（如 90 00 表示成功）。
-   **安全通道**：AA 协议通常在被动认证之后执行，可能在建立安全通道之前或之中，具体取决于护照的实现。
-   **可选性**：根据 ICAO 标准，AA 协议为可选机制，并非所有电子护照都支持。例如，某些较旧的护照可能缺乏 AA 支持，增加克隆风险。

### 2.4 相关安全机制对比

为更好地理解 AA 协议，可将其与其他电子护照安全机制对比：

| 机制名称            | 目的                   | 实现方式                     | 是否强制 |
| ------------------- | ---------------------- | ---------------------------- | -------- |
| 基本访问控制（BAC） | 防止数据撷取           | 使用 MRZ 数据加密通信        | 是       |
| 扩展访问控制（EAC） | 保护敏感生物识别数据   | 芯片认证和终端认证，密钥协商 | 否       |
| 主动认证（AA）      | 防止芯片克隆           | 芯片签名随机数，终端验证     | 否       |
| 被动认证（PA）      | 验证数据完整性和真实性 | 检查数字签名                 | 是       |

从表中可见，AA 协议专注于克隆检测，与 EAC 的芯片认证有相似之处，但实现更简单，主要依赖签名验证。

### 2.5 实施中的挑战与漏洞

搜索结果显示，AA 协议并非完美无缺。例如，2008 年 Jeroen van Beek 的研究表明，通过修改护照索引文件（EF.COM），攻击者可禁用 AA 机制，导致克隆风险增加（[Biometric passport - Wikipedia](https://en.wikipedia.org/wiki/Biometric_passport)）。2014 年，Calderoni 等人的研究进一步揭示了第一代电子护照中 AA 协议的潜在漏洞，涉及元数据文件（EF.COM, EF.SOD）的隐藏。

此外，AA 协议的随机数长度有限（如 8 字节）可能导致签名输入数据受限，影响安全性（[Using ePassport with Active/Chip Authentication to sign documents - Cryptography Stack Exchange](https://crypto.stackexchange.com/questions/75058/using-epassport-with-active-chip-authentication-to-sign-documents)）。

### 2.6 实际应用与支持

根据 Inverid 的博客（[Cloning detection for ePassports](https://www.inverid.com/blog/cloning-detection-epassports)），AA 协议由 ReadID 等工具支持，用于验证护照真实性。然而，实际支持情况因国而异，部分国家可能优先采用 EAC 中的芯片认证，因其更常用于当前护照。

### 2.7 结论

AA 协议是电子护照安全机制的重要组成部分，通过挑战-响应机制防止芯片克隆。其实现依赖 INTERNAL AUTHENTICATE 命令，APDU 格式为 CLA 0x00, INS 0x88，终端验证签名以确认芯片真实性。尽管可选性使其应用不普遍，但其在克隆检测中的作用不容忽视。用户应注意，部分护照可能缺乏 AA 支持，增加安全风险。

### 2.8 关键引文

-   [Biometric passport - Wikipedia](https://en.wikipedia.org/wiki/Biometric_passport)
-   [Cloning detection for ePassports](https://www.inverid.com/blog/cloning-detection-epassports)
-   [Using ePassport with Active/Chip Authentication to sign documents - Cryptography Stack Exchange](https://crypto.stackexchange.com/questions/75058/using-epassport-with-active-chip-authentication-to-sign-documents)

## 3 PA

### 3.1 直接回答

**关键点：**
- 被动认证（PA）是电子护照（ePassport）用于验证数据完整性和真实性的安全机制。
- 它通过数字签名和公钥基础设施（PKI）确保数据未被篡改且由发行机构签发。
- 令人惊讶的是，不同国家可能使用不同的加密算法，实施方式差异较大。

**什么是被动认证（PA）？**
被动认证（PA）是一种安全协议，用于验证电子护照芯片上数据的真实性和完整性。它确保数据未被篡改，并且是由发行国家签发的。PA不保护数据免受窃听，也不保证当前持有者拥有原始护照。

**工作原理**
PA通过以下步骤工作：
1. 从护照芯片读取数据组（DGs）和文档安全对象（SOD）。
2. 使用国家签名证书（CSCA）的公钥验证文档签名者证书（DSC）。
3. 使用DSC的公钥验证SOD的签名。
4. 计算数据组的哈希值，与SOD中的哈希值比较，确保数据未被更改。
如果所有步骤成功，数据被认为是真实的。

**参数和流程**
- **参数：** 需要的数据包括数据组、SOD、DSC和CSCA的公钥。
- **流程：** 包括读取芯片数据、验证证书链、检查签名和比较哈希值。

**实施细节**
PA允许使用多种加密算法（如RSA、ECDSA），具体取决于发行国。检查系统需要维护包含CSCA证书的信任存储库。

---

### 3.2 详细报告

被动认证（PA）是电子护照（ePassport）系统中的核心安全机制，旨在验证芯片上存储数据的完整性和真实性，确保数据未被篡改且由适当的发行机构签发。本报告详细概述了PA协议的参数声明、协议流程和实施细节，基于国际民航组织（ICAO）Doc 9303标准以及相关研究和实践。

#### 3.2.1 背景与概述

电子护照（ePassport）是基于ICAO Doc 9303标准的机器可读旅行证件，包含NFC芯片以存储个人数据，如面部图像、指纹和机器可读区（MRZ）信息。为了保护这些数据的完整性和真实性，PA被设计为一种“被动”机制，这意味着验证过程不需要与芯片交互，仅依赖于静态数据验证。

PA的主要目标是确保数据是由发行国签发的，并且在发行后未被更改。它通过数字签名和公钥基础设施（PKI）实现，但不提供对窃听的保护，也不确保当前持有者拥有原始护照（这些功能由隐私和克隆检测机制如基本访问控制（BAC）或芯片认证（CA）处理）。

令人惊讶的是，PA的实施在不同国家之间差异很大，ICAO标准允许使用多种加密算法，这导致实际应用中的多样性。

#### 3.2.2 参数声明

PA协议涉及以下关键参数，这些参数是验证过程中所需的输入和输出：

- **数据组（Data Groups, DGs）：**
  电子护照芯片上存储的各种数据元素，例如DG1（MRZ数据）、DG2（面部图像）、DG3（指纹）等。这些数据是验证完整性的基础。

- **文档安全对象（Document Security Object, SOD）：**
  SOD是一个特殊的数据结构，包含所有相关数据组的哈希值，并由发行机构使用其私钥进行数字签名。SOD是PA验证的核心。

- **文档签名者证书（Document Signer's Certificate, DSC）：**
  DSC包含用于验证SOD签名的公钥，并由国家签名证书机构（CSCA）签发。它通常存储在芯片的DG15中。

- **国家签名证书机构（Country Signing Certificate Authority, CSCA）：**
  CSCA是信任链的根，负责签发DSC，其公钥由检查系统用于验证DSC的真实性。CSCA的公钥通常通过ICAO公共密钥目录（PKD）分发。

验证过程的输出是布尔值，表示数据是否通过了完整性和真实性检查。

#### 3.2.3 协议流程

PA协议的步骤如下，详细描述了验证过程：

1. **读取电子护照芯片：**
   - 使用NFC或接触式接口从ePassport芯片读取所有相关数据组（DGs）和文档安全对象（SOD）。
   - 这通常需要先通过基本访问控制（BAC）或密码认证连接建立（PACE）等机制访问芯片。

2. **验证文档签名者证书（DSC）：**
   - 从芯片（通常在DG15）中提取DSC。
   - 使用预先加载的CSCA公钥验证DSC的签名，确保DSC是由可信的发行国签发的。
   - 这建立了信任链的第一个环节。

3. **验证SOD的签名：**
   - 使用从DSC中提取的公钥验证SOD的数字签名。
   - 如果签名验证成功，说明SOD未被篡改，并且是由发行机构签发的。

4. **检查数据完整性：**
   - 根据SOD中列出的数据组，计算每个数据组的哈希值（使用SOD指定的哈希算法，如SHA-1或SHA-256）。
   - 将计算得到的哈希值与SOD中存储的哈希值进行比较。
   - 如果所有哈希值匹配，说明数据组未被更改。

5. **结论：**
   - 如果上述所有步骤都成功，PA验证通过，数据被认为是真实的和完整的。
   - 否则，验证失败，说明可能存在篡改或证书无效。

#### 3.2.4 实施细节

PA的实施涉及多个技术方面，具体取决于ICAO Doc 9303的规范和各国的选择：

- **加密算法：**
  ICAO标准允许使用多种加密算法，包括但不限于：
  - 签名算法：RSA、ECDSA。
  - 哈希算法：SHA-1、SHA-256等。
  不同国家可能选择不同的算法组合，这导致PA的实施在全球范围内存在显著差异。

- **密钥管理和信任存储：**
  - CSCA负责生成和分发其公钥，通常通过ICAO PKD分发给检查系统。
  - 检查系统需要维护一个信任存储库，包含所有相关CSCA证书，以验证DSC。
  - DSC通常为特定时期或特定实体（如护照供应商）签发，以限制风险。

- **数据组哈希和SOD结构：**
  - SOD按照ICAO标准定义的逻辑数据结构（LDS）格式存储，包含数据组的哈希值列表。
  - 哈希计算需要严格遵循SOD中指定的算法和数据组顺序。

- **实际实施中的挑战：**
  - 由于不同国家的算法和证书选择，检查系统需要支持多种加密方案。
  - 一些较旧的ePassport可能缺乏克隆检测机制（如主动认证），仅依赖PA，这可能增加安全风险。
  - 实施中需要确保检查系统能够处理CSCA证书的更新和撤销。

以下表格总结了PA协议的关键参数和步骤：

| **参数/步骤**      | **描述**                        |
| -------------- | ----------------------------- |
| 数据组（DGs）       | 芯片上存储的个人数据，如MRZ、面部图像等，用于哈希计算。 |
| 文档安全对象（SOD）    | 包含数据组哈希值的签名对象，由DSC的私钥签名。      |
| 文档签名者证书（DSC）   | 包含公钥的证书，由CSCA签发，用于验证SOD签名。    |
| 国家签名证书机构（CSCA） | 信任链的根，其公钥用于验证DSC。             |
| 步骤1：读取芯片       | 从ePassport芯片读取DGs和SOD。        |
| 步骤2：验证DSC      | 使用CSCA公钥验证DSC的签名。             |
| 步骤3：验证SOD签名    | 使用DSC公钥验证SOD的签名。              |
| 步骤4：检查数据完整性    | 计算DGs的哈希值，与SOD中的哈希值比较。        |
| 步骤5：结论         | 如果所有步骤成功，数据被认为是真实的和完整的。       |
|                |                               |

#### 3.2.5 研究和来源

本报告基于多个来源，包括Inverid博客的详细解释（[Authenticity of ePassports](https://www.inverid.com/blog/authenticity-electronic-passports)、[Overview security mechanisms in ePassports](https://www.inverid.com/blog/overview-security-mechanisms-epassports)），以及ICAO官方网站的系统要求页面（[System Requirements](https://www.icao.int/Security/FAL/PKD/BVRT/Pages/System-Requirements.aspx)）。此外，Stack Overflow上的讨论（[ePassport Passive Authentication in Android using jmrtd](https://stackoverflow.com/questions/59578678/epassport-passive-authentication-in-androidjava-using-jmrtd)）提供了实际实施的见解。

#### 3.2.6 结论

被动认证（PA）是电子护照安全机制的重要组成部分，通过验证数字签名和哈希值确保数据的完整性和真实性。尽管其实现灵活且依赖于PKI，但不同国家的算法选择和实施差异可能影响全球互操作性。检查系统需要维护信任存储库并支持多种加密算法，以确保有效验证。

**关键引用：**
- [Authenticity of ePassports detailed explanation](https://www.inverid.com/blog/authenticity-electronic-passports)
- [Overview of security mechanisms in ePassports](https://www.inverid.com/blog/overview-security-mechanisms-epassports)
- [ICAO System Requirements for ePassport Validation](https://www.icao.int/Security/FAL/PKD/BVRT/Pages/System-Requirements.aspx)
- [ePassport Passive Authentication implementation in Android using jmrtd](https://stackoverflow.com/questions/59578678/epassport-passive-authentication-in-androidjava-using-jmrtd)

## 4 AA

### 4.1 关键点：

- 主动认证（AA）是电子护照的防克隆机制，通过验证芯片的真实性确保其未被复制。
- 流程包括读取芯片数据、发送随机挑战、芯片签名验证。
- 不同国家可能使用不同的加密算法，如RSA或ECDSA。

#### 4.1.1 什么是主动认证（AA）？

主动认证（AA）是一种可选的安全机制，用于检测电子护照芯片是否被克隆或替换。它通过让芯片对读卡器发送的随机挑战进行签名，并使用芯片的公钥验证签名来确保芯片的真实性。令人惊讶的是，并非所有国家都强制实施AA，较旧的护照可能完全缺乏克隆检测机制。

#### 4.1.2 工作原理

AA的工作流程如下：

1. 读卡器读取芯片上的文档安全对象（SOD）并验证其签名。
2. 读卡器从SOD或相关数据组中提取芯片的公钥。
3. 读卡器生成一个随机挑战并发送给芯片。
4. 芯片使用其私钥对挑战进行签名。
5. 读卡器使用芯片的公钥验证签名，如果成功则确认芯片为真。

#### 4.1.3 参数和流程

- **参数：** 包括随机挑战、芯片的私钥（存储在芯片上）、芯片的公钥（通常在SOD中）。
- **流程：** 涉及读取SOD、验证签名、发送挑战、芯片签名和验证。

#### 4.1.4 实施细节

AA支持多种加密算法（如RSA或ECDSA），具体取决于发行国。芯片需要支持特定的命令（如INTERNAL AUTHENTICATE）来执行签名操作。

---

### 4.2 详细报告

主动认证（AA）是电子护照（ePassport）系统中的一种可选安全机制，旨在防止芯片被克隆或替换，确保芯片的真实性和完整性。与被动认证（PA）不同，PA主要验证数据未被篡改，而AA专注于检测芯片是否为原始芯片。本报告详细概述了AA协议的概述、参数声明、协议流程和实施细节，基于国际民航组织（ICAO）Doc 9303标准以及相关研究和实践。

#### 4.2.1 背景与概述

电子护照（ePassport）是基于ICAO Doc 9303标准的机器可读旅行证件，包含NFC芯片以存储个人数据，如面部图像、指纹和机器可读区（MRZ）信息。为了防止芯片被克隆或替换，AA被设计为一种“主动”机制，这意味着它需要芯片与读卡器进行交互，通过挑战-响应协议验证芯片的真实性。AA不是强制性的，不同国家在是否实施AA以及使用的加密算法上存在显著差异。令人惊讶的是，较旧的ePassport可能完全缺乏克隆检测机制，这增加了安全风险。

#### 4.2.2 参数声明

AA协议涉及以下关键参数，这些参数是验证过程中所需的输入和输出：

- **随机挑战（Challenge）：**
  由读卡器生成的一个随机数，用于发送给芯片以进行签名验证。

- **芯片的私钥（Chip's Private Key）：**
  存储在ePassport芯片上的私钥，用于对挑战进行签名。这是芯片的唯一标识，确保其不可克隆。

- **芯片的公钥（Chip's Public Key）：**
  对应于芯片私钥的公钥，通常存储在文档安全对象（SOD）中，或在某些实现中存储在其他数据组中（如DG14或DG15）。该公钥由发行机构签名，确保其可信。

- **文档安全对象（SOD，Document Security Object）：**
  SOD是一个数据结构，包含所有相关数据组的哈希值，并可能包括芯片的公钥，用于AA。它由发行机构的私钥签名。

- **文档签名者证书（DSC，Document Signer's Certificate）：**
  DSC包含发行机构的公钥，用于验证SOD的签名。它由国家签名证书机构（CSCA）签发，通常存储在DG15中。

验证过程的输出是布尔值，表示芯片是否通过了真实性检查。

#### 4.2.3 协议流程

AA协议的步骤如下，详细描述了验证过程：

1. **读取并验证SOD：**
   - 读卡器通过NFC或接触式接口从ePassport芯片读取SOD。
   - 使用从DSC中提取的公钥验证SOD的签名，确保SOD未被篡改且由发行机构签发。这一步类似于PA的验证过程。

2. **提取芯片的公钥：**
   - 从SOD或相关数据组中提取芯片的公钥。需要注意的是，在某些实现中，芯片的公钥可能存储在SOD中，而在其他实现中可能在单独的数据组（如DG14）中。

3. **生成并发送挑战：**
   - 读卡器生成一个随机挑战（通常为8字节的随机数），并通过安全通道发送给芯片。

4. **芯片签名挑战：**
   - 芯片使用其私钥对接收到的挑战进行签名。签名过程通常使用RSA或ECDSA算法，具体取决于芯片的配置。
   - 芯片通过支持的命令（如INTERNAL AUTHENTICATE）返回签名的结果。

5. **验证签名：**
   - 读卡器使用从SOD中提取的芯片公钥验证签名的有效性。
   - 如果签名验证成功，说明芯片的私钥与公钥匹配，芯片被认为是真实的和原始的。

6. **结论：**
   - 如果所有步骤成功，AA验证通过，芯片被确认未被克隆或替换。
   - 否则，验证失败，可能表示芯片已被篡改或为克隆芯片。

#### 4.2.4 实施细节

AA的实施涉及多个技术方面，具体取决于ICAO Doc 9303的规范和各国的选择：

- **加密算法：**
  ICAO标准允许使用多种加密算法，包括但不限于：
  - 签名算法：RSA（通常使用ISO/IEC 9796-2方案1）或ECDSA。
  - 哈希算法：与签名算法相关的哈希函数，如SHA-1或SHA-256。
  不同国家可能选择不同的算法组合，这导致AA的实施在全球范围内存在显著差异。

- **密钥管理和信任存储：**
  - 芯片的私钥在制造过程中被写入芯片，并通过硬件保护机制确保其安全性和不可提取性。
  - 芯片的公钥由发行机构签名并存储在SOD中，读卡器需要通过CSCA的公钥验证DSC，然后通过DSC验证SOD，确保公钥的可信性。
  - 读卡器需要维护一个信任存储库，包含所有相关CSCA证书，以建立信任链。

- **命令和接口：**
  - AA通常通过ISO/IEC 7816-4定义的INTERNAL AUTHENTICATE命令实现。
  - 挑战长度通常为8字节，这限制了输入数据的长度，可能影响某些高级应用。

- **实际实施中的挑战：**
  - 由于不同国家的算法和证书选择，读卡器需要支持多种加密方案。
  - 一些较旧的ePassport可能不支持AA，这可能导致克隆检测失败。
  - 实施中需要确保读卡器能够处理CSCA证书的更新和撤销。

以下表格总结了AA协议的关键参数和步骤：

| **参数/步骤**               | **描述**                                                                 |
|-------------------------------|--------------------------------------------------------------------------|
| 随机挑战（Challenge）        | 由读卡器生成的随机数，用于发送给芯片以进行签名验证。                     |
| 芯片的私钥（Chip's Private Key） | 存储在芯片上的私钥，用于对挑战进行签名，确保芯片唯一性。                 |
| 芯片的公钥（Chip's Public Key） | 存储在SOD或相关数据组中，由发行机构签名，确保可信。                     |
| 文档安全对象（SOD）          | 包含数据组哈希值和可能包括芯片公钥，签名由发行机构私钥签名。             |
| 文档签名者证书（DSC）        | 包含发行机构公钥的证书，由CSCA签发，用于验证SOD签名。                    |
| 步骤1：读取并验证SOD         | 读卡器读取SOD并使用DSC公钥验证其签名。                                   |
| 步骤2：提取芯片公钥          | 从SOD或相关数据组中提取芯片的公钥。                                      |
| 步骤3：生成并发送挑战        | 读卡器生成随机挑战并发送给芯片。                                         |
| 步骤4：芯片签名挑战          | 芯片使用其私钥对挑战进行签名并返回。                                     |
| 步骤5：验证签名              | 读卡器使用芯片公钥验证签名的有效性。                                    |
| 步骤6：结论                  | 如果签名有效，芯片被确认为真；否则失败。                                 |

#### 4.2.5 研究和来源

本报告基于多个来源，包括Inverid博客的详细解释（[Cloning Detection for ePassports](https://www.inverid.com/blog/cloning-detection-epassports)、[Overview Security Mechanisms in ePassports](https://www.inverid.com/blog/overview-security-mechanisms-epassports)），以及ICAO官方网站的系统要求页面（[System Requirements](https://www.icao.int/Security/FAL/PKD/BVRT/Pages/System-Requirements.aspx)）。此外，Cryptography Stack Exchange上的讨论（[Using ePassport with Active/Chip Authentication to Sign Documents](https://crypto.stackexchange.com/questions/75058/using-epassport-with-active-chip-authentication-to-sign-documents)）提供了实际实施的见解。

#### 4.2.6 结论

主动认证（AA）是电子护照安全机制的重要组成部分，通过挑战-响应协议确保芯片的真实性和防克隆能力。尽管其实现灵活且依赖于PKI，但不同国家的算法选择和实施差异可能影响全球互操作性。读卡器需要维护信任存储库并支持多种加密算法，以确保有效验证。

**关键引用：**
- [Cloning Detection for ePassports Detailed Explanation](https://www.inverid.com/blog/cloning-detection-epassports)
- [Overview of Security Mechanisms in ePassports](https://www.inverid.com/blog/overview-security-mechanisms-epassports)
- [ICAO System Requirements for ePassport Validation](https://www.icao.int/Security/FAL/PKD/BVRT/Pages/System-Requirements.aspx)
- [Using ePassport with Active/Chip Authentication to Sign Documents Discussion](https://crypto.stackexchange.com/questions/75058/using-epassport-with-active-chip-authentication-to-sign-documents)

## 5 PA（Passive Authentication）

**PA** 是电子护照的数据完整性验证机制，用于确认芯片内存储的数据是否被篡改。通过验证安全对象目录（EF.SOD）中的数字签名和数据组哈希值，确保数据的真实性和完整性。

### 5.1 参数声明

- **EF.SOD**：安全对象目录文件，包含数据组的哈希值和数字签名。
- **哈希算法**：用于计算数据组哈希值的算法（如 SHA-256）。
- **DG 文件**：数据组文件（如 DG1、DG2）存储实际的护照数据。

### 5.2 协议流程

1. **读取 EF.SOD 文件**：
    - 使用会话密钥读取 EF.SOD 文件，获取各个数据组的哈希值和数字签名。
2. **计算数据组哈希值**：
    - 对读取到的 DG 文件重新计算哈希值。
3. **比对哈希值**：
    - 将计算得到的哈希值与 EF.SOD 中存储的哈希值进行比较，验证数据是否被篡改。
4. **验证数字签名**：
    - 使用国家签发的公钥证书验证 EF.SOD 中的数字签名，确认签名的有效性。

### 5.3 实现细节

#### 5.3.1 读取 EF.SOD 文件

```python
def select_SOD(connection, BAC_data):     
    # 使用会话密钥加密命令，读取 EF.SOD 文件     
    # 解密响应数据，获取 EF_SOD
```

#### 5.3.2 被动认证过程

```python
def PassiveAuth(EF_SOD):
    # 解析 EF_SOD，提取哈希值和数字签名
    # 对比计算的哈希值和存储的哈希值
    # 使用公钥验证数字签名
```

#### 5.3.3 验证结果

```python
flag = PassiveAuth(EF_SOD)
if flag == True:
    print("PA 认证通过！")
else:
    print("PA 认证失败！")
```

## 6 AA（Active Authentication，主动认证）

**AA** 是电子护照的防克隆机制，用于验证芯片的唯一性。通过挑战-响应协议，读取器验证芯片是否持有正确的私钥，防止芯片被复制。Active Authentication（AA）协议是电子护照安全机制的一部分，旨在确保护照芯片不是克隆的。通过证明芯片拥有与其公开密钥对应的私有密钥，AA 协议防止了未经授权的复制。

### 6.1 参数声明

- **RND<sub>IFD</sub>**：读取器生成的挑战随机数。
- **签名数据 S**：芯片对 RND<sub>IFD</sub> 的签名。
- **EF.DG15**：存储芯片的公钥，供验证签名使用。

### 6.2 协议流程

1. **生成挑战随机数**：
    - 读取器生成 8 字节的随机数 RND<sub>IFD</sub>。
2. **发送 INTERNAL AUTHENTICATE 命令**：
    - 将 RND<sub>IFD</sub> 发送给芯片，请求签名。
3. **芯片返回签名数据**：
    - 芯片使用私钥对 RND<sub>IFD</sub> 进行签名，返回签名数据 S。
4. **读取 EF.DG15 公钥**：
    - 读取 EF.DG15，获取芯片的公钥。
5. **验证签名**：
    - 使用公钥验证签名数据 S，确认芯片持有正确的私钥。

### 6.3 实现细节

生成挑战随机数并发送认证请求

```python
def internal_authenticate(connection, RND_IFD):
    # 发送 INTERNAL AUTHENTICATE 命令，传入 RND_IFD
    # 返回芯片的签名数据
```

读取 EF.DG15 获取公钥

```python
EF_DG15, BAC_data = select_DG15(connection, BAC_data)
Kpu = EF_DG15.hex().upper()
```

验证签名

```python
def Verify_AA_RSA_M(KPu, S):
    # 使用芯片的公钥 KPu，验证签名数据 S
    # 解密签名，验证包含的随机数是否与 RND_IFD 匹配
```

认证结果

```python
if do_AA(connection, BAC_data) == True:
    print("AA 认证通过！")
else:
    print("AA 认证失败！")
```