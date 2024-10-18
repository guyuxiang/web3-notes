## openzeppelin utils/cryptography模块



#### ECDSA合约库

ECDSA库是一个验证地址真实身份的工具库，其作用是在链上验证某签名信息是否由地址的私钥持有者进行签名的



recover函数实现了三种入参的重载, 可传入待签名数据的hash和签名的不同表示形式, 会还原出签名者的地址,如果产生错误会触发revert



```solidity
function recover(bytes32 hash, bytes memory signature) internal pure returns (address)
function recover(bytes32 hash, bytes32 r, bytes32 vs) internal pure returns (address)
function recover(bytes32 hash, uint8 v, bytes32 r, bytes32 s) internal pure returns (address)
```

signature为r,s,v的合并

vs为ERC-2098：紧凑签名表示 https://eips.ethereum.org/EIPS/eip-2098

签名数据格式为

```
 "\x19Ethereum Signed Message:\n" + 原始签名内容字节长度 + 原始签名内容
```



#### EIP712合约库

EIP-712是一个用于对结构化数据求hash值以及签名的标准,解决数据“链下签名+链上验证”的问题,签名数据的结构化可以解决数据的可用性和安全性, 本库实现的整体编码规则为v4版本，匹配Metamask中的JSON RPC方法 ——`eth_signTypedDataV4`

![image-20240423154112038](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20240423154112038.png)

EIP-712结构化的签名消息可以显示, 另外字段链id可防止由链硬分叉而引起的重放攻击。

![image-20240423154148104](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20240423154148104.png)

该合约提供了EIP-712的编码定义和解析



域分隔符（domain separator）:主要防止一个 DApp 的签名还能在另一个 DApp 中工作，从而导致签名冲突。

```javascript
{
    name: "Auction dApp", // DApp 的名字
    version: "2", // DApp 的版本
    chainId: "1", // [EIP-155] 定义的 chainId
    verifyingContract: "0x1c56346...", // 验签合约地址
    salt: "0x43efba6b4..." // 硬编码到合约和 DApp 中的一个随机数值
}
```



另外hashStruct为要签名的具体信息, 最后对域分隔符和hashStruct进行拼接再进行hash就可计算签名



合约构造函数初始化域分隔符, 传入应用名和应用版本, constructor中将参数初始化到immutable常量，后续无法修改，除非依靠合约升级

```solidity
constructor(string memory name, string memory version)
```



获取当前的domain separator

```solidity
function _domainSeparatorV4() internal view returns (bytes32)
```



返回基于本domain的结构化数据的完整EIP712的编码信息。该编码信息往往用作对结构化数据签名的摘要。内部使用ECDSA库中的toTypedDataHash方法，传入本domain separator以及结构化数据的hash计算整个摘要并返回

```solidity
function _hashTypedDataV4(bytes32 structHash) internal view virtual returns (bytes32)
```



#### SignatureChecker合约库

该合约库用于链上签名验证, 该库提供的验签函数既支持EOA账户地址的签名验证也支持IERC1271标准合约的签名验证。

该函数内部会尝试去使用ECDSA.tryRecover验证EOA账户的签名和调用符合IERC1271的验签合约去验证,只要其中一个验证通过即返回true

```solidity
function isValidSignatureNow(
        address signer,
        bytes32 hash,
        bytes memory signature
    ) internal view returns (bool)
```



#### **IERC1271**

IERC1271是一个提供自定义验签名过程的合约对外的接口标准

签名的验证过程可以自定义实现, 如果验签通过返回0x1626ba7e，即bytes4(keccak256("isValidSignature(bytes32,bytes)")

```solidity
interface IERC1271 {
    function isValidSignature(bytes32 hash, bytes memory signature) external view returns (bytes4 magicValue);
}
```

ERC-1271最常见的用途:
1.当使用智能合约钱包时，这些服务严重依赖ERC-1271来提供完整的签名功能
2.零Gas交易中验证签名
3.去中心化交易所（DEX）上的限价挂单
4.多签钱包


有验签需求的合约A需要通过外部调用IERC1271实现合约B的`isValidSignature`方法且该方法的具体实现是合约A不可控的。出于安全考虑，建议在合约A中硬编码对`isValidSignature`调用的gas limit



#### MerkleProof合约库

提供了用于验证merkle树proof的工具函数,  减少链上的存储,  通常用来实现白名单空投校验,链上存储白名单的merkle树根, 验证一个用户地址是否属于白名单列表

processProof函数传入排序好的证明hash数组和需要证明的叶子节点leaf, 来迭代计算merkle tree的root

```solidity
function processProof(bytes32[] memory proof, bytes32 leaf) internal pure returns (bytes32)
```

verify函数相比processProof函数多传入一个root值, 用来比较和计算出的root值是否相对,返回bool类型

```solidity
function verify(bytes32[] memory proof, bytes32 root, bytes32 leaf) internal pure returns (bool)
```



merkle树和对应proof证明可利用Openzeppelin提供的js库生成，该js库与MerkleProof库配合使用是安全的：https://github.com/OpenZeppelin/merkle-tree



#### MessageHashUtils合约库

Ethereum特定的签名msg结构为：

```
"\x19Ethereum Signed Message:\n" + 原始签名内容字节长度 + 原始签名内容
```

该库可以将传入的hash值转换为Ethereum特定的待签名消息hash, 通过在原始msg中添加前缀使后续计算出的签名可以被识别为Ethereum特定的签名，这样做是防止滥用。

```solidity
function toEthSignedMessageHash(bytes32 messageHash) internal pure returns (bytes32 digest)
```

```solidity
function toEthSignedMessageHash(bytes memory message) internal pure returns (bytes32)
```



通过传入的domainSeparator和structHash计算EIP-712的特定签名消息hash

```solidity
function toTypedDataHash(bytes32 domainSeparator, bytes32 structHash) internal pure returns (bytes32 digest)
```

