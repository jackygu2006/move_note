# 在aptos测试链上部署并运行move Module

注：`Module`相当于`Solidity`的智能合约。

假设以下操作的`move`工作目录为：`~/aptos-core/aptos-move/move-examples/hello_blockchain`，该目录下有`./sources/HelloBlockchain.move`和`HelloBlockchainTest.move`两个`move`源程序。

另外还有一个前端脚本目录：`~/aptos-core/developer-docs-site/static/examples/`。

同时开两个窗口，分别在上面两个目录中操作。

## 1. 准备
检测是否安装`aptos`cli开发工具，检测命令`aptos -h`。

如果显示`aptos`没有安装的话，执行：`cargo install --git https://github.com/aptos-labs/aptos-core.git aptos`

## 2. 本地调试move模块
在`move`中，加入测试代码。用`#[test(account = @0x1)]`表示测试开始。函数类型为`public entry fun`，一般用`assert!()`来判断测试结果是否符合要求
```
#[test(account = @0x1)]
public entry fun sender_can_set_message(account: signer) acquires MessageHolder {
		let addr = signer::address_of(&account);
		set_message(account,  b"Hello, Blockchain");

		assert!(
			get_message(addr) == string::utf8(b"Hello, Blockchain"),
			ENO_MESSAGE
		);
}
```
调试命令：`cargo test`。

## 3. 部署
### 3.1 部署流程介绍
因为目前还没有成熟的IDE，所以，只能手动部署模块。

部署分两步，第一步是将move的模块源码编译成字节码，第二步是将字节码部署到链上。

因为`aptos`链的模块必须部署在账号下，所以，在部署模块时，必须有部署人的账号。

### 3.2 执行ts脚本
进入部署与测试脚本（部署与测试都放在一起，只是`payload`不同而已）目录：`~/aptos-core/developer-docs-site/static/examples/`，

在这里提供了三种前端部署的语言：`python`，`typescript`，`rust`。

我们选择`typescript`，进入目录，打开：`hello_blockchain.ts`脚本，这里一共有四个方法：
* main：主入口
* publishModule：发布模块到链上
* getMessage：测试从前端获取message
* setMessage：测试从前端设置链上数据

`yarn hello_blockchain message.mv`，运行该脚本。

注意：这时候`message.mv`并不存在同级目录下，等会会说到。

#### 第一步：创建Alice和Bob的账号
在ts脚本中，执行下面代码，得到`Alice`和`Bob`的两个地址：
```
const alice = new AptosAccount();
const bob = new AptosAccount();

console.log("\n=== Addresses ===");
console.log(`Alice: ${alice.address()}`);
console.log(`Bob: ${bob.address()}`);
```

### 第二步：从水龙头领币
使用如下代码从水龙头给这两个账号充测试币，
```
await faucetClient.fundAccount(alice.address(), 5_000);
await faucetClient.fundAccount(bob.address(), 5_000);
```

显示结果如下：
```
=== Addresses ===
Alice: 0x5141733d0caef9dac90b6da394721821679310e9d3934778acf25fbf50549d60 // 地址会变，每次执行都不一样
Bob: 0xa824e2f88462960bded7af2ed0fe3576acc06e76f096dc77978ebd32cb614951 // 地址会变

=== Initial Balances ===
Alice: 5000
Bob: 5000
```

**当看到：`Update the module with Alice's address, build, copy to the provided path, and press enter.`时，将会暂停。**

#### 第二步：编译成mv字节码
回到`move`的工作目录中，修改`Move.coml`中的`HelloBlockchain`为`Alice`地址，如果想由`Bob`部署该模块，则写`Bob`的地址。

在该工作目录中，执行：
```
aptos move compile
```

执行后，会在工作目录下新建一个`build`目录，并会在`build`下按照`BuildInfo.yaml`中的`package_name`再建一个目录，包括`abi`，`bytecode`，`source_maps`的文件都在该目录下。

其中我们需要的是在`bytecode_modules`目录下的`message.mv`，这是一个二进制文件。

#### 第三步：部署到链
将在`move`工作目录下生成的`message.mv`复制到ts脚本目录，即`hello_blockchain.ts`所在的目录中。

回到第一步暂停的终端窗口，按回车键，读取字节码，并部署到链上。

这一步执行的ts代码为：
```
const moduleHex = fs.readFileSync(modulePath).toString("hex");
let txHash = await publishModule(alice, moduleHex);
await client.waitForTransaction(txHash);
```
其中`modulePath`就是脚本命令：`yarn hello_blockchain message.mv`中的`message.mv`，也就是同目录下的字节码文件名。如果字节码文件不与ts脚本在同一目录，则需要输入路径。

## 4. 调用模块方法
完成模块部署后，按脚本代码，将开始测试模块方法，下面是`Alice`调用`setMessasge`方法写入数据：
```
txHash = await setMessage(alice.address(), alice, "Hello, Blockchain");
await client.waitForTransaction(txHash);
```

`setMessage`和`getMessage`的实现代码如下：
```
async function setMessage(contractAddress: HexString, accountFrom: AptosAccount, message: string): Promise<string> {
  const scriptFunctionPayload = new TxnBuilderTypes.TransactionPayloadScriptFunction(
    TxnBuilderTypes.ScriptFunction.natural(
      `${contractAddress.toString()}::message`,
      "set_message",
      [],
      [BCS.bcsSerializeStr(message)],
    ),
  );

  const [{ sequence_number: sequenceNumber }, chainId] = await Promise.all([
    client.getAccount(accountFrom.address()),
    client.getChainId(),
  ]);

  const rawTxn = new TxnBuilderTypes.RawTransaction(
    TxnBuilderTypes.AccountAddress.fromHex(accountFrom.address()),
    BigInt(sequenceNumber),
    scriptFunctionPayload,
    1000n,
    1n,
    BigInt(Math.floor(Date.now() / 1000) + 10),
    new TxnBuilderTypes.ChainId(chainId),
  );

  const bcsTxn = AptosClient.generateBCSTransaction(accountFrom, rawTxn);
  const transactionRes = await client.submitSignedBCSTransaction(bcsTxn);

  return transactionRes.hash;
}

async function getMessage(contractAddress: HexString, accountAddress: MaybeHexString): Promise<string> {
  try {
    const resource = await client.getAccountResource(
      accountAddress,
      `${contractAddress.toString()}::message::MessageHolder`,
    );
    return (resource as any).data["message"];
  } catch (_) {
    return "";
  }
}

```

如果执行正常，会显示如下测试结果：

```
=== Testing Alice ===
Publishing...
Initial value:
Setting the message to "Hello, Blockchain"
New value: Hello, Blockchain

=== Testing Bob ===
Initial value:
Setting the message to "Hello, Blockchain"
New value: Hello, Blockchain
```
这说明链上模块执行正常。
