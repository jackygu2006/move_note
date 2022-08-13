# MOVE语言学习笔记
## 1. 基本语法
### 1.1 基本数据类型
* 整型：`u8`, `u64`, `u128`
* 布尔：`bool`
* 地址：`address`
* 结构体：`struct {}`
  
强制转换：`as`

### 1.2 表达式
* 语句用`;` 隔开
* 作为作用域或者函数返回值时，不带`;`
* 赋值：`let`
* 未使用：`_`，注意move所有变量都必须被使用，否则使用`let _ = xxx`

### 1.3 块与作用域
* `{}`
* 变量仅存在于其作用域（或代码块）内，当作用域结束时变量随之消亡

### 条件控制
* `if() {} else {};`
* 可以`let a = if() {} else {};`为`a`赋值
* 可以不用`else`，但是如果把if复制给一个变量，必须有else
* 注意必须用`;`结尾

### 循环
* `while() {};`
* `while`不能把自己赋值给其他变量
* 无限循环`loop {};`
* 可以在循环中加`break`或`continue`
* 注意，如果`break`和`continue`是代码块中的最后一个关键字，则不能在其后加分号，因为后面的任何代码都不会被执行。
* 中止：`abort(0);`
* 有条件中止：`aborts_if x == 0;`
* `assert!(<condition>, <code>)`
  
### 模块
* 模块是发布在特定地址下的打包在一起的一组函数和结构体
* 模块在发布者的地址下发布。标准库在 0x1 地址下发布，用`0x1::模块名`引用，如`0x1::String`
* 定义模块
  ```
	module Module {
		public fun() {}
	}
	```
* 默认情况下，模块将在发布者的地址下进行编译和发布。但如果只是测试或开发，或者想要在模块中指定地址，使用`address {}`，如：
  ```
	address 0x1 {
		module Module {
			...
		}
	}
	```
* `use`导入模块，如：`use 0x1::Vector;`，使用`Vector::empty<u64>();`创建数组
* 可以直接导入模块的方法：如`use 0x1::Signer::address_of;`后, 直接使用`address_of(account);`
* 同时导入模块和模块方法：
  ```
	use 0x1::Vector::{
		Self, // Self == Imported module
		empty
	};
	```
* 使用`use as`，为导入模块自定义名称，如：`use 0x1::Vector as V;`后，使用`V.empty<u64>()`创建数组

### 常量
* 如：`const RECEIVER : address = 0x999;`
* 常量只对定义它们的脚本或模块来说是本地可见的

### 函数
* `fun 函数名(参数:参数类型,...): 返回值类型 {}`
* Move函数使用`snake_case`命名规则，也就是小写字母以及下划线作为单词分隔符。
* `main()`主函数，没有返回值
* 多个返回值，用`(返回值1,返回值2,返回值3)`，返回值类型必须要和函数定义中一致
* 函数默认不可见，即私有函数，只能在模块内访问
* `public fun`可以被外部访问


## 2. 高阶数据结构
### 2.1 结构体
* 定义
```
struct Country {
	id: u8,
	population: u64
}
```

* 实例化
```
let country = Country {
	id: c_id,
	population: c_population
};
```
* 访问结构体成员，用`.`
* 回收/销毁结构体
```
let Country { id: _, population: _ } = country;
```
注意：Resource 结构体则必需被销毁

### 2.2 Abilities
* key: 被修饰的值可以作为键值对全局状态进行访问
* copy: 被修饰的值可以被复制
* store: 被修饰的值可以被存储到全局状态
* drop: 被修饰的值在作用域结束时可以被丢弃
* 定义方法：在结构体中用`has`
* 注：Destructuring 并不需要 Drop ability

### 2.3 所有权和引用
* 每个变量只有一个所有者，当把变量作为参数传递给函数时，该函数将成为新所有者，并且第一个函数不再拥有该变量。
* 调用函数时，默认`move`关键词：`M::value(move a);` 等同于 `M::value(a);`
* 副本传递参数，使用`copy`关键词：`M::value(copy a);`，这是`a`仍旧可以在当前作用域中有效
* 不可改变参数的引用：`&`
* 可改变参数的引用：`&mut `，注意与参数名之间有空格
* Borrow检查：类似于借用的概念
* 获取引用参数的值：`*`，该符号实际上是产生了一个副本，要确保这个值具有 Copy ability。
* 复制一个结构体的字段：`*&`
* 基本类型引用（如`u8`，`u64`等），默认使用`copy`方式

### 2.4 泛型：不限于指定的数据类型
* 如：
```
module Storage {
	struct Box<T> {
			value: T
	}
}
```
* abilities限制符，如：
```
fun name<T: key + store + drop + copy>() {}
```
不常用，建议在定义结构体中用`has`设定`Ability`

* 并非泛型中指定的每种类型参数都必须被使用

### 2.5 数组
* 定义一个数组：
```
use 0x1::Vector;
let a = Vector::empty<&u8>();
```
* 获取数组长度
```
let a_len = Vector::length(&a);
```

* 压入数据
```
Vector::push_back(&mut a, i);
```

* 弹出数据
```
Vector::pop_back(&mut a);
```
注意：以上需要改变数组内容，需要使用`&mut a`

* Vector 最多可以存储 18446744073709551615u64（u64最大值）个非引用类型的值
* 使用Vector表示字符串
```
let str = b"hello world";
let len = Vector::length<u8>(&str); // 获取字符串长度
```
* 对Vector的不可变的引用
```
Vector::borrow()
```
* 对Vector的可变的应用
```
Vector::borrow_mut<E>(v: &mut vector<E>, i: u64): &E;
```

## 3. Resource
* 定义：Resource 是一种用`key`和`store` ability 限制了的结构体，如：
```
module M {
	struct T has key, store {
			field: u8
	}
}
```
* 特点：
  * Resource 存储在帐户下，只有在分配帐户后才会存在，并且只能通过该帐户访问。
  * 一个帐户同一时刻只能容纳一个某类型的 Resource。
  * Resource 不能被复制
  * Resource 必需被使用
* 模块里最主要的 Resource 通常跟模块取相同的名称
* Resource 将永久保存在发送者的地址下，没有人可以从所有者那里修改或取走此 Resource。只能将 Resource 放在自己的帐户下。你无权访问另一个帐户的 signer 值，因此无法放置 Resource 到其它账户。
* 将一个Resource移至自己账户下：`move_to`
```
move_to<Collection>(&account, Collection {
	items: Vector::empty<Collection>()
})
```

* 从自己账户中取出并销毁Resource: `move_from`
```
let collection = move_from<Collection>(signer::address_of(&account));
let Collection { items: _ } = collection;
```
Resource 必需被使用。因此，从账户下取出 Resource 时，要么将其作为返回值传递，要么将其销毁。

* 查看账户下是否拥有Resource: `exists`
```
native fun exists<T: key>(addr: address): bool;
```
任何人都可以查看某个账号下是否拥有某个Resource

* 不可变的获取某个Resource: `native fun borrow_global<T: key>(addr: address): &T;`
```
use std::signer;
let owner = signer::address_of(account);
let collection = borrow_global<Collection>(owner);
```

* 可变的获取某个Resource: `native fun borrow_global_mut<T: key>(addr: address): &mut T;`
```
use std::signer;
public fun add_item(account: &signer) acquires T {
	let collection = borrow_global_mut<T>(signer::address_of(account));
	Vector::push_back(&mut collection.items, Item {});
}
```

* Acquires 关键字，用来显式定义此函数获取的所有 Resource
```
fun <name>(<args...>): <ret_type> acquires T, T1 ... {}
```

## 4. 常用snipes
### 4.1 输出
```
use 0x1::Debug;
Debug::print<u8>(number);
Debug::print<Vector<u8>>(b"Hello world!");
```

### 4.2 常用内置模块
```
use std::string;
use std::error;
use aptos_std::event;
use std::signer;
```

### 4.3 定义字符串
```
struct StringExample {
	str: string::String,
}
let str = string::utf8(b"Hello, Blockchain"),
```

### 4.4 定义Event
* 定义时间数据类
```
use aptos_std::event;
struct MessageChangeEvent has drop, store {
	from_message: string::String,
	to_message: string::String,
}
```

* 定义事件
```
event::EventHandle<MessageChangeEvent>
```

* 新建事件
```
event::new_event_handle<MessageChangeEvent>(&account)
```

* 发送事件
```
event::emit_event(&mut old_message_holder.message_change_events, MessageChangeEvent {
	from_message,
	to_message: copy message,
});
```

### 4.5 全局访问某个Resource的成员
```
let str: string::String = *&borrow_global<MessageHolder>(addr).message
```

### 4.6 error
```
error::not_found(0u8);
```

### 4.7 测试代码
函数名前加：
```
#[test(account = @0x1)]
public entry fun sender_can_set_message(account: signer) acquires MessageHolder {
		let addr = signer::address_of(&account);
		set_message(account,  b"Hello, Blockchain");

		// 校验
		assert!(
			get_message(addr) == string::utf8(b"Hello, Blockchain"),
			ENO_MESSAGE
		);
}
```

### 4.8 ACL控制
```
use std::acl::Self;
acl::empty(); // 初始化
acl::add(&mut participants, signer::address_of(account)); // 添加
acl::remove(&mut participants, participant); // 移除
```

看完move语言，来说说为啥aptos等基于move语言的公链标榜自己安全了，理由：
* 1- Resource（资产）拥有严格结构定义，只拥有key和store两个权限；
* 2- 所有Resource（资产）都在创建人的账号中，其他人要调用，只能用borrow等方法，相当于借用；
* 3- 资产的转移（transfer）不像ERC20等，是通过对数值的加加减减完成转移，而是通过move_to, move_from, borrow*等方法操作；
* 4- 对Resource（资产）有严格的作用域外销毁机制，不会冒出新的不明资产；
* 5- 所有方法默认private，防止程序员在写合约时忘了定义，默认成public，被黑客利用；
* 6- 没有owner的概念，所有资产都在创建者账号下面，确保创建人对资产的控制权；
* 7- 全局访问收到严格限制，没有delegatecall等敏感的调用方法；
* 8- 在函数间传递参数时，严格的规定了那些能改（&mut），那些参数不能被改动，并且默认不能被改动；
* 9- 参数的引用按照资产转移的理念，在默认情况下，一个参数被另外一个函数处理时，在原函数中无法使用；
* 10- 标准数据类型很简单，开发过程不用担心溢出，这是很多以太坊上黑客攻击的重灾区；

其实吧，如果一个成熟的程序员用solidity写智能合约，认真点，上述安全问题还是能够避免的，毕竟现在solidity程序员不像几年前的了，那会菜鸟太多。