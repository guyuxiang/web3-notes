# Solana SDK

### 事件

要在客户端应用程序中接收已发出的事件，请使用此 [`addEventListener()`](https://github.com/coral-xyz/anchor/blob/0e5285aecdf410fa0779b7cd09a47f235882c156/ts/packages/anchor/src/program/event.ts#L74-L123) 方法。此方法会自动 [解析和解码](https://github.com/coral-xyz/anchor/blob/0e5285aecdf410fa0779b7cd09a47f235882c156/ts/packages/anchor/src/program/event.ts#L232-L253) 程序日志中的事件数据。

```
import * as anchor from "@anchor-lang/core";
import { Program } from "@anchor-lang/core";
import { Event } from "../target/types/event";
 
describe("event", () => {
  // Configure the client to use the local cluster.
  anchor.setProvider(anchor.AnchorProvider.env());
 
  const program = anchor.workspace.Event as Program<Event>;
 
  it("Emits custom event", async () => {
    // Set up listener before sending transaction


    const listenerId = program.addEventListener("customEvent", event => {
      // Do something with the event data
      console.log("Event Data:", event);
    });
 
    // Message to be emitted in the event
    const message = "Hello, Solana!";
    // Send transaction
    await program.methods.emitEvent(message).rpc();
 
    // Remove listener
    await program.removeEventListener(listenerId);
  });
});
```



### 错误

当程序发生错误时，Anchor 的 TypeScript 客户端 SDK 会返回 包含错误信息的详细[错误响应。以下是一个错误响应示例，展示了其结构和可用字段：](https://github.com/coral-xyz/anchor/blob/0e5285aecdf410fa0779b7cd09a47f235882c156/ts/packages/anchor/src/error.ts#L51-L71)

```
{
  errorLogs: [
    'Program log: AnchorError thrown in programs/custom-error/src/lib.rs:11. Error Code: AmountTooLarge. Error Number: 6001. Error Message: Amount must be less than or equal to 100.'
  ],
  logs: [
    'Program 9oECKMeeyf1fWNPKzyrB2x1AbLjHDFjs139kEyFwBpoV invoke [1]',
    'Program log: Instruction: ValidateAmount',
    'Program log: AnchorError thrown in programs/custom-error/src/lib.rs:11. Error Code: AmountTooLarge. Error Number: 6001. Error Message: Amount must be less than or equal to 100.',
    'Program 9oECKMeeyf1fWNPKzyrB2x1AbLjHDFjs139kEyFwBpoV consumed 2153 of 200000 compute units',
    'Program 9oECKMeeyf1fWNPKzyrB2x1AbLjHDFjs139kEyFwBpoV failed: custom program error: 0x1771'
  ],
  error: {
    errorCode: { code: 'AmountTooLarge', number: 6001 },
    errorMessage: 'Amount must be less than or equal to 100',
    comparedValues: undefined,
    origin: { file: 'programs/custom-error/src/lib.rs', line: 11 }
  },
  _programErrorStack: ProgramErrorStack {
    stack: [
      [PublicKey [PublicKey(9oECKMeeyf1fWNPKzyrB2x1AbLjHDFjs139kEyFwBpoV)]]
    ]
  }
}
```





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

