## openzeppelin proxy模块

proxy库提供了多种合约升级方式

├── Proxy.sol
├── Clones.sol
├── ERC1967
│   ├── ERC1967Proxy.sol
│   └── ERC1967Upgrade.sol
├── beacon
│   ├── BeaconProxy.sol
│   ├── IBeacon.sol
│   └── UpgradeableBeacon.sol
├── transparent
│   ├── ProxyAdmin.sol
│   └── TransparentUpgradeableProxy.sol
└── utils
    ├── Initializable.sol
    └── UUPSUpgradeable.sol



#### Proxy.sol

是基础的代理合约，继承该合约后可以构造函数中去存储逻辑合约地址，合约通过内联汇编实现了delegate，让没有返回值的回调函数fallback也可以返回数据

```solidity
function _implementation() internal view virtual returns (address);
```

```solidity
fallback() external payable virtual
```

```solidity
function _delegate(address implementation) internal virtual {
    assembly {
        // 将msg.data拷贝到内存里
        // calldatacopy操作码的参数: 内存起始位置，calldata起始位置，calldata长度
        calldatacopy(0, 0, calldatasize())

        // 利用delegatecall调用implementation合约
        // delegatecall操作码的参数：gas, 目标合约地址，input mem起始位置，input mem长度，output area mem            起始位置，output area mem长度
        // output area起始位置和长度位置，所以设为0
        // delegatecall成功返回1，失败返回0
        let result := delegatecall(gas(), implementation, 0, calldatasize(), 0, 0)

        // 将return data拷贝到内存
        // returndata操作码的参数：内存起始位置，returndata起始位置，returndata长度
        returndatacopy(0, 0, returndatasize())

        switch result
        // 如果delegate call失败，revert
        case 0 {
            revert(0, returndatasize())
        }
        // 如果delegate call成功，返回mem起始位置为0，长度为returndatasize()的数据（格式为bytes）
        default {
            return(0, returndatasize())
        }
    }
}
```



普通的代理合约Proxy.sol有两个缺点：

1. 为了插槽位置一致， 每个代理、逻辑合约都要包含一个implement address
2. 选择器冲突selector clash。函数选择器（selector）是函数签名的哈希的前4个字节。例如`mint(address account)`的选择器为`bytes4(keccak256("mint(address)"))`，也就是`0x6a627842`由于函数选择器仅有4个字节，范围很小，因此两个不同的函数可能会有相同的选择器，在同一个合约里，如果selector clash了，编译会报错，但是如果代理合约和逻辑合约的selector clash了，就会造成一些意想不到的错误，比如如果逻辑合约的`a`函数和代理合约的升级函数的选择器相同，那么管理人就会在调用`a`函数的时候，将代理合约升级成一个黑洞合约

ERC1967可以解决问题1

透明代理`Transparent Proxy`和通用可升级代理`UUPS`可以解决问题2



ERC1976定义了特殊的slot来存储逻辑合约地址，这个slot是constant类型的，不占据storage存储空间。

```solidity
bytes32 internal constant IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
```

升级函数会读取IMPLEMENTATION_SLOT位置的数据并修改

```solidity
function upgradeToAndCall(address newImplementation, bytes memory data) internal
```

```solidity
function _setImplementation(address newImplementation) private {
    require(Address.isContract(newImplementation), "ERC1967: new implementation is not a contract");
    StorageSlot.getAddressSlot(_IMPLEMENTATION_SLOT).value = newImplementation;
}
```



#### transparent

透明代理通过设置权限检验来解决升级函数选择器冲突问题

管理员只能调用代理合约的升级函数对合约升级，不能通过回调函数调用逻辑合约。

其它用户不能调用可升级函数，但是可以调用逻辑合约的函数

```solidity
    function upgradeAndCall(
        ITransparentUpgradeableProxy proxy,
        address implementation,
        bytes memory data
    ) public payable virtual onlyOwner {
        proxy.upgradeToAndCall{value: msg.value}(implementation, data);
    }
```

```solidity
function _fallback() internal virtual override {
        if (msg.sender == _proxyAdmin()) {
            if (msg.sig != ITransparentUpgradeableProxy.upgradeToAndCall.selector) {
                revert ProxyDeniedAdminAccess();
            } else {
                _dispatchUpgradeToAndCall();
            }
        } else {
            super._fallback();
        }
    }
```



#### UUPS

UUPS（universal upgradeable proxy standard，通用可升级代理）将升级函数也放在逻辑合约中。这样一来，如果有其它函数与升级函数存在“选择器冲突”，编译时就会报错。

UUPS合约可以将升级函数放在逻辑合约中，并检查调用者是否为管理员

```
function upgradeToAndCall(address newImplementation, bytes memory data) public payable virtual onlyProxy
```

![img](https://www.wtf.academy/assets/images/49-1-6d28dd55317040f7cbd5866bd4ec613d.png)



#### 信标代理

以上两种代理，都存在一种缺陷，就是如果我要升级一批具有相同逻辑合约的代理合约，那么需要在每个代理合约都执行一遍升级（因为每个代理合约独立存储了_implementation）。信标合约，就是将所有的具有相同逻辑合约的代理合约的_implementation 只存一份在信标合约中，所有的代理合约通过和信标合约接口调用，获取_implementation，这样，在升级的时候，就可以只升级信标合约，就能搞定所有的代理合约的升级

从信标合约获取逻辑合约地址

```solidity
function implementation() external view returns (address);
```

升级时直接升级信标合约的implementation即可

```solidity
function upgradeTo(address newImplementation) public virtual onlyOwner
```

