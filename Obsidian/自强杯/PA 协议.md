# PA 协议

被动认证（PA）是电子护照（ePassport）系统中的核心安全机制，旨在验证芯片上存储数据的完整性和真实性，确保数据未被篡改且由适当的发行机构签发。本报告详细概述了PA协议的参数声明、协议流程和实施细节，基于国际民航组织（ICAO）Doc 9303标准以及相关研究和实践。为了保护电子护照中数据的完整性和真实性，PA被设计为一种“被动”机制，这意味着验证过程不需要与芯片交互，仅依赖于静态数据验证。

**PA** 是电子护照的数据完整性验证机制，用于确认芯片内存储的数据是否被篡改。通过验证安全对象目录（EF.SOD）中的数字签名和数据组哈希值，确保数据的真实性和完整性。

PA的主要目标是确保数据是由发行国签发的，并且在发行后未被更改。它通过数字签名和公钥基础设施（PKI）实现，但不提供对窃听的保护，也不确保当前持有者拥有原始护照。

## 1 参数声明

PA协议涉及以下关键参数，这些参数是验证过程中所需的输入和输出：

- **数据组（Data Groups, DGs）：**
  电子护照芯片上存储的各种数据元素，例如DG1（MRZ数据）、DG2（面部图像）、DG3（指纹）等。这些数据是验证完整性的基础。

- **文档安全对象（Document Security Object, SOD）：**
  SOD是一个特殊的数据结构，包含所有相关数据组的哈希值，并由发行机构使用其私钥进行数字签名。SOD是PA验证的核心。

- **EF.SOD**：安全对象目录文件，包含数据组的哈希值和数字签名。

- **文档签名者证书（Document Signer's Certificate, DSC）：**
  DSC包含用于验证SOD签名的公钥，并由国家签名证书机构（CSCA）签发。它通常存储在芯片的DG15中。

- **国家签名证书机构（Country Signing Certificate Authority, CSCA）：**
  CSCA是信任链的根，负责签发DSC，其公钥由检查系统用于验证DSC的真实性。CSCA的公钥通常通过ICAO公共密钥目录（PKD）分发。

验证过程的输出是布尔值，表示数据是否通过了完整性和真实性检查。

## 2 协议流程

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

## 3 实现细节

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

### 3.1 读取 EF.SOD 文件

```python
def select_SOD(connection, BAC_data):     
    # 使用会话密钥加密命令，读取 EF.SOD 文件     
    # 解密响应数据，获取 EF_SOD
```

### 3.2 被动认证过程

```python
def PassiveAuth(EF_SOD):
    # 解析 EF_SOD，提取哈希值和数字签名
    # 对比计算的哈希值和存储的哈希值
    # 使用公钥验证数字签名
```

### 3.3 验证结果

```python
flag = PassiveAuth(EF_SOD)
if flag == True:
    print("PA 认证通过！")
else:
    print("PA 认证失败！")
```

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

被动认证（PA）是电子护照安全机制的重要组成部分，通过验证数字签名和哈希值确保数据的完整性和真实性。尽管其实现灵活且依赖于PKI，但不同国家的算法选择和实施差异可能影响全球互操作性。检查系统需要维护信任存储库并支持多种加密算法，以确保有效验证。