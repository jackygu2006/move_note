# 部署并运行Module
Module相当于Solidity的智能合约

假设以下操作的`move`工作目录为：`~/aptos-core/aptos-move/move-examples/hello_blockchain`，该目录下有`./sources/HelloBlockchain.move`和`HelloBlockchainTest.move`两个`move`源程序。

## 1. 调试
在move中，加入测试代码，一般用assert!()来判断测试结果是否符合要求
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
调试命令：`cargo test`

## 2. 创建账号
如果已经由账号，跳过此步骤。

在ts脚本中，执行下面代码，得到`Alice`和`Bob`的两个地址：
```
  const alice = new AptosAccount();
  const bob = new AptosAccount();

  console.log("\n=== Addresses ===");
  console.log(`Alice: ${alice.address()}`);
  console.log(`Bob: ${bob.address()}`);
```

## 2. 编译成mv字节码
回到`move`的工作目录中，修改`Move.coml`中的`HelloBlockchain`为`Alice`地址，如果想由`Bob`部署该模块，则写`Bob`的地址。

在该工作目录中，执行：
```
aptos move compile
```

执行后，会在工作目录下新建一个`build`目录，并会在`build`下按照`BuildInfo.yaml`中的`package_name`再建一个目录，包括`abi`，`bytecode`，`source_maps`的文件都在该目录下。

其中我们需要的是在`bytecode_modules`目录下的`message.mv`，这是一个二进制文件。

如果显示`aptos`没有安装的话，执行：`cargo install --git https://github.com/aptos-labs/aptos-core.git aptos`



## 3. 部署
在这里，使用Javascript方式来部署模块。

部署

## 4. 调用模块方法

