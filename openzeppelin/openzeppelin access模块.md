

## Openzeppelin Access模块

#### AccessControl合约

该合约为角色管理合约, 其中有两个关键角色：role和roleAdmin

每个角色role维护其下的多个地址, 和其管理员角色roleAdmin

其管理员角色roleAdmin下也维护有自己的多个地址

![image-20240416161800385](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20240416161800385.png)

查询某地址是否有某角色:

```solidity
function hasRole(bytes32 role, address account) external view returns (bool);
```

查询某角色的管理员角色:

```solidity
 function getRoleAdmin(bytes32 role) external view returns (bytes32);
```

设置某角色的管理员角色(不对外, 为了合约的安全性最好只在构造方法中调用一次, 不设置默认为默认管理员角色 DEFAULT_ADMIN_ROLE = 0x00):

```solidity
 function _setRoleAdmin(bytes32 role, bytes32 adminRole) internal virtual
```



具有roleAdmin角色的用户可以使用以下函数对role角色进行操作:

给指定地址授权指定角色:

```solidity
function grantRole(bytes32 role, address account) public virtual override onlyRole(getRoleAdmin(role)){};
```

移除指定地址的角色
```solidity
function revokeRole(bytes32 role, address account) public virtual override onlyRole(getRoleAdmin(role)){};
```



具有某role角色的用户可以使用以下函数:

自己抛弃自己的指定角色:

```solidity
function renounceRole(bytes32 role, address account) public virtual override{};
```



使用该合约的流程:

在构造函数中

1. 创建管理员角色

2. 授权指定地址为管理员角色

3. 创建业务角色

4. 设置业务角色的管理员角色

   

#### AccessControlDefaultAdminRules合约

todo:整理该合约的具体使用场景

该合约继承AccessControl合约, 拓展了对默认管理员角色(DEFAULT_ADMIN_ROLE = 0x00)的操作, 其中“默认管理员”角色中最多只能有一个地址

在构造函数中传入默认管理员角色账户和变更账户的延迟时间

```solidity
constructor(uint48 initialDelay, address initialDefaultAdmin) {
    if (initialDefaultAdmin == address(0)) {
        revert AccessControlInvalidDefaultAdmin(address(0));
    }
    _currentDelay = initialDelay;
    _grantRole(DEFAULT_ADMIN_ROLE, initialDefaultAdmin);
}
```



转让默认管理员角色的地址，同时会设置一个时间延迟(只能由原默认管理员角色地址调用), 生成生效时间点，在当前时间基础上加上“延迟时间”参数: 

```solidity
function beginDefaultAdminTransfer(address newAdmin) public virtual onlyRole(DEFAULT_ADMIN_ROLE)
```



新的账户需要在时间延迟后发起接受，才能完成管理员账户转移:

```solidity
function acceptDefaultAdminTransfer() public virtual
```



在新账户接受成为“默认管理员”身份组账户之前，当前“默认管理员”身份组账户可以取消上一步的账户变更:

```solidity
function cancelDefaultAdminTransfer() public virtual onlyRole(DEFAULT_ADMIN_ROLE)
```



修改账户变更时“时间延迟”参数(当前“默认管理员”身份组账户),“时间延迟”参数修改的生效也有延迟 :

```solidity
function changeDefaultAdminDelay(uint48 newDelay) public virtual onlyRole(DEFAULT_ADMIN_ROLE)
```



取消上一步“时间延迟”参数的修改(当前“默认管理员”身份组账户):

```solidity
function rollbackDefaultAdminDelay() public virtual onlyRole(DEFAULT_ADMIN_ROLE)
```




#### AccessControlEnumerable合约

继承AccessControl合约, 引入openzeppelin-contracts/contracts/utils/structs/EnumerableSet.sol, 为每个角色新增了一个可列举成员的集合结构

在_grantRole和_revokeRole的过程中,会对该集合进行增减

```solidity
 mapping(bytes32 role => EnumerableSet.AddressSet) private _roleMembers;
```

获取某角色下指定索引(0~getRoleMemberCount)的地址

```solidity
function getRoleMember(bytes32 role, uint256 index) public view virtual returns (address)
```

获取某角色下的地址数

```solidity
function getRoleMemberCount(bytes32 role) public view virtual returns (uint256)
```



#### Ownable合约

继承该合约后合约部署者为最初的合约所有者, 可根据业务需求来对使用以下功能

场景: 可升级合约在确定不升级后,可以使用销毁函数来防止再升级



使用修饰器 onlyOwner 来限制只有合约拥有者可以来进行操作:

```solidity
modifier onlyOwner()
```



销毁合约拥有者权限(需要合约所有者才能调用): 

```solidity
function renounceOwnership() public virtual onlyOwner
```



转移合约所有者权限:

```solidity
function transferOwnership(address newOwner) public virtual onlyOwner
```



#### Ownable2Step合约

该合约继承Ownable合约, 将所有权转移给对方, 对方需要接受才会真正转移



转移合约拥有者权限(设置待接受者):

```solidity
 function transferOwnership(address newOwner) public virtual override onlyOwner
```



接受所有权(调用者需要和待接收者匹配才能成功):

```solidity
function acceptOwnership() public virtual
```

