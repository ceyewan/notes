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
