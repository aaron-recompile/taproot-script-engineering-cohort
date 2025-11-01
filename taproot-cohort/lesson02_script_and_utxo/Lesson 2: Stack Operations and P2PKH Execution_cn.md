# 第 2 课：栈操作和 P2PKH 执行

**周次**: 2  
**时长**: 自主学习（预计 4-5 小时）  
**先修要求**: 第 1 课（密钥和地址）

---

## 🎯 学习目标

完成本课后，你将能够：

1. 解释 UTXO 模型及其与账户模型的区别
2. 理解 Bitcoin Script 的基于栈的执行模型
3. 构建和解码 P2PKH 锁定和解锁脚本
4. 逐步追踪 P2PKH 脚本执行过程
5. 在测试网上构建完整的 P2PKH 交易
6. 使用 StackFlow 可视化脚本执行

---

## 🧩 核心概念

### UTXO 模型：数字现金，而非数字银行

比特币不使用"账户余额"——它使用**未花费交易输出（UTXOs）**。

**心智模型对比：**

| 传统银行 | 比特币 UTXO 模型 |
|---------|----------------|
| 账户显示余额：$500 | 你有特定的"钞票"：$200 + $100 + $100 + $100 |
| 花费 $350，余额更新为 $150 | 花费 $350：提供 $400 的钞票，找回 $50 零钱 |
| 没有"零钱"的概念 | 零钱是一个新的 UTXO |
| 基于状态（可变余额） | 基于交易（不可变的 UTXO） |

**UTXO 示例：Alice 向 Bob 发送 7 BTC**

```
初始状态:
  Alice 拥有: 10 BTC (UTXO #1)
  
交易:
  输入:  10 BTC (Alice 的 UTXO #1) [销毁]
  输出: 7 BTC  (新 UTXO → Bob)
  输出: 3 BTC  (新 UTXO → Alice 作为找零)
  
结果:
  Bob 拥有:   7 BTC (UTXO #2)
  Alice 拥有: 3 BTC (UTXO #3)
```

**关键 UTXO 属性：**

✅ **完全消费**：UTXO 必须完全花费（不能部分花费）  
✅ **原子创建**：所有输出要么全部创建，要么交易失败  
✅ **找零处理**：输入 - 输出 = 交易手续费  
✅ **并行处理**：每个 UTXO 只能花费一次（防止双花）  
✅ **唯一标识**：`交易ID:输出索引`（例如 `TX123:0`）

---

### Bitcoin Script：可编程的支出条件

每个 UTXO 包含一个**锁定脚本（ScriptPubKey）**，用于定义支出条件。

**脚本执行模型：**
```
解锁脚本 (ScriptSig) + 锁定脚本 (ScriptPubKey) → TRUE/FALSE
```

**组件：**

| 组件 | 位置 | 目的 | 示例 |
|------|------|------|------|
| **锁定脚本** | 附加到 UTXO 输出 | 定义支出条件 | "证明你拥有这个公钥哈希的私钥" |
| **解锁脚本** | 在花费时提供 | 满足条件 | "这是我的签名 + 公钥" |

**验证过程：**
1. 连接解锁脚本 + 锁定脚本
2. 在比特币虚拟机中作为单个程序执行
3. 如果执行后栈顶为 TRUE（非零）→ 有效花费

---

### 基于栈的执行：基础

Bitcoin Script 使用**后进先出（LIFO）栈**，类似于 Forth 或 PostScript。

**基本栈操作：**

```
初始状态（空栈）：
┌─────────────┐
│ (empty)     │
└─────────────┘

PUSH 3 后：
┌─────────────┐
│ 3           │
└─────────────┘

PUSH 5 后：
┌─────────────┐
│ 5           │ ← 栈顶
│ 3           │
└─────────────┘

OP_ADD 后：
┌─────────────┐
│ 8           │ ← 结果：5 + 3
└─────────────┘
```

**关键栈操作：**

| 操作 | 描述 | 效果 |
|------|------|------|
| `PUSH <data>` | 将数据压入栈 | 栈增长 |
| `OP_DUP` | 复制栈顶项 | `[x] → [x, x]` |
| `OP_HASH160` | 哈希栈顶项（SHA256→RIPEMD160） | `[data] → [hash]` |
| `OP_EQUALVERIFY` | 比较栈顶两项，不相等则失败 | `[x, x] → []` 或失败 |
| `OP_CHECKSIG` | 验证签名 | `[sig, pubkey] → [TRUE/FALSE]` |

---

### P2PKH：支付到公钥哈希

**最基础的比特币脚本**

**锁定脚本（ScriptPubKey）：**
```
OP_DUP OP_HASH160 <pubkey_hash> OP_EQUALVERIFY OP_CHECKSIG
```

**英文含义：**
> "这个 UTXO 可以被任何提供以下内容的人花费：
> 1. 一个哈希为 `<pubkey_hash>` 的公钥
> 2. 来自相应私钥的有效签名"

**解锁脚本（ScriptSig）：**
```
<signature> <public_key>
```

**可视化表示：**
```
ScriptSig           ScriptPubKey
┌─────────────┐    ┌───────────────────────────────────────────┐
│ <signature> │    │ OP_DUP OP_HASH160 <hash> OP_EQUALVERIFY  │
│ <pubkey>    │ +  │ OP_CHECKSIG                                │
└─────────────┘    └───────────────────────────────────────────┘
        ↓
   组合脚本执行
```

---

### 逐步 P2PKH 执行

让我们追踪一个真实的 P2PKH 交易执行：

**锁定脚本：**
```
OP_DUP OP_HASH160 <340cfcff...7a571> OP_EQUALVERIFY OP_CHECKSIG
```

**解锁脚本：**
```
<30440220...914f01>  // 签名 (71 字节)
<02898711...8519>    // 公钥 (33 字节)
```

**执行追踪：**

**步骤 1**：PUSH 签名
```
┌─────────────────────────┐
│ 30440220...914f01 (sig) │
└─────────────────────────┘
```

**步骤 2**：PUSH 公钥
```
┌─────────────────────────┐
│ 02898711...8519 (pk)    │ ← 栈顶
│ 30440220...914f01 (sig) │
└─────────────────────────┘
```

**步骤 3**：OP_DUP（复制栈顶项）
```
┌─────────────────────────┐
│ 02898711...8519 (pk)    │ ← 栈顶
│ 02898711...8519 (pk)    │
│ 30440220...914f01 (sig) │
└─────────────────────────┘
```

**步骤 4**：OP_HASH160（哈希栈顶公钥）
```
┌─────────────────────────┐
│ 340cfcff...7a571 (hash) │ ← 计算出的哈希
│ 02898711...8519 (pk)    │
│ 30440220...914f01 (sig) │
└─────────────────────────┘
```

**步骤 5**：PUSH 预期哈希（来自锁定脚本）
```
┌─────────────────────────┐
│ 340cfcff...7a571 (exp)  │ ← 预期哈希
│ 340cfcff...7a571 (comp) │ ← 计算出的哈希
│ 02898711...8519 (pk)    │
│ 30440220...914f01 (sig) │
└─────────────────────────┘
```

**步骤 6**：OP_EQUALVERIFY（比较，如果相等则移除两项）
```
┌─────────────────────────┐
│ 02898711...8519 (pk)    │ ← 栈顶
│ 30440220...914f01 (sig) │
└─────────────────────────┘
```
✅ 哈希匹配！继续执行。（如果不匹配，脚本在这里失败）

**步骤 7**：OP_CHECKSIG（验证签名与公钥和交易）
```
┌─────────────────────────┐
│ 1 (TRUE)                │ ← 签名有效！
└─────────────────────────┘
```

**结果**：栈顶为 TRUE → **交易有效！** ✅

---

## 💻 示例代码

### 设置：安装所需库

```bash
pip install python-bitcoinutils
```

### 示例 1：构建完整的 P2PKH 交易

```python
from bitcoinutils.setup import setup
from bitcoinutils.utils import to_satoshis
from bitcoinutils.transactions import Transaction, TxInput, TxOutput
from bitcoinutils.keys import P2wpkhAddress, P2pkhAddress, PrivateKey
from bitcoinutils.script import Script

def build_p2pkh_transaction():
    """
    在测试网上构建完整的 P2PKH 交易
    从传统 P2PKH 地址发送到 SegWit 地址
    """
    # 设置测试网环境
    setup('testnet')
    
    # 发送方信息 - 传统 P2PKH
    # 重要：使用你在第 1 课中的自己的私钥！
    private_key = PrivateKey('cPeon9fBsW2BxwJTALj3hGzh9vm8C52Uqsce7MzXGS1iFJkPF4AT')
    public_key = private_key.get_public_key()
    from_address_str = "myYHJtG3cyoRseuTwvViGHgP2efAvZkYa4"
    from_address = P2pkhAddress(from_address_str)
    
    # 接收方 - SegWit 地址
    to_address = P2wpkhAddress('tb1qckeg66a6jx3xjw5mrpmte5ujjv3cjrajtvm9r4')
    
    print("="*60)
    print("交易设置")
    print("="*60)
    print(f"发送方 (P2PKH):  {from_address_str}")
    print(f"接收方 (P2WPKH): {to_address.to_string()}")
    print()
    
    # 创建交易输入（引用先前的 UTXO）
    # 用你在第 1 课中的实际 UTXO 替换！
    txin = TxInput(
        '34b90a15d0a9ec9ff3d7bed2536533c73278a9559391cb8c9778b7e7141806f7',
        1  # vout 索引
    )
    
    # 计算金额
    total_input = 0.00029606  # 输入金额（BTC）
    amount_to_send = 0.00029400  # 发送金额
    fee = total_input - amount_to_send  # 交易手续费 (206 sats)
    
    print("="*60)
    print("金额")
    print("="*60)
    print(f"输入:  {total_input} BTC ({to_satoshis(total_input)} sats)")
    print(f"输出: {amount_to_send} BTC ({to_satoshis(amount_to_send)} sats)")
    print(f"手续费:    {fee} BTC ({to_satoshis(fee)} sats)")
    print()
    
    # 创建交易输出
    txout = TxOutput(
        to_satoshis(amount_to_send),
        to_address.to_script_pub_key()
    )
    
    # 创建未签名交易
    tx = Transaction([txin], [txout])
    
    print("="*60)
    print("未签名交易")
    print("="*60)
    print(tx.serialize())
    print()
    
    # 获取用于签名的 P2PKH 锁定脚本
    p2pkh_script = from_address.to_script_pub_key()
    
    print("="*60)
    print("锁定脚本 (ScriptPubKey)")
    print("="*60)
    print(f"十六进制: {p2pkh_script.to_hex()}")
    print(f"汇编: {p2pkh_script.to_string()}")
    print()
    
    # 签名交易输入
    signature = private_key.sign_input(tx, 0, p2pkh_script)
    
    # 创建解锁脚本：<signature> <public_key>
    txin.script_sig = Script([signature, public_key.to_hex()])
    
    # 获取已签名交易
    signed_tx = tx.serialize()
    
    print("="*60)
    print("已签名交易")
    print("="*60)
    print(signed_tx)
    print()
    print(f"交易大小: {tx.get_size()} 字节")
    print()
    
    # 显示解锁脚本
    print("="*60)
    print("解锁脚本 (ScriptSig)")
    print("="*60)
    print(f"十六进制: {txin.script_sig.to_hex()}")
    print()
    
    print("="*60)
    print("准备广播！")
    print("="*60)
    print("使用: bitcoin-cli sendrawtransaction <hex>")
    print("或: https://mempool.space/testnet/tx/push")
    
    return tx

if __name__ == "__main__":
    tx = build_p2pkh_transaction()
```

### 示例 2：解码和分析脚本组件

```python
from bitcoinutils.script import Script

def analyze_p2pkh_scripts():
    """
    解码和分析 P2PKH 锁定和解锁脚本
    """
    # 示例锁定脚本 (ScriptPubKey)
    locking_hex = "76a914c5b28d6bba91a2693a9b1876bcd3929323890fb288ac"
    locking_script = Script.from_raw(bytes.fromhex(locking_hex))
    
    print("="*60)
    print("P2PKH 锁定脚本分析")
    print("="*60)
    print(f"十六进制: {locking_hex}")
    print(f"汇编: {locking_script.to_string()}")
    print()
    print("分解：")
    print("  76    = OP_DUP")
    print("  a9    = OP_HASH160")
    print("  14    = OP_PUSHBYTES_20 (压入 20 字节)")
    print("  c5b2...0fb2 = 公钥哈希 (20 字节)")
    print("  88    = OP_EQUALVERIFY")
    print("  ac    = OP_CHECKSIG")
    print()
    
    # 示例解锁脚本 (ScriptSig)
    unlocking_hex = ("473044022055c309fe3f6099f4f881d0fd960923eb91aff0d8ef3501a2fc04"
                     "dce99aca6c230220222f8d5c6c14db9b85ca77b2fcffd8f8b06f2f3e1dcc3e93"
                     "2c1e2f9e6f1443d01210289871e6bf63f5cbe1b38c05e89d6c391c59e9f8f"
                     "695da44bf3d20ca674c8519")
    
    print("="*60)
    print("P2PKH 解锁脚本分析")
    print("="*60)
    print(f"十六进制: {unlocking_hex}")
    print()
    print("分解：")
    print("  47    = OP_PUSHBYTES_71 (压入 71 字节)")
    print("  3044...3d01 = DER 编码的签名 (71 字节)")
    print("  21    = OP_PUSHBYTES_33 (压入 33 字节)")
    print("  0289...8519 = 压缩公钥 (33 字节)")
    print()

if __name__ == "__main__":
    analyze_p2pkh_scripts()
```

### 示例 3：模拟栈执行

```python
def simulate_p2pkh_stack():
    """
    逐步模拟 P2PKH 栈执行
    （教育可视化）
    """
    print("="*60)
    print("P2PKH 栈执行模拟")
    print("="*60)
    print()
    
    stack = []
    
    def show_stack(operation):
        print(f"\n{operation}:")
        print("┌" + "─"*40 + "┐")
        if not stack:
            print("│ " + "(empty)".ljust(38) + " │")
        else:
            for i, item in enumerate(reversed(stack)):
                top_marker = " ← Top" if i == 0 else ""
                print(f"│ {item[:36].ljust(38)} │{top_marker}")
        print("└" + "─"*40 + "┘")
    
    # 步骤 1: 压入签名
    stack.append("30440220...914f01 (signature)")
    show_stack("步骤 1: PUSH <signature>")
    
    # 步骤 2: 压入公钥
    stack.append("02898711...8519 (public_key)")
    show_stack("步骤 2: PUSH <public_key>")
    
    # 步骤 3: OP_DUP
    stack.append(stack[-1])
    show_stack("步骤 3: OP_DUP")
    
    # 步骤 4: OP_HASH160
    stack[-1] = "340cfcff...7a571 (computed_hash)"
    show_stack("步骤 4: OP_HASH160")
    
    # 步骤 5: 压入预期哈希
    stack.append("340cfcff...7a571 (expected_hash)")
    show_stack("步骤 5: PUSH <expected_hash>")
    
    # 步骤 6: OP_EQUALVERIFY
    if stack[-1] == stack[-2].replace("computed", "expected"):
        stack.pop()  # 移除预期
        stack.pop()  # 移除计算值
        show_stack("步骤 6: OP_EQUALVERIFY (哈希匹配！)")
    
    # 步骤 7: OP_CHECKSIG
    stack.pop()  # 移除公钥
    stack.pop()  # 移除签名
    stack.append("1 (TRUE)")
    show_stack("步骤 7: OP_CHECKSIG")
    
    print("\n✅ 交易有效！栈顶为 TRUE。\n")

if __name__ == "__main__":
    simulate_p2pkh_stack()
```

---

## 🧪 实践任务

### 任务 1：构建并广播你的第一个 P2PKH 交易

**目标**：在比特币测试网上创建完整的 P2PKH 交易。

**步骤：**

1. 使用你在第 1 课中的测试网私钥
2. 在测试网浏览器上检查你的地址余额
3. 用你的 UTXO 详情修改示例代码
4. 构建并签名交易
5. 将其广播到测试网
6. 在浏览器上验证确认

**交付成果：**
- 交易十六进制（已签名）
- 广播后的交易 ID
- 测试网浏览器上的交易链接

### 任务 2：栈执行追踪

**挑战**：给定这个 P2PKH 锁定脚本：
```
76a914c5b28d6bba91a2693a9b1876bcd3929323890fb288ac
```

和这个解锁脚本：
```
47304402...443d01 21028987...8519
```

**问题：**
1. 组合脚本中有多少个操作？
2. 执行期间的最大栈深度是多少？
3. 签名验证发生在哪个步骤？
4. 什么会导致脚本在步骤 6（OP_EQUALVERIFY）失败？

### 任务 3：UTXO 追踪练习

**场景**：Alice 有一个价值 0.05 BTC 的 UTXO。她想向 Bob 发送 0.03 BTC 并支付 0.0001 BTC 的手续费。

**问题：**
1. Alice 将消费多少个 UTXO？
2. 将创建多少个新 UTXO？
3. Alice 的找零输出价值是多少？
4. 如果 Alice 稍后想花费 0.025 BTC，她可以使用哪个（些）UTXO？

---

## 📘 阅读材料

### 必读材料
- **Mastering Taproot** - 第 2 章：Bitcoin Script 基础（第 12-25 页）

### 补充资源
- **Bitcoin Wiki**: Script - https://en.bitcoin.it/wiki/Script
- **Bitcoin Optech**: P2PKH 深度解析
- **交互式脚本调试器**: https://siminchen.github.io/bitcoinIDE/build/editor.html

### 视频和教程
- "Bitcoin Script 如何工作" - Chaincode Labs
- "UTXO vs 账户模型" - Andreas Antonopoulos

---

## 💬 讨论话题

### 问题 1：UTXO 模型设计选择
**"为什么中本聪选择 UTXO 模型而不是账户模型？对于去中心化系统，在并行验证和状态管理方面有什么具体优势？"**

*提示：思考验证者如何同时检查多个交易。*

### 问题 2：基于栈的执行
**"Bitcoin Script 故意不是图灵完备的（没有循环）。为什么这是一个安全特性而不是限制？它防止了什么攻击？"**

*提示：考虑"停机问题"和 DoS 攻击向量。*

### 问题 3：P2PKH vs 未来改进
**"在 P2PKH 中，花费时会暴露公钥。这与 Taproot 的方法相比如何？在后量子世界中，早期暴露公钥可能是什么安全担忧？"**

---

## 🎙️ 讲师笔记

恭喜！你现在已经理解了比特币可编程货币系统的基础。

基于栈的执行模型起初可能看起来很陌生——特别是如果你习惯于传统编程语言的话。但这种简单性是比特币的超能力。每个操作都是可预测的、确定性的和可审计的。没有隐藏状态，没有复杂的内存管理，没有无限循环。

P2PKH 是 Bitcoin Script 的"Hello World"。它很简单，但它教会了你需要了解的关于锁定和解锁脚本如何交互的一切。当我们在后面的课程中学习 Taproot 时，你将看到这个相同的基于栈的模型如何实现更复杂的支出条件——同时保持相同的安全保证。

关键洞察：**Bitcoin Script 是关于证明条件，而不是执行复杂程序**。你不是在"运行代码"——你是在提供密码学证明，证明你满足支出条件。

下周，我们将探索 P2SH（支付到脚本哈希），它引入了一个强大的创新：在简单哈希后面隐藏复杂脚本。这是 Taproot 隐私功能的直接祖先。

继续尝试栈执行示例——可视化栈对于以后调试真实的 Taproot 脚本至关重要！

—Aaron

---

## 📌 关键要点

✅ UTXO 就像数字"钞票"——必须完全花费  
✅ Bitcoin Script 基于栈，简单且确定  
✅ P2PKH = "证明你拥有这个公钥哈希的私钥"  
✅ ScriptSig（解锁）+ ScriptPubKey（锁定）= 组合执行  
✅ 栈执行是可视化和可追踪的——对调试至关重要  
✅ 理解 P2PKH 为你准备 Taproot 的多路径支出

---

**上一课**：[← 第 1 课：密钥和地址](./lesson-01-keys-and-addresses.md)  
**下一课**：[第 3 课：P2SH 多重签名和时间锁 →](./lesson-03-p2sh-multisig.md)

---

*本课程是 Chaincode Labs (2026) Taproot 脚本工程队列的一部分*

