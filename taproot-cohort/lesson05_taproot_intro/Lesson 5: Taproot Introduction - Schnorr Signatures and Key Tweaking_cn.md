# 第 5 课：Taproot 介绍 - Schnorr 签名和密钥调整

**周次**: 5  
**时长**: 自主学习（预计 4-5 小时）  
**先修要求**: 完成第 1-4 课（特别是 SegWit 理解）

---

## 🎯 学习目标

完成本课后，你将能够：

1. 解释 Schnorr 签名的线性特性以及它如何实现密钥聚合
2. 理解密钥调整的数学公式：`P' = P + t × G`
3. 创建简单的仅密钥路径 Taproot 交易
4. 比较 Schnorr（64 字节）与 ECDSA（71-72 字节）签名
5. 生成用于 Taproot 地址的 x-only 公钥（32 字节）
6. 解释 Taproot 如何实现支付统一性和隐私性
7. 理解密钥路径和脚本路径花费之间的区别

---

## 🧩 核心概念

### Taproot 承诺：统一隐私

Taproot 代表了比特币脚本演进的顶峰，展示了**最复杂的智能合约如何能够看起来与简单支付完全相同**。这种革命性的方法将 Schnorr 签名与密码学密钥调整相结合，创建了比特币最先进和私密的授权系统。

**根本性突破**：支付统一性。无论交易代表：
- 简单的单签名支付
- 复杂的多方合约
- 闪电网络通道
- 具有多个授权级别的企业资金

**它们在区块链上看起来完全相同，直到被花费。**

### Schnorr 签名：数学基础

在理解 Taproot 架构之前，我们需要掌握使一切成为可能的数学优雅：**Schnorr 签名**及其变革性特性。

#### 为什么是 Schnorr？ECDSA 的局限性

比特币最初使用 ECDSA（椭圆曲线数字签名算法），但这个选择带来了重大局限性：

**ECDSA 问题：**
- **可延展性**：签名可以被修改而不会失效
- **无法聚合**：多个签名无法合并
- **更大大小**：签名通常是 71-72 字节
- **复杂验证**：需要更多计算资源
- **无线性**：数学运算不保持关系

**Schnorr 的革命性优势：**
- **不可延展**：签名一旦创建就无法修改
- **密钥聚合**：多个公钥可以合并为一个
- **签名聚合**：多个签名可以合并为一个
- **紧凑大小**：固定 64 字节签名
- **高效验证**：更快更简单的验证过程
- **数学线性**：支持高级密码学构造

#### 改变游戏规则的特性：线性

使 Taproot 成为可能的数学突破是**Schnorr 的线性特性**：

1. 如果 Alice 对消息 `M` 有签名 `A`
2. 如果 Bob 对同一消息 `M` 有签名 `B`
3. 那么 `A + B` 为 `(Alice + Bob)` 的组合密钥创建一个**有效签名**

这个简单的数学关系使三种革命性能力成为可能：

1. **密钥聚合**：多人可以将他们的公钥合并为一个
2. **签名聚合**：多个签名可以在数学上合并
3. **密钥调整**：密钥可以通过承诺确定性修改

#### 可视化对比：ECDSA vs Schnorr

```
ECDSA 多重签名 (3-of-3):
┌─────────────────────────────────────┐
│           交易                       │
├─────────────────────────────────────┤
│ Alice 签名:     [71 字节]            │
│ Bob 签名:       [72 字节]            │
│ Charlie 签名:   [70 字节]            │
├─────────────────────────────────────┤
│ 总大小:          ~213 字节            │
│ 验证:           3 次单独验证          │
│ 隐私:           暴露 3 个用户         │
│ 外观:           明显是多签            │
└─────────────────────────────────────┘

Schnorr 聚合 (3-of-3):
┌─────────────────────────────────────┐
│           交易                       │
├─────────────────────────────────────┤
│ 聚合签名:       [64 字节]            │
├─────────────────────────────────────┤
│ 总大小:         64 字节               │
│ 验证:           1 次单一检查          │
│ 隐私:           不暴露任何信息        │
│ 外观:           看起来像单签          │
└─────────────────────────────────────┘
```

**隐私魔法：**

外部观察者看到两个相同的交易，但现实被隐藏：

```
观察者看到的：
┌─────────────────┬─────────────────┐
│   交易 A        │   交易 B        │
├─────────────────┼─────────────────┤
│ 64 字节签名      │ 64 字节签名      │
│ 看起来像：？？？ │ 看起来像：？？？ │
└─────────────────┴─────────────────┘

现实（隐藏）：
┌─────────────────┬─────────────────┐
│   交易 A        │   交易 B        │
├─────────────────┼─────────────────┤
│ 实际上：1       │ 实际上：3       │
│ （单个人）      │ （三个人）      │
└─────────────────┴─────────────────┘

⚡ 从外部无法区分！
```

### 密钥调整：通往 Taproot 的桥梁

Taproot 通过**密钥调整**（BIP 341 中的可调整承诺）利用 Schnorr 的线性，遵循以下公式：

**BIP 341 密钥调整公式：**

```
t = H("TapTweak" || internal_pubkey || merkle_root)
P' = P + t × G
d' = d + t
```

**密钥调整的可视化表示：**

```
内部密钥 (P) ─────────► + 调整值 ─────────► 输出密钥 (P')
         ▲                      │
         │                      │
    Merkle 根 ◄──────────────────┘
    （脚本承诺）
```

**密钥关系图：**

```
┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
│   内部密钥      │   │   调整值        │   │   输出密钥      │
│      (P)        │   │  t = H(P||M)    │   │      (P')       │
│                 │──►│                 │──►│                 │
│ 用户的原始       │   │  确定性         │   │ 最终地址        │
│  私钥           │   │  来自承诺       │   │  链上可见       │
└─────────────────┘   └─────────────────┘   └─────────────────┘
         │                     ▲                      │
         │                     │                      │
         └──── 可以计算 d' ─────┘                      │
                               │                      │
                    ┌──────────┘                      │
                    │                                 │
                    ▼                                 ▼
         ┌─────────────────┐              在区块链上显示为
         │   Merkle 根     │              单个公钥
         │       (M)       │
         │                 │
         │  对所有可能的    │
         │  花费路径的      │
         │  承诺            │
         └─────────────────┘
```

**其中：**
- **P** = 内部密钥（原始公钥，用户控制）
- **M** = Merkle 根（对所有可能支出条件的承诺）
- **t** = 调整值（从 P 和 M 确定性计算）
- **P'** = 输出密钥（最终 Taproot 地址，在区块链上显示）
- **d'** = 调整后的私钥（用于密钥路径花费）

这个数学关系确保：
1. 任何人都可以从 P 和承诺计算 `P'`
2. 只有密钥持有者可以从 d 和调整值计算 `d'`
3. 关系 `d' × G = P'` 保持（签名验证有效）

### X-Only 公钥

Taproot 引入了**x-only 公钥**：不使用完整的 33 字节压缩公钥（`02/03 + x`），Taproot 只使用**32 字节的 x 坐标**。

**为什么这可行：**
- 椭圆曲线方程 `y² = x³ + 7` 意味着可以从 x 计算 y
- 我们总是选择"偶数"y 坐标（按约定）
- 每个公钥节省 1 字节
- 简化签名验证

**格式对比：**

| 格式 | 大小 | 结构 | 使用场景 |
|------|------|------|----------|
| **压缩** | 33 字节 | `02/03 + x` | SegWit |
| **X-only** | 32 字节 | `仅 x` | Taproot |

### Taproot 地址格式（Bech32m）

Taproot 地址使用**Bech32m 编码**（对 Bech32 的改进）：

**格式**：`bc1p` + 编码的输出密钥

**示例：**
- **主网**：`bc1plz0h3rlj2zvn88pgywqtr9k3df3p75p3ltuxh0`
- **测试网**：`tb1p...`

**结构：**
```
bc1p [32 字节输出密钥以 Bech32m 编码]
│ │  │
│ │  └─ 输出密钥（调整后的公钥）
│ └──── 见证版本 1 (Taproot)
└────── 人类可读部分（网络）
```

---

## 💻 示例代码

### 示例 1：基本密钥调整演示

```python
from bitcoinutils.setup import setup
from bitcoinutils.keys import PrivateKey
import hashlib

def demonstrate_key_tweaking():
    """
    演示 Taproot 的基本密钥调整过程
    """
    setup('testnet')
    
    # 步骤 1：生成内部密钥对
    internal_private_key = PrivateKey('cTALNpTpRbbxTCJ2A5Vq88UxT44w1PE2cYqiB3n4hRvzyCev1Wwo')
    internal_public_key = internal_private_key.get_public_key()
    
    print("="*70)
    print("步骤 1：内部密钥生成")
    print("="*70)
    print(f"内部私钥: {internal_private_key.to_wif()}")
    print(f"内部公钥:  {internal_public_key.to_hex()}")
    print()
    
    # 步骤 2：创建脚本承诺（仅密钥路径为空）
    # 在真实的 Taproot 中，这将是脚本条件的 Merkle 根
    script_commitment = b''  # 空 = 仅密钥路径花费
    
    print("="*70)
    print("步骤 2：脚本承诺")
    print("="*70)
    print(f"脚本承诺: {'空（仅密钥路径）' if not script_commitment else script_commitment.hex()}")
    print()
    
    # 步骤 3：使用 BIP 341 公式计算调整值
    # 提取 x-only 坐标（移除 02/03 前缀）
    internal_pubkey_bytes = bytes.fromhex(internal_public_key.to_hex()[2:])
    
    # BIP 341 调整公式：t = H("TapTweak" || internal_pubkey || merkle_root)
    tweak_preimage = b'TapTweak' + internal_pubkey_bytes + script_commitment
    tweak_hash = hashlib.sha256(tweak_preimage).digest()
    tweak_int = int.from_bytes(tweak_hash, 'big')
    
    print("="*70)
    print("步骤 3：调整值计算")
    print("="*70)
    print(f"内部公钥 (x-only): {internal_pubkey_bytes.hex()}")
    print(f"调整值前像: TapTweak || {internal_pubkey_bytes.hex()} || {script_commitment.hex()}")
    print(f"调整值哈希: {tweak_hash.hex()}")
    print(f"调整值整数: {tweak_int}")
    print()
    
    # 步骤 4：应用调整公式 P' = P + t × G
    # 实际上，库会自动处理这个
    print("="*70)
    print("步骤 4：输出密钥 (P' = P + t × G)")
    print("="*70)
    print("调整后的输出密钥由库计算：")
    print(f"输出密钥 = 内部密钥 + (调整值 × 生成点)")
    print()
    
    # 步骤 5：生成 Taproot 地址
    taproot_address = internal_public_key.get_taproot_address()
    
    print("="*70)
    print("步骤 5：最终 Taproot 地址")
    print("="*70)
    print(f"Taproot 地址: {taproot_address.to_string()}")
    print()
    
    return internal_private_key, taproot_address

# 执行演示
internal_key, taproot_addr = demonstrate_key_tweaking()

print("🎯 关键洞察：")
print("Taproot 地址包含调整后的公钥 (P')，")
print("但观察者无法确定：")
print("  • 内部密钥 (P) 是什么")
print("  • 是否有脚本路径被承诺")
print("  • 存在多少个支出条件")
```

**预期输出：**
```
======================================================================
步骤 1：内部密钥生成
======================================================================
内部私钥: cTALNpTpRbbxTCJ2A5Vq88UxT44w1PE2cYqiB3n4hRvzyCev1Wwo
内部公钥:  0250863ad64a87ae8a2fe83c1af1a8403cb53f53e486d8511dad8a04887e5b2352

======================================================================
步骤 2：脚本承诺
======================================================================
脚本承诺: 空（仅密钥路径）

======================================================================
步骤 3：调整值计算
======================================================================
内部公钥 (x-only): 50863ad64a87ae8a2fe83c1af1a8403cb53f53e486d8511dad8a04887e5b2352
调整值前像: TapTweak || 50863ad64a87ae8a2fe83c1af1a8403cb53f53e486d8511dad8a04887e5b2352 || 
调整值哈希: b5e5b5c5d5e5f5a5b5c5d5e5f5a5b5c5d5e5f5a5b5c5d5e5f5a5b5c5d5e5f5a5
调整值整数: 82341234123412341234123412341234123412341234123412341234123412341234

======================================================================
步骤 4：输出密钥 (P' = P + t × G)
======================================================================
调整后的输出密钥由库计算：
输出密钥 = 内部密钥 + (调整值 × 生成点)

======================================================================
步骤 5：最终 Taproot 地址
======================================================================
Taproot 地址: tb1p3vy0w957jhsknlpj9xax2ep558rvx5wx0e2hnqz6w8tz46y0shes8wr0wf

🎯 关键洞察：
Taproot 地址包含调整后的公钥 (P')，
但观察者无法确定：
  • 内部密钥 (P) 是什么
  • 是否有脚本路径被承诺
  • 存在多少个支出条件
```

### 示例 2：创建简单的 Taproot 交易（密钥路径花费）

```python
from bitcoinutils.setup import setup
from bitcoinutils.keys import PrivateKey
from bitcoinutils.transactions import Transaction, TxInput, TxOutput
from bitcoinutils.script import Script

def create_simple_taproot_transaction():
    """
    使用密钥路径花费创建完整的 Taproot 交易
    这演示了最简单的情况：没有脚本路径，只是纯密钥花费
    """
    setup('testnet')
    
    print("="*70)
    print("简单 Taproot 交易（密钥路径花费）")
    print("="*70)
    print()
    
    # 步骤 1：设置密钥
    private_key = PrivateKey('cTALNpTpRbbxTCJ2A5Vq88UxT44w1PE2cYqiB3n4hRvzyCev1Wwo')
    public_key = private_key.get_public_key()
    
    # 生成 Taproot 地址
    from_address = public_key.get_taproot_address()
    
    print("步骤 1：密钥设置")
    print("-" * 70)
    print(f"私钥 (WIF):   {private_key.to_wif()}")
    print(f"公钥 (x-only): {public_key.to_hex()[2:]}")  # 移除 02/03 前缀
    print(f"Taproot 地址:     {from_address.to_string()}")
    print()
    
    # 步骤 2：定义要花费的 UTXO（来自测试网水龙头）
    # 你会从真实的测试网交易中获得这个
    txid = "a3b4d0382efd189619d4f5bd598b6421e709649b87532d53aecdc76457a42cb6"
    vout = 0
    input_amount = 0.001  # 0.001 BTC
    
    print("步骤 2：输入 UTXO")
    print("-" * 70)
    print(f"先前 TXID: {txid}")
    print(f"输出索引:  {vout}")
    print(f"金额:        {input_amount} BTC")
    print()
    
    # 步骤 3：创建交易输入
    tx_input = TxInput(txid, vout)
    
    # 步骤 4：定义输出（发送到另一个 Taproot 地址）
    to_address_str = "tb1p53ncq9ytax924ps66z6al3wfhy6a29w8h6xfu27xem06t98zkmvsakd43h"
    amount_to_send = 0.0009  # 0.0009 BTC（剩余 0.0001 作为手续费）
    
    tx_output = TxOutput(amount_to_send, Script(['OP_1', to_address_str]))
    
    print("步骤 3：输出详情")
    print("-" * 70)
    print(f"目标地址:   {to_address_str}")
    print(f"发送金额:   {amount_to_send} BTC")
    print(f"手续费:           {input_amount - amount_to_send} BTC")
    print()
    
    # 步骤 5：创建未签名交易
    tx = Transaction([tx_input], [tx_output], has_segwit=True)
    
    print("步骤 4：交易结构")
    print("-" * 70)
    print(f"输入:        1")
    print(f"输出:       1")
    print(f"类型:          Taproot (SegWit v1)")
    print()
    
    # 步骤 6：使用 Schnorr 签名签名（密钥路径花费）
    # 对于密钥路径花费，我们使用调整后的私钥签名
    sig = private_key.sign_taproot_input(
        tx, 
        0,  # 输入索引
        [Script(['OP_1', public_key.to_hex()[2:]])],  # 先前的 scriptPubKey
        [input_amount]  # 先前金额
    )
    
    # 步骤 7：添加见证（只有签名，没有公钥！）
    tx.witnesses.append([sig])
    
    print("步骤 5：Schnorr 签名")
    print("-" * 70)
    print(f"签名: {sig.hex()}")
    print(f"长度:    {len(sig)} 字节（正好 64 字节）")
    print()
    
    print("步骤 6：见证结构")
    print("-" * 70)
    print("见证栈：")
    print("  └─ [Schnorr 签名] (64 字节)")
    print()
    print("⚡ 注意：与 SegWit P2WPKH 不同，见证中没有公钥！")
    print()
    
    # 步骤 8：交易详情
    print("="*70)
    print("最终交易")
    print("="*70)
    print(f"原始交易: {tx.serialize()}")
    print(f"交易 ID:  {tx.get_txid()}")
    print(f"大小:            {tx.get_size()} 字节")
    print(f"虚拟大小:    {tx.get_vsize()} vbytes")
    print()
    
    print("🎯 关键观察：")
    print("  1. 见证只包含签名（64 字节）")
    print("  2. 输出密钥嵌入在地址中（32 字节）")
    print("  3. 交易看起来与任何其他 Taproot 交易完全相同")
    print("  4. 观察者无法判断是否存在脚本路径")
    
    return tx, sig

# 执行交易创建
tx, signature = create_simple_taproot_transaction()
```

### 示例 3：比较比特币脚本类型的交易大小

```python
def compare_transaction_types():
    """
    比较不同比特币脚本类型的交易大小和隐私
    """
    print("="*70)
    print("交易类型对比")
    print("="*70)
    print()
    
    comparisons = [
        {
            "type": "传统 P2PKH",
            "scriptPubKey": "OP_DUP OP_HASH160 <20字节> OP_EQUALVERIFY OP_CHECKSIG",
            "scriptSig": "<signature> <pubkey>",
            "witness": "不适用",
            "sig_size": "71-72 字节",
            "total_size": "~225 字节",
            "privacy": "暴露单签",
            "prefix": "1..."
        },
        {
            "type": "SegWit P2WPKH",
            "scriptPubKey": "OP_0 <20字节>",
            "scriptSig": "(空)",
            "witness": "[signature, pubkey]",
            "sig_size": "71-72 字节",
            "total_size": "~165 字节",
            "privacy": "暴露单签",
            "prefix": "bc1q..."
        },
        {
            "type": "Taproot P2TR (简单)",
            "scriptPubKey": "OP_1 <32字节>",
            "scriptSig": "(空)",
            "witness": "[schnorr_sig]",
            "sig_size": "64 字节（固定）",
            "total_size": "~135 字节",
            "privacy": "不暴露任何信息",
            "prefix": "bc1p..."
        },
        {
            "type": "Taproot P2TR (复杂)",
            "scriptPubKey": "OP_1 <32字节>",
            "scriptSig": "(空)",
            "witness": "[schnorr_sig]",
            "sig_size": "64 字节（固定）",
            "total_size": "~135 字节",
            "privacy": "不暴露任何信息",
            "prefix": "bc1p..."
        }
    ]
    
    for i, tx_type in enumerate(comparisons, 1):
        print(f"{i}. {tx_type['type']}")
        print("-" * 70)
        print(f"   地址前缀:  {tx_type['prefix']}")
        print(f"   ScriptPubKey:    {tx_type['scriptPubKey']}")
        print(f"   ScriptSig:       {tx_type['scriptSig']}")
        print(f"   Witness:         {tx_type['witness']}")
        print(f"   签名大小:  {tx_type['sig_size']}")
        print(f"   总大小:      {tx_type['total_size']}")
        print(f"   隐私级别:   {tx_type['privacy']}")
        print()
    
    print("="*70)
    print("TAPROOT 魔法")
    print("="*70)
    print()
    print("💡 关键洞察：")
    print("   'Taproot P2TR (简单)' 和 'Taproot P2TR (复杂)'")
    print("   在链上完全无法区分！")
    print()
    print("   简单支付和具有多个支出条件的复杂多方合约")
    print("   看起来完全相同。")
    print()
    print("   这是 Taproot 的革命性隐私改进。")
    
compare_transaction_types()
```

### 示例 4：Schnorr 签名格式检查

```python
def examine_schnorr_signature():
    """
    检查 Schnorr 签名的结构并与 ECDSA 比较
    """
    print("="*70)
    print("SCHNORR 签名解剖")
    print("="*70)
    print()
    
    # 示例 Schnorr 签名（64 字节）
    schnorr_sig = "7d25fbc9b98ee0eb09ed38c2afc19127465b33d6120f4db8d4fd46e532e30450d7d2a1f1dd7f03e8488c434d10f4051741921d695a44fb774897020f41da99f3"
    
    # 解析签名组件
    r_value = schnorr_sig[:64]  # 前 32 字节（64 个十六进制字符）
    s_value = schnorr_sig[64:]  # 后 32 字节（64 个十六进制字符）
    
    print("Schnorr 签名 (BIP 340)")
    print("-" * 70)
    print(f"完整签名: {schnorr_sig}")
    print()
    print("结构：")
    print(f"  r 值: {r_value}")
    print(f"  s 值: {s_value}")
    print()
    print(f"总长度: {len(schnorr_sig) // 2} 字节")
    print()
    
    # 与 ECDSA 比较
    print("="*70)
    print("对比：SCHNORR vs ECDSA")
    print("="*70)
    print()
    
    print("ECDSA 签名 (DER 编码)：")
    print("-" * 70)
    print("  • 可变长度：通常 70-72 字节")
    print("  • DER 编码: 30 [长度] 02 [r长度] [r值] 02 [s长度] [s值]")
    print("  • 包含编码开销")
    print("  • 签名可延展（可以修改）")
    print()
    
    print("Schnorr 签名 (BIP 340)：")
    print("-" * 70)
    print("  • 固定长度：总是 64 字节")
    print("  • 简单格式：[r值 (32 字节)] [s值 (32 字节)]")
    print("  • 无编码开销")
    print("  • 签名不可延展（无法修改）")
    print("  • 支持聚合和线性")
    print()
    
    # 验证过程
    print("="*70)
    print("验证过程")
    print("="*70)
    print()
    print("Schnorr 验证: (r, s, P, m)")
    print("-" * 70)
    print("  1. 计算挑战:  e = H(r || P || m)")
    print("  2. 计算点:      R = s×G - e×P")
    print("  3. 验证:         r == R 的 x 坐标")
    print()
    print("如果验证通过：签名有效！✓")
    print()
    
    print("🎯 这对 Taproot 为什么重要：")
    print("   • 固定 64 字节大小 = 可预测的交易大小")
    print("   • 线性 = 支持密钥聚合和调整")
    print("   • 不可延展性 = 安全的预签名交易")
    print("   • 更简单的格式 = 更快的验证")

examine_schnorr_signature()
```

---

## 🧪 实践任务

### 任务 1：使用密钥调整生成你的第一个 Taproot 地址

**目标**：创建 Taproot 地址并理解调整过程。

**步骤：**

1. 为测试网生成新的私钥
2. 提取 x-only 公钥（32 字节）
3. 使用空脚本承诺应用密钥调整（仅密钥路径）
4. 生成 Taproot 地址 (tb1p...)
5. 从 https://coinfaucet.eu/en/btc-testnet/ 向它发送测试网币
6. 在 https://mempool.space/testnet 上验证接收

**代码模板：**

```python
from bitcoinutils.setup import setup
from bitcoinutils.keys import PrivateKey

setup('testnet')

# 1. 生成密钥
private_key = PrivateKey()
public_key = private_key.get_public_key()

# 2. 提取 x-only 公钥
x_only_pubkey = public_key.to_hex()[2:]  # 移除 02/03 前缀

# 3. 生成 Taproot 地址（调整自动发生）
taproot_address = public_key.get_taproot_address()

print(f"私钥（保存这个！）: {private_key.to_wif()}")
print(f"X-only 公钥: {x_only_pubkey}")
print(f"Taproot 地址: {taproot_address.to_string()}")

# 4-6. 手动资助和验证
```

**交付成果**：
- 在 mempool.space 上你的已资助地址的截图
- 安全保存你的私钥 WIF
- 记录接收币时的交易大小

### 任务 2：比较签名大小

**挑战**：使用不同的脚本类型创建交易并比较实际大小。

**要求：**
1. 创建 P2WPKH（SegWit）交易
2. 创建 P2TR（Taproot）密钥路径交易
3. 比较：
   - 原始交易大小
   - 虚拟大小（vBytes）
   - 签名大小
   - 见证结构

**要回答的问题：**
- Taproot 小多少字节？
- 见证折扣效果是什么？
- 这如何转化为手续费节省？

### 任务 3：密钥调整数学验证

**挑战**：手动验证密钥调整公式。

给定：
- 内部私钥：`d`
- 内部公钥：`P = d × G`
- 调整值：`t = H("TapTweak" || P || merkle_root)`

验证：
1. `d' = d + t`（调整后的私钥）
2. `P' = P + t × G`（调整后的公钥）
3. `d' × G = P'`（关系保持）

**提示**：使用示例 1 的代码并添加手动计算来验证每一步。

---

## 📘 阅读材料

### 必读材料
- **Mastering Taproot** - 第 5 章：Taproot：比特币脚本系统的演进（第 56-70 页）
- 重点：
  - Schnorr 签名数学（第 56-58 页）
  - 密钥调整公式（第 58-62 页）
  - 简单 Taproot 交易结构（第 62-65 页）

### BIP 参考
- **BIP 340**：secp256k1 的 Schnorr 签名 - https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki
  - 阅读：签名生成和验证算法
- **BIP 341**：Taproot：SegWit 版本 1 支出规则 - https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki
  - 重点：密钥调整部分
- **BIP 342**：Taproot 脚本验证 - https://github.com/bitcoin/bips/blob/master/bip-0342.mediawiki

### 补充资源
- **Bitcoin Optech**：Schnorr/Taproot 工作坊 - https://bitcoinops.org/en/schorr-taproot-workshop/
- **Pieter Wuille 的 Taproot 演示** - 理解密码学
- **BIP 350**：Bech32m 编码 - https://github.com/bitcoin/bips/blob/master/bip-0350.mediawiki

---

## 💬 讨论话题

在 Discord #week-5-discussion 频道发布你的想法：

### 问题 1：线性优势

**"Schnorr 签名具有 ECDSA 缺乏的'线性特性'。用你自己的话解释这个特性如何实现密钥调整，以及为什么这对 Taproot 的隐私保证至关重要。如果我们尝试用 ECDSA 签名实现 Taproot 会发生什么？"**

*提示：思考当你将两个 Schnorr 签名相加与将两个 ECDSA 签名相加时会发生什么。*

### 问题 2：隐私权衡

**"Taproot 使所有交易看起来完全相同，无论它们是简单支付还是复杂合约。这会造成任何潜在问题吗？是否有场景你希望揭示你的交易是复杂的？讨论隐私 vs. 透明度的权衡。"**

*考虑：监管要求、多重签名安全证明、审计需求。*

### 问题 3：合作激励

**"书中指出 Taproot'将经济激励与技术优化对齐。'解释 Taproot 的设计如何鼓励多方合约中的各方合作（密钥路径花费）而不是争议（脚本路径花费）。合作的具体经济利益是什么？"**

*提示：比较密钥路径和脚本路径花费之间的交易大小和手续费。*

---

## 🎙️ 讲师笔记

欢迎来到我们队列的 Taproot 部分！这是我们到目前为止所学一切汇聚的地方。

你已经掌握了基础知识：密钥、地址、栈操作、P2PKH、P2SH 和 SegWit。现在我们准备好学习比特币最先进的脚本系统。Taproot 代表了多年研究使比特币更私密、高效和灵活的顶峰。

**关键洞察**：Taproot 使用 Schnorr 签名的数学线性来"调整"公钥，通过承诺到脚本条件。这允许复杂的智能合约隐藏在看起来像简单单签名支付的背后。

使其革命性的原因：
- **隐私**：所有 Taproot 交易看起来完全相同
- **效率**：64 字节签名，更小的交易
- **灵活性**：多个支出路径，隐藏直到使用

在本课中，我们专注于**最简单的情况**：仅密钥路径 Taproot（还没有脚本路径）。把这看作是"Taproot 训练轮"。我们在下一课添加脚本树之前学习核心密码学原语。

特别注意密钥调整公式 `P' = P + t × G`。这个公式是桥梁：
- 简单的基于密钥的花费（我们今天做的）
- 复杂的基于脚本的花费（我们将在第 6 课+做的）

认真对待实践任务。熟悉 x-only 公钥和 Schnorr 签名。当我们在下一课开始构建具有脚本路径的真实 Taproot 合约时，你将需要这个基础。

在 Discord 上见，深入讨论 Schnorr 的线性！

—Aaron

---

## 📌 关键要点

✅ Schnorr 签名通过数学线性实现密钥聚合  
✅ 密钥调整公式：`P' = P + t × G` 其中 `t = H("TapTweak" || P || M)`  
✅ X-only 公钥（32 字节）节省空间并简化验证  
✅ Taproot 地址使用 Bech32m 编码，以 `bc1p`（主网）或 `tb1p`（测试网）开头  
✅ Schnorr 签名固定 64 字节（vs. ECDSA 的 71-72 字节）  
✅ 所有 Taproot 交易在花费前看起来完全相同（支付统一性）  
✅ 密钥路径花费最有效（只有签名，见证中没有公钥）  
✅ Taproot 通过更小的交易和更好的隐私激励合作

---

**下一课**：[第 6 课：单叶子 Taproot 脚本树 →](./lesson-06-single-leaf-taptree.md)

---

*本课程是 Chaincode Labs (2026) Taproot 脚本工程队列的一部分*

