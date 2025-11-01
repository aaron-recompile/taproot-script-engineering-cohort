# 第 3 课：P2SH 多重签名和时间锁

**周次**: 3  
**时长**: 自主学习（预计 5-6 小时）  
**先修要求**: 第 1-2 课（密钥、地址、P2PKH）

---

## 🎯 学习目标

完成本课后，你将能够：

1. 理解 P2SH 的两阶段执行模型（哈希验证 → 脚本执行）
2. 构建 2-of-3 多重签名资金合约
3. 实现 CSV（CheckSequenceVerify）时间锁定继承脚本
4. 追踪 P2SH 栈执行，包含栈重置机制
5. 认识 P2SH 的局限性以及 Taproot 如何解决它们
6. 在测试网上构建和广播真实的 P2SH 交易

---

## 🧩 核心概念

### P2SH 架构：哈希背后的脚本

P2SH（支付到脚本哈希）通过使**复杂脚本隐藏在简单地址后面**彻底改变了比特币。

**核心创新：**
```
替代：
  OP_2 <pk1> <pk2> <pk3> OP_3 OP_CHECKMULTISIG
使用：
  OP_HASH160 <script_hash> OP_EQUAL
```

**P2SH 优势：**
- ✅ **紧凑地址**：复杂脚本 → 20 字节哈希
- ✅ **成本转移**：接收方支付脚本揭示费用，而非发送方
- ✅ **隐私**：脚本在花费前不会暴露
- ✅ **向后兼容**：适用于所有比特币软件

**P2SH 两阶段执行模型：**

```
阶段 1：哈希验证
  ScriptSig + ScriptPubKey
  ↓
  验证: HASH160(redeem_script) == expected_hash
  
阶段 2：脚本执行（如果阶段 1 成功）
  栈重置 → 执行 redeem_script
  ↓
  验证: 赎回脚本条件满足
```

**关键机制：栈重置**

Bitcoin Core 识别 P2SH 模式 `OP_HASH160 <hash> OP_EQUAL` 并触发特殊处理：

1. **阶段 1 完成**，栈顶为 TRUE
2. **栈重置**到 ScriptSig 之后的状态（丢弃 TRUE）
3. **从原始 ScriptSig 中提取**赎回脚本
4. **阶段 2 开始**，使用干净的栈执行赎回脚本

这个两阶段模型是 Taproot 提交-揭示模式的直接祖先！

---

### 多重签名资金：2-of-3 企业安全

**业务场景：初创企业资金**

考虑一个区块链初创企业，有三个利益相关者：
- **Alice（CEO）**：运营权限
- **Bob（CTO）**：技术监督  
- **Carol（CFO）**：财务控制

**策略**：任何资金转移需要 **2-of-3** 签名。

**为什么是 2-of-3？**
- ❌ **不是 3-of-3**：防止一个密钥丢失导致运营瘫痪
- ❌ **不是 1-of-3**：防止单人风险
- ✅ **2-of-3**：平衡安全性和运营灵活性

### 赎回脚本构建

```python
redeem_script = Script([
    'OP_2',              # 需要 2 个签名
    alice_pk,            # Alice 的公钥 (33 字节)
    bob_pk,              # Bob 的公钥 (33 字节)
    carol_pk,            # Carol 的公钥 (33 字节)
    'OP_3',              # 总共 3 个密钥
    'OP_CHECKMULTISIG'   # 多重签名验证操作码
])
```

**序列化格式：**
```
52 21 02898711... 21 0284b595... 21 0317aa89... 53 ae

52 = OP_2
21 = PUSH 33 字节 (压缩公钥)
53 = OP_3
ae = OP_CHECKMULTISIG
```

**P2SH 地址生成：**
```
1. 序列化赎回脚本 → 字节
2. Hash160(redeem_script) → 20 字节哈希
3. 添加版本 (0xc4 测试网，0x05 主网)
4. Base58Check 编码 → 地址以 '2' 开头（测试网）
```

---

### OP_CHECKMULTISIG：多重签名的主力

**栈消费模式：**
```
输入栈：
  OP_0          // Bug 解决方法（额外项）
  <sig1>        // 第一个签名
  <sig2>        // 第二个签名
  OP_2          // 签名数量
  <pk1>         // 第一个公钥
  <pk2>         // 第二个公钥
  <pk3>         // 第三个公钥
  OP_3          // 公钥总数

OP_CHECKMULTISIG 消费：
  - 密钥数量 (3)
  - 所有 3 个公钥
  - 签名数量 (2)
  - 所有 2 个签名
  - 额外项 (OP_0，由于 off-by-one bug)

输出：
  TRUE 或 FALSE
```

**著名的 OP_CHECKMULTISIG Bug：**

比特币的原始实现有一个 off-by-one 错误，会从栈中多弹出一个项。这是**共识代码**，无法在不进行硬分叉的情况下更改。

**解决方案**：对于多重签名脚本，始终在 ScriptSig 的第一项推送 `OP_0`。

---

### CSV 时间锁：相对时间约束

**CSV (CheckSequenceVerify) - BIP 112**

实现**相对时间锁**：花费延迟相对于 UTXO 创建的时间。

**使用场景：**
- 🏦 **数字继承**：继承人在不活动期后访问
- ⚡ **闪电网络**：争议解决的结算延迟
- 🔒 **托管**：时间到期后自动释放

**CSV 脚本模式：**
```
<delay_blocks>
OP_CHECKSEQUENCEVERIFY
OP_DROP
<standard_script>  // 例如，P2PKH
```

**BIP 68 序列号编码：**
```
交易输入中的 nSequence 字段：
  Bit 31:    0 = 启用，1 = 禁用
  Bit 22:    0 = 区块，1 = 512 秒间隔
  Bits 0-15: 相对锁定时间值

示例：3 个区块延迟
  nSequence = 0x00000003
```

---

### P2SH 栈执行：完整追踪

**交易示例：**
```
TXID: e68bef534c7536300c3ae5ccd0f79e031cab29d262380a37269151e8ba
```

**阶段 1：哈希验证**

**初始栈**（空）：
```
┌────────────────────────────────┐
│ (empty)                        │
└────────────────────────────────┘
```

**步骤 1**：PUSH OP_0（多重签名 bug 解决方法）
```
┌────────────────────────────────┐
│ 00                             │
└────────────────────────────────┘
```

**步骤 2**：PUSH Alice 的签名
```
┌────────────────────────────────┐
│ 30440220694f...7a6501 (alice)  │
│ 00                             │
└────────────────────────────────┘
```

**步骤 3**：PUSH Bob 的签名
```
┌────────────────────────────────┐
│ 3044022065f8...fd9e01 (bob)    │
│ 30440220694f...7a6501 (alice)  │
│ 00                             │
└────────────────────────────────┘
```

**步骤 4**：PUSH redeem_script
```
┌────────────────────────────────┐
│ 522102898711...ae (redeem)     │
│ 3044022065f8...fd9e01 (bob)    │
│ 30440220694f...7a6501 (alice)  │
│ 00                             │
└────────────────────────────────┘
```

**步骤 5**：OP_HASH160（哈希赎回脚本）
```
┌────────────────────────────────┐
│ dd81b5beb3d8...ca (computed)   │
│ 3044022065f8...fd9e01 (bob)    │
│ 30440220694f...7a6501 (alice)  │
│ 00                             │
└────────────────────────────────┘
```

**步骤 6**：PUSH expected_hash（来自锁定脚本）
```
┌────────────────────────────────┐
│ dd81b5beb3d8...ca (expected)   │
│ dd81b5beb3d8...ca (computed)   │
│ 3044022065f8...fd9e01 (bob)    │
│ 30440220694f...7a6501 (alice)  │
│ 00                             │
└────────────────────────────────┘
```

**步骤 7**：OP_EQUAL
```
┌────────────────────────────────┐
│ 1 (TRUE)                       │
│ 3044022065f8...fd9e01 (bob)    │
│ 30440220694f...7a6501 (alice)  │
│ 00                             │
└────────────────────────────────┘
```

**✅ 阶段 1 完成：哈希匹配！**

---

**关键：栈重置机制**

Bitcoin Core 检测到 P2SH 模式并：
1. 丢弃阶段 1 的 TRUE
2. 将栈重置到 **ScriptSig 之后**的状态
3. 提取赎回脚本用于阶段 2

**重置后的栈：**
```
┌────────────────────────────────┐
│ 3044022065f8...fd9e01 (bob)    │
│ 30440220694f...7a6501 (alice)  │
│ 00                             │
└────────────────────────────────┘
```

---

**阶段 2：赎回脚本执行**

**赎回脚本：**
```
OP_2 <alice_pk> <bob_pk> <carol_pk> OP_3 OP_CHECKMULTISIG
```

**步骤 8**：OP_2（推送所需签名数量）
```
┌────────────────────────────────┐
│ 2                              │
│ 3044022065f8...fd9e01 (bob)    │
│ 30440220694f...7a6501 (alice)  │
│ 00                             │
└────────────────────────────────┘
```

**步骤 9-11**：PUSH alice_pk, bob_pk, carol_pk
```
┌────────────────────────────────┐
│ 0317aa89... (carol_pk)         │
│ 0284b595... (bob_pk)           │
│ 02898711... (alice_pk)         │
│ 2                              │
│ 3044022065f8...fd9e01 (bob)    │
│ 30440220694f...7a6501 (alice)  │
│ 00                             │
└────────────────────────────────┘
```

**步骤 12**：OP_3（推送总密钥数量）
```
┌────────────────────────────────┐
│ 3                              │
│ 0317aa89... (carol_pk)         │
│ 0284b595... (bob_pk)           │
│ 02898711... (alice_pk)         │
│ 2                              │
│ 3044022065f8...fd9e01 (bob)    │
│ 30440220694f...7a6501 (alice)  │
│ 00                             │
└────────────────────────────────┘
```

**步骤 13**：OP_CHECKMULTISIG

验证过程：
1. ✅ Alice 的签名针对 Alice 的公钥验证
2. ✅ Bob 的签名针对 Bob 的公钥验证
3. ✅ 满足所需阈值（2-of-3）

```
┌────────────────────────────────┐
│ 1 (TRUE)                       │
└────────────────────────────────┘
```

**✅✅ 交易有效：两个阶段都成功！**

---

## 💻 示例代码

### 示例 1：创建 2-of-3 多重签名 P2SH 地址

```python
from bitcoinutils.setup import setup
from bitcoinutils.keys import PrivateKey
from bitcoinutils.script import Script
from bitcoinutils.keys import P2shAddress

def create_multisig_p2sh():
    """
    创建 2-of-3 多重签名 P2SH 地址
    企业资金示例
    """
    setup('testnet')
    
    # 利益相关者公钥
    alice_pk = '02898711e6bf63f5cbe1b38c05e89d6c391c59e9f8f695da44bf3d20ca674c8519'
    bob_pk = '0284b5951609b76619a1ce7f48977b4312ebe226987166ef044bfb374ceef63af5'
    carol_pk = '0317aa89b43f46a0c0cdbd9a302f2508337ba6a06d123854481b52de9c20996011'
    
    print("="*60)
    print("2-OF-3 多重签名 P2SH 地址创建")
    print("="*60)
    print(f"Alice 的公钥:  {alice_pk}")
    print(f"Bob 的公钥:    {bob_pk}")
    print(f"Carol 的公钥:  {carol_pk}")
    print()
    
    # 2-of-3 多重签名赎回脚本
    redeem_script = Script([
        'OP_2',              # 需要 2 个签名
        alice_pk,            # Alice 的公钥
        bob_pk,              # Bob 的公钥
        carol_pk,            # Carol 的公钥
        'OP_3',              # 总共 3 个密钥
        'OP_CHECKMULTISIG'   # 多重签名验证
    ])
    
    print("="*60)
    print("赎回脚本")
    print("="*60)
    print(f"十六进制: {redeem_script.to_hex()}")
    print(f"汇编: {redeem_script.to_string()}")
    print(f"大小: {len(redeem_script.to_bytes())} 字节")
    print()
    
    # 生成 P2SH 地址
    p2sh_addr = P2shAddress.from_script(redeem_script)
    
    print("="*60)
    print("P2SH 地址")
    print("="*60)
    print(f"地址: {p2sh_addr.to_string()}")
    print(f"脚本哈希: {p2sh_addr.to_hash160()}")
    print()
    
    # 锁定脚本（在区块链上出现的）
    locking_script = p2sh_addr.to_script_pub_key()
    print("="*60)
    print("锁定脚本 (ScriptPubKey)")
    print("="*60)
    print(f"十六进制: {locking_script.to_hex()}")
    print(f"汇编: {locking_script.to_string()}")
    print()
    
    print("="*60)
    print("向此地址发送测试网币！")
    print("="*60)
    
    return p2sh_addr, redeem_script

if __name__ == "__main__":
    addr, redeem_script = create_multisig_p2sh()
```

### 示例 2：从 2-of-3 多重签名花费（Alice + Bob 签名）

```python
from bitcoinutils.setup import setup
from bitcoinutils.utils import to_satoshis
from bitcoinutils.transactions import Transaction, TxInput, TxOutput
from bitcoinutils.keys import PrivateKey, P2wpkhAddress
from bitcoinutils.script import Script

def spend_multisig_p2sh():
    """
    从 2-of-3 多重签名 P2SH 花费
    Alice 和 Bob 授权支付
    """
    setup('testnet')
    
    # 私钥（在生产环境中保持安全！）
    alice_sk = PrivateKey('cMahea7zqjxrtgAbB7LSGbcQUr1uX1ojuat9jZodMN87JcbXMTcA')
    bob_sk = PrivateKey('cMahea7zqjxrtgAbB7LSGbcQUr1uX1ojuat9jZodMN87K8CaLbxz')
    
    # 赎回脚本（与创建时相同）
    alice_pk = alice_sk.get_public_key().to_hex()
    bob_pk = bob_sk.get_public_key().to_hex()
    carol_pk = '0317aa89b43f46a0c0cdbd9a302f2508337ba6a06d123854481b52de9c20996011'
    
    redeem_script = Script([
        'OP_2',
        alice_pk,
        bob_pk,
        carol_pk,
        'OP_3',
        'OP_CHECKMULTISIG'
    ])
    
    # 先前的 UTXO 详情（用你的 UTXO 替换！）
    utxo_txid = '4b869865bc4a156d7e0ba14590b5c8971e57b8198af64d88872558ca88a8ba5f'
    utxo_vout = 0
    utxo_amount = 0.00001600  # 1,600 satoshis
    
    # 接收方地址
    recipient_address = P2wpkhAddress('tb1qckeg66a6jx3xjw5mrpmte5ujjv3cjrajtvm9r4')
    
    # 创建交易
    txin = TxInput(utxo_txid, utxo_vout)
    
    # 发送 888 sats，剩余是手续费
    txout = TxOutput(to_satoshis(0.00000888), recipient_address.to_script_pub_key())
    
    tx = Transaction([txin], [txout])
    
    print("="*60)
    print("从多重签名 P2SH 花费")
    print("="*60)
    print(f"输入 UTXO: {utxo_txid}:{utxo_vout}")
    print(f"金额: {utxo_amount} BTC")
    print(f"接收方: {recipient_address.to_string()}")
    print(f"发送: 0.00000888 BTC")
    print(f"手续费: {utxo_amount - 0.00000888} BTC")
    print()
    
    # 使用 Alice 和 Bob 的密钥签名
    print("="*60)
    print("签名阶段")
    print("="*60)
    print("Alice 签名中...")
    alice_sig = alice_sk.sign_input(tx, 0, redeem_script)
    print(f"Alice 签名: {alice_sig[:40]}...")
    
    print("Bob 签名中...")
    bob_sig = bob_sk.sign_input(tx, 0, redeem_script)
    print(f"Bob 签名: {bob_sig[:40]}...")
    print()
    
    # 构造 ScriptSig
    # 顺序很重要：OP_0，签名按匹配公钥的顺序排列
    txin.script_sig = Script([
        'OP_0',              # OP_CHECKMULTISIG bug 解决方法
        alice_sig,           # 第一个签名
        bob_sig,             # 第二个签名
        redeem_script.to_hex()  # 揭示赎回脚本
    ])
    
    print("="*60)
    print("解锁脚本 (ScriptSig)")
    print("="*60)
    print(f"组件:")
    print(f"  1. OP_0 (bug 解决方法)")
    print(f"  2. Alice 的签名 (71 字节)")
    print(f"  3. Bob 的签名 (71 字节)")
    print(f"  4. 赎回脚本 ({len(redeem_script.to_bytes())} 字节)")
    print()
    
    # 获取已签名交易
    signed_tx = tx.serialize()
    
    print("="*60)
    print("已签名交易")
    print("="*60)
    print(signed_tx)
    print()
    print(f"交易大小: {tx.get_size()} 字节")
    print()
    print("准备广播！")
    print("使用: https://mempool.space/testnet/tx/push")
    
    return tx

if __name__ == "__main__":
    tx = spend_multisig_p2sh()
```

### 示例 3：CSV 时间锁定继承

```python
from bitcoinutils.setup import setup
from bitcoinutils.transactions import Sequence
from bitcoinutils.constants import TYPE_RELATIVE_TIMELOCK
from bitcoinutils.script import Script
from bitcoinutils.keys import P2shAddress, PrivateKey

def create_csv_inheritance():
    """
    创建一个 3 个区块时间锁定的继承脚本
    受益人只能在 3 个区块后花费
    """
    setup('testnet')
    
    # 受益人信息
    beneficiary_sk = PrivateKey('cTALNpTpRbbxTCJ2A5Vq88UxT44w1PE2cYqiB3n4hRvzyCev1Wwo')
    beneficiary_pk = beneficiary_sk.get_public_key()
    beneficiary_pkh = beneficiary_pk.to_hash160()
    
    # 3 个区块相对时间锁
    relative_blocks = 3
    seq = Sequence(TYPE_RELATIVE_TIMELOCK, relative_blocks)
    
    print("="*60)
    print("CSV 时间锁定继承脚本")
    print("="*60)
    print(f"延迟: {relative_blocks} 个区块")
    print(f"序列值: 0x{seq.for_script():08x}")
    print(f"受益人: {beneficiary_pk.to_address().to_string()}")
    print()
    
    # 组合 CSV + P2PKH 脚本
    redeem_script = Script([
        seq.for_script(),          # PUSH 3
        'OP_CHECKSEQUENCEVERIFY',  # 验证时间锁
        'OP_DROP',                 # 从栈中移除延迟值
        'OP_DUP',                  # 标准 P2PKH 从这里开始
        'OP_HASH160',
        beneficiary_pkh,
        'OP_EQUALVERIFY',
        'OP_CHECKSIG'
    ])
    
    print("="*60)
    print("赎回脚本分解")
    print("="*60)
    print(f"十六进制: {redeem_script.to_hex()}")
    print()
    print("组件:")
    print(f"  1. PUSH 3 (延迟值)")
    print(f"  2. OP_CHECKSEQUENCEVERIFY")
    print(f"  3. OP_DROP")
    print(f"  4-8. 标准 P2PKH")
    print()
    
    # 生成 P2SH 地址
    p2sh_addr = P2shAddress.from_script(redeem_script)
    
    print("="*60)
    print("P2SH 地址")
    print("="*60)
    print(f"地址: {p2sh_addr.to_string()}")
    print()
    print("⏰ 发送到此地址的资金只能由")
    print("   受益人在 3 个区块后花费！")
    print()
    
    return p2sh_addr, redeem_script, seq

if __name__ == "__main__":
    addr, redeem, seq = create_csv_inheritance()
```

### 示例 4：花费 CSV 时间锁定 UTXO

```python
def spend_csv_inheritance():
    """
    在时间锁过期后从 CSV 锁定的 P2SH 花费
    """
    setup('testnet')
    
    # 获取 CSV 脚本组件
    beneficiary_sk = PrivateKey('cTALNpTpRbbxTCJ2A5Vq88UxT44w1PE2cYqiB3n4hRvzyCev1Wwo')
    beneficiary_pk = beneficiary_sk.get_public_key()
    
    # 必须在 UTXO 创建后等待 3 个区块
    seq = Sequence(TYPE_RELATIVE_TIMELOCK, 3)
    
    # 重新创建赎回脚本
    redeem_script = Script([
        seq.for_script(),
        'OP_CHECKSEQUENCEVERIFY',
        'OP_DROP',
        'OP_DUP',
        'OP_HASH160',
        beneficiary_pk.to_hash160(),
        'OP_EQUALVERIFY',
        'OP_CHECKSIG'
    ])
    
    # UTXO 详情
    utxo_txid = '34f5bf0cf328d77059b5674e71442ded8cdcfc723d0136733e0dbf1808619'
    utxo_vout = 0
    
    # 关键：在交易输入中设置序列号
    txin = TxInput(utxo_txid, utxo_vout, sequence=seq.for_input_sequence())
    
    recipient = P2wpkhAddress('tb1qckeg66a6jx3xjw5mrpmte5ujjv3cjrajtvm9r4')
    txout = TxOutput(to_satoshis(0.00000500), recipient.to_script_pub_key())
    
    tx = Transaction([txin], [txout])
    
    print("="*60)
    print("花费时间锁定的 UTXO")
    print("="*60)
    print(f"序列值: 0x{seq.for_input_sequence():08x}")
    print(f"时间锁: {3} 个区块（必须已过去）")
    print()
    
    # 签名交易
    sig = beneficiary_sk.sign_input(tx, 0, redeem_script)
    
    # 构造 ScriptSig
    txin.script_sig = Script([
        sig,                    # P2PKH 的签名
        beneficiary_pk.to_hex(),  # P2PKH 的公钥
        redeem_script.to_hex()    # 揭示脚本
    ])
    
    signed_tx = tx.serialize()
    
    print("="*60)
    print("已签名交易")
    print("="*60)
    print(signed_tx)
    print()
    print("⚠️  如果少于 3 个区块已过去，这将失败！")
    print("   错误: 'non-BIP68-final'")
    print()
    
    return tx

if __name__ == "__main__":
    tx = spend_csv_inheritance()
```

---

## 🧪 实践任务

### 任务 1：构建你自己的 2-of-3 多重签名资金

**目标**：在测试网上创建完整的 2-of-3 多重签名设置。

**步骤：**

1. 生成三个密钥对（Alice、Bob、Carol）
2. 创建 2-of-3 赎回脚本
3. 生成 P2SH 地址
4. 向该地址发送测试网币
5. 创建由 Alice + Bob 签名的交易
6. 广播并在浏览器上验证

**交付成果：**
- P2SH 地址
- 赎回脚本（十六进制）
- 成功花费后的交易 ID
- 浏览器上交易的截图

**奖励挑战**：尝试使用 Bob + Carol 花费！

### 任务 2：实现 3 个区块时间锁

**场景**：创建一个在 3 个区块后解锁的数字继承脚本。

**要求：**
1. 使用 3 个区块延迟的 CSV
2. 与受益人的 P2PKH 结合
3. 测试交易创建
4. 计算大小所需的最小手续费

**问题**：如果你在 3 个区块之前尝试广播会发生什么？测试一下！

### 任务 3：P2SH 栈执行追踪

**挑战**：给定这个 P2SH 锁定脚本：
```
a914dd81b5beb3d8ed72ea5f19f93d2e0b12145cb0ca87
```

和这个解锁脚本：
```
00 <sig1> <sig2> <redeem_script>
```

**任务：**
1. 识别两个执行阶段
2. 绘制阶段 1 每一步的栈状态
3. 解释栈重置机制
4. 绘制阶段 2 每一步的栈状态
5. 签名验证在哪个步骤执行？

---

## 📘 阅读材料

### 必读材料
- **Mastering Taproot** - 第 3 章：P2SH 脚本工程（第 27-41 页）

### BIP 参考
- **BIP 16**：支付到脚本哈希 - https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki
- **BIP 112**：CHECKSEQUENCEVERIFY - https://github.com/bitcoin/bips/blob/master/bip-0112.mediawiki
- **BIP 68**：使用共识强制序列号的相对锁定时间 - https://github.com/bitcoin/bips/blob/master/bip-0068.mediawiki

### 补充资源
- **Bitcoin Wiki**：多重签名 - https://en.bitcoin.it/wiki/Multisignature
- **Bitcoin Optech**：CheckSequenceVerify - https://bitcoinops.org/en/topics/op_checksequenceverify/
- **Andreas Antonopoulos**："精通比特币" 第 7 章（高级交易）

---

## 💬 讨论话题

### 问题 1：P2SH 隐私权衡
**"P2SH 将脚本复杂性隐藏在哈希后面，但完整脚本在花费时会被揭示。这与 Taproot 的方法相比如何？当只使用复杂多条件脚本的一个分支时，隐私影响是什么？"**

*提示：考虑在花费时链上泄露的信息。*

### 问题 2：栈重置机制
**"为什么 Bitcoin Core 需要在 P2SH 执行的阶段 1 和阶段 2 之间重置栈？如果阶段 1 的 TRUE 在赎回脚本执行期间保留在栈上会发生什么？"**

*提示：思考 OP_CHECKMULTISIG 或其他操作码可能如何受到影响。*

### 问题 3：CSV vs CLTV
**"CSV (CheckSequenceVerify) 使用相对时间，而 CLTV (CheckLockTimeVerify) 使用绝对时间。对于数字继承场景，为什么可能更喜欢 CSV？CLTV 在什么时候更好？"**

*提示：考虑如果资金交易被延迟会发生什么。*

---

## 🎙️ 讲师笔记

恭喜你达到 P2SH 里程碑！这里是 Bitcoin Script 真正强大的地方。

两阶段执行模型——先哈希验证再脚本执行——优雅但微妙。栈重置机制让许多开发者困惑，所以确保你真的理解为什么 Bitcoin Core 丢弃那个 TRUE 并重置到 ScriptSig 之后的状态。这不是任意的；它是必要的，以防止阶段 1 的结果干扰阶段 2 的逻辑。

多重签名示例展示了 P2SH 的优势：复杂的授权要求隐藏在简单的 34 字符地址后面。但请注意限制：**花费时必须揭示赎回脚本的每个细节**，甚至未使用的分支。如果你有 5 个可能的花费条件但只使用一个，所有 5 个都会在链上暴露。

这就是为什么 Taproot 是革命性的。它使用类似的提交-揭示模式（像 P2SH 的哈希承诺），但有一个关键改进：**只揭示执行的路径**。未使用的分支完全隐藏。我们将在第 5 课开始看到这一点。

CSV 时间锁很有趣，因为它们是**相对的**，而不是绝对的。时钟在 UTXO 确认时开始滴答作响，而不是某个预定日期。这使得它们非常适合继承（3 个月不活动）、支付通道（争议解决期），或任何时间相对于交易本身本身的场景。

密切关注 BIP 68 中的序列号编码——它很复杂但对于闪电网络和其他第 2 层协议来说是必不可少的。

下周：SegWit！我们将探索见证隔离如何解决交易可延展性，并为 Taproot 的效率奠定基础。

继续构建！

—Aaron

---

## 📌 关键要点

✅ P2SH 使复杂脚本隐藏在简单地址后面  
✅ 两阶段执行：哈希检查 → 栈重置 → 脚本执行  
✅ OP_CHECKMULTISIG 需要 OP_0 来解决 off-by-one bug  
✅ 2-of-3 多重签名平衡安全性与运营灵活性  
✅ CSV 提供相对时间锁（自 UTXO 创建以来的区块）  
✅ P2SH 局限性：完整脚本揭示，无选择性披露  
✅ Taproot 建立在 P2SH 基础上，但只揭示执行的路径

---

**上一课**：[← 第 2 课：Bitcoin Script 和 P2PKH](./lesson-02-bitcoin-script-p2pkh.md)  
**下一课**：[第 4 课：SegWit 交易和见证结构 →](./lesson-04-segwit-witness.md)

---

*本课程是 Chaincode Labs (2026) Taproot 脚本工程队列的一部分*

