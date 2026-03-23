# anchor-go

https://github.com/gagliardetto/anchor-go

`anchor-go`[为使用anchor框架编写的](https://github.com/solana-foundation/anchor)[Solana](https://solana.com/)程序（智能合约）生成 Go 客户端。

它读取anchor 生成的[IDL](https://www.anchor-lang.com/docs/basics/idl)（接口定义语言）文件，并生成可用于与程序及其数据结构进行交互的 Go 代码。

```
go install github.com/gagliardetto/anchor-go@latest

# Generate code from an IDL file
anchor-go --idl /path/to/idl.json --output ./generated --program-id 0123456789abcdef0123456789abcdef0123456789

# This version of `anchor-go` only supports the IDL format of anchor starting with version v0.30.0.
# If you have an older version of an IDL (many programs still have those), you can convert them with `anchor idl convert <my-program-old-idl.json> > <my-program-NEW-idl.json>` to the new format.
```