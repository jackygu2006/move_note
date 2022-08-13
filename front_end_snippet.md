# 前端ts与合约交互的常用代码段

## 引入常用包
```
import { AptosAccount, TxnBuilderTypes, BCS, MaybeHexString, HexString, AptosClient, FaucetClient } from "aptos";
```
## 初始化客户端
```
export const NODE_URL = "https://fullnode.devnet.aptoslabs.com";
const client = new AptosClient(NODE_URL);
```

## 初始化水龙头
```
export const FAUCET_URL = "https://faucet.devnet.aptoslabs.com";
const faucetClient = new FaucetClient(NODE_URL, FAUCET_URL);
```

## 根据私钥获取账户对象
```
const privateKeyHexToAccount = (hexString: string): AptosAccount =>
	new AptosAccount(new HexString(hexString).toUint8Array());
```

## 获取账户余额
```
async function accountBalance(accountAddress: MaybeHexString): Promise<number | null> {
	try{
		const resource = await client.getAccountResource(
			accountAddress, 
			"0x1::coin::CoinStore<0x1::aptos_coin::AptosCoin>"
		);
		if (resource == null) return null;
		// resource.data.coin.value中获得余额
		return parseInt((resource.data as any)["coin"]["value"]);
	} catch (_) {
		return 0;
	}
}
```
以上是获取主链币余额，如果是某一个资产代币，则将`0x1::coin::CoinStore<0x1::aptos_coin::AptosCoin>`中内容做如下修改：
* 把<>中的`0x1`改为代币合约账号，也就是创建者的账号；
* 把`aptos_coin`改为需要的module名；
* 把`AptosCoin`改为需要的struct名；

## 转账
```
async function transfer(accountFrom: AptosAccount, recipient: MaybeHexString, amount: number): Promise<string> {
  const token = new TxnBuilderTypes.TypeTagStruct(TxnBuilderTypes.StructTag.fromString("0x1::aptos_coin::AptosCoin"));

  const scriptFunctionPayload = new TxnBuilderTypes.TransactionPayloadScriptFunction(
    TxnBuilderTypes.ScriptFunction.natural(
      "0x1::coin",
      "transfer",
      [token],
      [BCS.bcsToBytes(TxnBuilderTypes.AccountAddress.fromHex(recipient)), BCS.bcsSerializeUint64(amount)],
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
    1000n, // 1000个代币
    1n,
    BigInt(Math.floor(Date.now() / 1000) + 10),
    new TxnBuilderTypes.ChainId(chainId),
  );

  const bcsTxn = AptosClient.generateBCSTransaction(accountFrom, rawTxn);
  const pendingTxn = await client.submitSignedBCSTransaction(bcsTxn);

  return pendingTxn.hash;
}
```

## 等待交易确认
```
await client.waitForTransaction(txHash);
```

## 创建代币
使用标准库`0x1::coin_manager::initialize`创建一个默认代币，该代币拥有 mint/burn/transfer，以及事件（register, deposit, withdraw）功能。

标准代码如下：
```
const client = new AptosClient(NODE_URL);
/** Initializes the new coin */
async function initializeCoin(accountFrom: AptosAccount, coinTypeAddress: HexString): Promise<string> {
  const token = new TxnBuilderTypes.TypeTagStruct(
    TxnBuilderTypes.StructTag.fromString(`${coinTypeAddress.hex()}::moon_coin::MoonCoin`),
  );

  const serializer = new BCS.Serializer();
  serializer.serializeBool(false);

  const scriptFunctionPayload = new TxnBuilderTypes.TransactionPayloadScriptFunction(
    TxnBuilderTypes.ScriptFunction.natural(
      "0x1::managed_coin",
      "initialize",
      [token],
      [BCS.bcsSerializeStr("Moon Coin"), BCS.bcsSerializeStr("MOON"), BCS.bcsSerializeUint64(6), serializer.getBytes()],
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
  const pendingTxn = await client.submitSignedBCSTransaction(bcsTxn);

  return pendingTxn.hash;
}
```
`initializeCoin`方法的`accountFrom`是操作账户，`coinTypeAddress`: 是部署的MoonCoin合约，合约代码很简单：
```
module MoonCoinType::moon_coin {
    struct MoonCoin {}
}
```
可以在该模块中加入自定义功能。


## 注册代币
在`move`语言中，只有经过注册的代币，才能被接受方收到。
```
/** Receiver needs to register the coin before they can receive it */
async function registerCoin(coinReceiver: AptosAccount, coinTypeAddress: HexString): Promise<string> {
  const token = new TxnBuilderTypes.TypeTagStruct(
    TxnBuilderTypes.StructTag.fromString(`${coinTypeAddress.hex()}::moon_coin::MoonCoin`),
  );

  const scriptFunctionPayload = new TxnBuilderTypes.TransactionPayloadScriptFunction(
    TxnBuilderTypes.ScriptFunction.natural("0x1::coins", "register", [token], []),
  );

  const [{ sequence_number: sequenceNumber }, chainId] = await Promise.all([
    client.getAccount(coinReceiver.address()),
    client.getChainId(),
  ]);

  const rawTxn = new TxnBuilderTypes.RawTransaction(
    TxnBuilderTypes.AccountAddress.fromHex(coinReceiver.address()),
    BigInt(sequenceNumber),
    scriptFunctionPayload,
    1000n,
    1n,
    BigInt(Math.floor(Date.now() / 1000) + 10),
    new TxnBuilderTypes.ChainId(chainId),
  );

  const bcsTxn = AptosClient.generateBCSTransaction(coinReceiver, rawTxn);
  const pendingTxn = await client.submitSignedBCSTransaction(bcsTxn);

  return pendingTxn.hash;
}
```

## 铸币
代币所有人调用`0x1::managed_coin::mint`为铸币，并发送到指定接受人账号，注意该接受人必须注册了该代币。

```
/** Mints the newly created coin to a specified receiver address */
async function mintCoin(
  coinOwner: AptosAccount,
  coinTypeAddress: HexString,
  receiverAddress: HexString,
  amount: number,
): Promise<string> {
  const token = new TxnBuilderTypes.TypeTagStruct(
		TxnBuilderTypes.StructTag.fromString(`${coinTypeAddress.hex()}::moon_coin::MoonCoin`),
	);

  const scriptFunctionPayload = new TxnBuilderTypes.TransactionPayloadScriptFunction(
    TxnBuilderTypes.ScriptFunction.natural(
      "0x1::managed_coin",
      "mint",
      [token],
      [BCS.bcsToBytes(TxnBuilderTypes.AccountAddress.fromHex(receiverAddress.hex())), BCS.bcsSerializeUint64(amount)],
    ),
  );

  const [{ sequence_number: sequenceNumber }, chainId] = await Promise.all([
    client.getAccount(coinOwner.address()),
    client.getChainId(),
  ]);

  const rawTxn = new TxnBuilderTypes.RawTransaction(
    TxnBuilderTypes.AccountAddress.fromHex(coinOwner.address()),
    BigInt(sequenceNumber),
    scriptFunctionPayload,
    1000n,
    1n,
    BigInt(Math.floor(Date.now() / 1000) + 10),
    new TxnBuilderTypes.ChainId(chainId),
  );

  const bcsTxn = AptosClient.generateBCSTransaction(coinOwner, rawTxn);
  const pendingTxn = await client.submitSignedBCSTransaction(bcsTxn);
  return pendingTxn.hash;
}
```










## 部署模块
```
/** Publish a new module to the blockchain within the specified account */
export async function publishModule(accountFrom: AptosAccount, moduleHex: string): Promise<string> {
  const moudleBundlePayload = new TxnBuilderTypes.TransactionPayloadModuleBundle(
    new TxnBuilderTypes.ModuleBundle([new TxnBuilderTypes.Module(new HexString(moduleHex).toUint8Array())]),
  );

  const [{ sequence_number: sequenceNumber }, chainId] = await Promise.all([
    client.getAccount(accountFrom.address()),
    client.getChainId(),
  ]);

  const rawTxn = new TxnBuilderTypes.RawTransaction(
    TxnBuilderTypes.AccountAddress.fromHex(accountFrom.address()),
    BigInt(sequenceNumber),
    moudleBundlePayload,
    1000n,
    1n,
    BigInt(Math.floor(Date.now() / 1000) + 10),
    new TxnBuilderTypes.ChainId(chainId),
  );

  const bcsTxn = AptosClient.generateBCSTransaction(accountFrom, rawTxn);
  const transactionRes = await client.submitSignedBCSTransaction(bcsTxn);

  return transactionRes.hash;
}
```

`moduleHex`是`mv`后缀的字节码文件内容，用：
```
const moduleHex = fs.readFileSync(modulePath).toString("hex");
```

## 调用合约方法，改变链的状态
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
```

## 从模块中读取数据
```
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
对应move的方法：
```
public fun get_message(addr: address): string::String acquires MessageHolder {
		assert!(exists<MessageHolder>(addr), error::not_found(ENO_MESSAGE));
		*&borrow_global<MessageHolder>(addr).message
}
```