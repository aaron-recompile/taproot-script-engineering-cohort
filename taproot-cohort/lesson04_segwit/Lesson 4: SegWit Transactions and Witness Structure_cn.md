# 第 4 课：SegWit 交易和见证结构

**周次**: 4  
**时长**: 自主学习（预计 5-6 小时）  
**先修要求**: 第 1-3 课（密钥、P2PKH、P2SH）

---

## 🎯 学习目标

完成本课后，你将能够：

1. 解释交易可延展性以及 SegWit 如何解决它
2. 理解见证结构和见证隔离概念
3. 构建完整的 P2WPKH（原生 SegWit）交易
4. 使用见证数据追踪 P2WPKH 栈执行
5. 比较传统、P2SH 包装的 SegWit 和原生 SegWit
6. 计算交易权重单位和手续费优化
7. 认识到 SegWit 是 Taproot 的基础

---

## 🧩 核心概念

### 交易可延展性：SegWit 要解决的问题

**什么是交易可延展性？**

交易可延展性是指能够修改交易的签名**而不会使其失效**，从而改变交易 ID (TXID)。

**核心问题：**

```
传统交易 TXID 计算：
TXID = SHA256(SHA256(整个交易包括签名))
                               ^^^^^^^^^^^^^^^^^^^^
                               问题：可变的！
```

**为什么签名是可延展的（ECDSA）：**

DER 签名编码允许多种有效表示：
```
原始签名:  304402201234567890abcdef...   (71 字节)
可延展签名: 3045022100001234567890abcdef... (72 字节，带零填充)
```

**两个签名都：**
- ✅ 密码学有效
- ✅ 通过 ECDSA 验证
- ❌ 产生**不同的 TXID**

**为什么这会破坏第 2 层协议：**

```
闪电网络通道设置：

资金交易 (TXID_A) ──→ 承诺交易 ──→ 超时交易
     ↓                      ↓                 ↓
  创建                引用              引用
  通道                TXID_A            TXID_B

如果 TXID_A 可延展：
  → 承诺交易失效
  → 超时交易失效
  → 整个通道不可用
  → 资金可能被锁定！
```

**SegWit 的解决方案：**

```
SegWit 交易 TXID 计算：
TXID = SHA256(SHA256(不含见证数据的交易))
                          ^^^^^^^^^^^^^^^^
                          从 TXID 中排除！
```

**结果：**
- 🔒 **固定 TXID**：签名不在 TXID 计算中
- ⚡ **第 2 层协议**：闪电网络成为可能
- 🔗 **交易链**：预签名交易可靠工作

---

### 见证结构：隔离的签名

**传统 vs SegWit 结构：**

**传统交易：**
```
┌─────────────────────────────────────────┐
│ 版本 │ 输入 │ 输出 │ 锁定时间 │  } 全部在 TXID 中
│         │ ┌─────┐│         │          │
│         │ │ScSig││         │          │
│         │ └─────┘│         │          │
└─────────────────────────────────────────┘
     ↓
TXID = SHA256(SHA256(所有内容))
```

**SegWit 交易：**
```
┌─────────────────────────────────────────┐
│ 版本 │ 输入 │ 输出 │ 锁定时间 │  } 基础交易
│         │ ┌─────┐│         │          │    (在 TXID 中)
│         │ │空   ││         │          │
│         │ └─────┘│         │          │
└─────────────────────────────────────────┘
        } TXID = SHA256(SHA256(仅基础部分))

┌─────────────────────────────────────────┐
│        见证数据（分离）                    │  } 不在 TXID 中
│  ┌─────────────────────────────────┐   │    (通过 wtxid
│  │ 签名 │ 公钥                      │   │     merkle 承诺)
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

**关键洞察：**
> 见证数据通过见证 merkle 树**承诺**到区块中，但**排除**在 TXID 计算之外。这种分离称为"见证隔离"。

---

### SegWit 地址类型

**P2WPKH（支付到见证公钥哈希）：**

```
Bech32 地址: bc1qw508d6qejxtdg4y5r3zarvary0c5xw7kygt080
            ^^^^
            bc1q = 原生 SegWit（版本 0）

锁定脚本: 00 14 <20字节公钥哈希>
          ^^    ^^^^^^^^^^^^^^^^^^^^^
          │     公钥哈希
          见证版本 0
```

**优势：**
- 🪶 **更低手续费**：比传统约便宜 40%
- ✅ **更好的错误检测**：Bech32 编码
- 🔤 **大小写不敏感**：减少复制粘贴错误
- 🚀 **原生 SegWit**：不需要包装

**P2SH-P2WPKH（包装的 SegWit）：**

```
Base58 地址: 3J98t1WpEZ73CNmQviecrnyiWrnqRhWNLy
            ^
            3 = P2SH 包装

锁定脚本: OP_HASH160 <20字节脚本哈希> OP_EQUAL
赎回脚本: 00 14 <20字节公钥哈希>

目的: 与旧钱包向后兼容
```

**何时使用：**
- **P2WPKH**：现代钱包，最低手续费
- **P2SH-P2WPKH**：传统钱包兼容性

---

### 交易权重单位：手续费优化

**权重单位公式（BIP 141）：**
```
权重 = (基础大小 × 4) + 见证大小

基础大小:    除了见证数据之外的所有内容
见证大小:    签名 + 公钥数据
```

**为什么 × 4 乘数？**

为见证数据创造**75% 折扣**：
```
传统 P2PKH:
  大小: 250 字节
  权重: 250 × 4 = 1000 WU
  
SegWit P2WPKH:
  基础: 110 字节 → 110 × 4 = 440 WU
  见证: 107 字节 → 107 × 1 = 107 WU
  总权重: 547 WU
  
节省: 45% 更少的权重 = 45% 更低的手续费！
```

**虚拟大小（vsize）：**
```
vsize = ceil(权重 / 4)

这是手续费估算器使用的！
```

---

### P2WPKH 栈执行

**锁定脚本（ScriptPubKey）：**
```
00 14 1d7cd6c75c2e86c08d0bad1b4c982c3e4f33e7f5
```

**见证栈：**
```
项 0: <signature>   (71 字节)
项 1: <public_key>  (33 字节)
```

**模式识别：**

Bitcoin Core 看到 `OP_0 <20字节>` 并将其解释为 **P2WPKH**，等同于：

```
OP_DUP OP_HASH160 <pubkey_hash> OP_EQUALVERIFY OP_CHECKSIG
```

**执行步骤：**

**初始状态**（已加载见证项）：
```
┌─────────────────────────────────┐
│ 021f8f6e70fc...f5 (public_key)  │ ← 栈顶
│ 304402201234...78 (signature)   │
└─────────────────────────────────┘
```

**步骤 1**：OP_DUP（复制公钥）
```
┌─────────────────────────────────┐
│ 021f8f6e70fc...f5 (public_key)  │
│ 021f8f6e70fc...f5 (public_key)  │
│ 304402201234...78 (signature)   │
└─────────────────────────────────┘
```

**步骤 2**：OP_HASH160（哈希公钥）
```
┌─────────────────────────────────┐
│ 1d7cd6c75c2e...f5 (computed)    │
│ 021f8f6e70fc...f5 (public_key)  │
│ 304402201234...78 (signature)   │
└─────────────────────────────────┘
```

**步骤 3**：PUSH 预期哈希（来自锁定脚本）
```
┌─────────────────────────────────┐
│ 1d7cd6c75c2e...f5 (expected)    │
│ 1d7cd6c75c2e...f5 (computed)    │
│ 021f8f6e70fc...f5 (public_key)  │
│ 304402201234...78 (signature)   │
└─────────────────────────────────┘
```

**步骤 4**：OP_EQUALVERIFY（比较哈希，如果相等则移除）
```
┌─────────────────────────────────┐
│ 021f8f6e70fc...f5 (public_key)  │
│ 304402201234...78 (signature)   │
└─────────────────────────────────┘
```
✅ 哈希匹配！继续。

**步骤 5**：OP_CHECKSIG（验证签名）
```
┌─────────────────────────────────┐
│ 1 (TRUE)                        │
└─────────────────────────────────┘
```

**✅ 交易有效！**

**与传统的主要区别：**
- 无 ScriptSig（空）
- 见证中的授权数据
- 执行模型与 P2PKH 相同

---

### SegWit 到 Taproot 的演进

**SegWit 为 Taproot 建立了三个关键基础：**

1. **见证结构**
   ```
   SegWit:  见证 = [signature, public_key]
   Taproot: 见证 = [signature] 或 [signature, script, control_block]
   ```

2. **版本字节系统**
   ```
   OP_0  = 见证 v0 (SegWit)
   OP_1  = 见证 v1 (Taproot)
   OP_2+ = 未来升级
   ```

3. **权重单位经济学**
   ```
   SegWit: 见证数据 75% 折扣
   Taproot: 在此基础上实现更高效率
   ```

**Taproot = SegWit + Schnorr + MAST**

---

## 💻 示例代码

### 示例 1：创建原生 SegWit (P2WPKH) 交易

```python
from bitcoinutils.setup import setup
from bitcoinutils.utils import to_satoshis
from bitcoinutils.transactions import Transaction, TxInput, TxOutput
from bitcoinutils.keys import PrivateKey, P2wpkhAddress

def create_segwit_transaction():
    """
    构建完整的原生 SegWit 交易
    在测试网上从 P2WPKH 发送到 P2WPKH
    """
    setup('testnet')
    
    # 创建密钥和地址
    private_key = PrivateKey('cTALNpTpRbbxTCJ2A5Vq88UxT44w1PE2cYqiB3n4hRvzyCev1Wwo')
    public_key = private_key.get_public_key()
    
    # 发送方地址（我们的 SegWit 地址）
    from_address = P2wpkhAddress(public_key.get_address().to_hash160())
    
    # 接收方地址（收件人 SegWit 地址）
    to_address = P2wpkhAddress('tb1qw508d6qejxtdg4y5r3zarvary0c5xw7kygt080')
    
    print("="*60)
    print("SEGWIT 交易创建")
    print("="*60)
    print(f"发送方: {from_address.to_string()}")
    print(f"接收方:   {to_address.to_string()}")
    print()
    
    # 要花费的 UTXO（用你的实际 UTXO 替换！）
    utxo_txid = 'a1b2c3d4e5f6789012345678901234567890123456789012345678901234567890'
    utxo_vout = 0
    utxo_amount = 0.00029000  # 29,000 sats
    
    # 创建交易组件
    txin = TxInput(utxo_txid, utxo_vout)
    
    # 发送 28,500 sats，剩余是手续费 (500 sats)
    txout = TxOutput(to_satoshis(0.00028500), to_address.to_script_pub_key())
    
    # 构建未签名交易
    tx = Transaction([txin], [txout])
    
    print("="*60)
    print("未签名交易")
    print("="*60)
    print(f"十六进制: {tx.serialize()}")
    print()
    
    # SegWit 签名：见证数据，不是 scriptSig！
    print("="*60)
    print("使用见证签名")
    print("="*60)
    
    signature = private_key.sign_input(tx, 0, from_address.to_script_pub_key())
    print(f"签名: {signature[:40]}...")
    print(f"公钥: {public_key.to_hex()}")
    print()
    
    # 关键：设置见证，不是 script_sig
    txin.witness = [signature, public_key.to_hex()]
    
    # 验证 scriptSig 为空
    print("="*60)
    print("SEGWIT 结构验证")
    print("="*60)
    print(f"ScriptSig: '{txin.script_sig.to_hex() if txin.script_sig else '(empty)'}'")
    print(f"见证项: {len(txin.witness)}")
    print(f"  [0] 签名 ({len(signature)//2} 字节)")
    print(f"  [1] 公钥 ({len(public_key.to_hex())//2} 字节)")
    print()
    
    # 完整的已签名交易
    signed_tx = tx.serialize()
    
    print("="*60)
    print("已签名交易")
    print("="*60)
    print(signed_tx)
    print()
    
    # 大小和权重分析
    base_size = 110  # 近似基础大小
    witness_size = len(signature)//2 + len(public_key.to_hex())//2 + 2  # +2 表示长度字节
    weight = (base_size * 4) + witness_size
    vsize = (weight + 3) // 4  # 向上取整
    
    print("="*60)
    print("交易指标")
    print("="*60)
    print(f"基础大小:    ~{base_size} 字节")
    print(f"见证大小: ~{witness_size} 字节")
    print(f"权重:       ~{weight} WU")
    print(f"虚拟大小: ~{vsize} vbytes")
    print()
    print(f"手续费: {utxo_amount - 0.00028500} BTC ({(utxo_amount - 0.00028500)*100000000:.0f} sats)")
    print(f"费率: ~{((utxo_amount - 0.00028500)*100000000 / vsize):.1f} sats/vbyte")
    print()
    
    return tx

if __name__ == "__main__":
    tx = create_segwit_transaction()
```

### 示例 2：解码和分析 SegWit 交易结构

```python
def analyze_segwit_structure():
    """
    解码原始 SegWit 交易并识别组件
    """
    # 示例已签名 SegWit 交易
    raw_tx = (
        "02000000"  # 版本
        "00"        # 标记 (SegWit 指示符)
        "01"        # 标志 (SegWit 版本)
        "01"        # 输入数量
        "a1b2c3d4e5f6789012345678901234567890123456789012345678901234567890"  # TXID
        "00000000"  # VOUT
        "00"        # ScriptSig 长度 (空)
        "ffffffff"  # 序列号
        "01"        # 输出数量
        "146f000000000000"  # 金额 (28,500 sats)
        "16"        # ScriptPubKey 长度 (22 字节)
        "00141d7cd6c75c2e86c08d0bad1b4c982c3e4f33e7f5"  # ScriptPubKey
        # 见证数据
        "02"        # 见证栈项数量
        "47"        # 签名长度 (71 字节)
        "304402201234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef02201234567801"
        "21"        # 公钥长度 (33 字节)
        "021f8f6e70fcb96c29e6c87b47ac21e5e9c2e9e5f5f5f5f5f5f5f5f5"
        "00000000"  # 锁定时间
    )
    
    print("="*60)
    print("SEGWIT 交易结构分析")
    print("="*60)
    print()
    
    print("[版本]        02000000")
    print("[标记]         00          ← SegWit 指示符")
    print("[标志]           01          ← SegWit 版本")
    print("[输入数量]    01")
    print("[TXID]           a1b2c3d4... (32 字节)")
    print("[VOUT]           00000000")
    print("[SCRIPTSIG长度]  00          ← 空！(SegWit 特性)")
    print("[序列号]       ffffffff")
    print("[输出数量]   01")
    print("[金额]          146f000000000000 (28,500 sats)")
    print("[脚本长度]     16          (22 字节)")
    print("[SCRIPTPUBKEY]   00141d7cd6c7... (见证 v0 公钥哈希)")
    print()
    print("--- 见证数据（隔离） ---")
    print()
    print("[见证项数量]  02          ← 2 项")
    print("[签名长度]        47          (71 字节)")
    print("[签名]      30440220... (ECDSA 签名)")
    print("[公钥长度]         21          (33 字节)")
    print("[公钥]     021f8f6e... (压缩公钥)")
    print()
    print("[锁定时间]       00000000")
    print()
    
    print("="*60)
    print("关键观察")
    print("="*60)
    print("✓ 标记 (00) + 标志 (01) 标识 SegWit 交易")
    print("✓ ScriptSig 为空 (00)")
    print("✓ 见证数据在输出之后")
    print("✓ TXID 计算时不包含见证数据")
    print()

if __name__ == "__main__":
    analyze_segwit_structure()
```

### 示例 3：比较交易大小（传统 vs SegWit）

```python
def compare_transaction_types():
    """
    比较传统和 SegWit 交易的大小和成本
    """
    print("="*60)
    print("交易类型对比")
    print("="*60)
    print()
    
    # 典型的 1 输入 1 输出交易大小
    legacy_p2pkh = {
        'type': '传统 P2PKH',
        'total_size': 250,
        'base_size': 250,
        'witness_size': 0,
        'weight': 250 * 4,
        'vsize': 250
    }
    
    native_segwit = {
        'type': '原生 SegWit (P2WPKH)',
        'total_size': 193,
        'base_size': 110,
        'witness_size': 107,
        'weight': (110 * 4) + 107,
        'vsize': ((110 * 4) + 107 + 3) // 4
    }
    
    wrapped_segwit = {
        'type': 'P2SH 包装的 SegWit',
        'total_size': 217,
        'base_size': 133,
        'witness_size': 107,
        'weight': (133 * 4) + 107,
        'vsize': ((133 * 4) + 107 + 3) // 4
    }
    
    # 计算节省
    fee_rate = 10  # sats/vbyte
    
    for tx_type in [legacy_p2pkh, native_segwit, wrapped_segwit]:
        fee = tx_type['vsize'] * fee_rate
        
        print(f"{tx_type['type']}")
        print(f"  总大小:   {tx_type['total_size']} 字节")
        print(f"  基础大小:    {tx_type['base_size']} 字节")
        print(f"  见证大小: {tx_type['witness_size']} 字节")
        print(f"  权重:       {tx_type['weight']} WU")
        print(f"  虚拟大小: {tx_type['vsize']} vbytes")
        print(f"  手续费 (@10 sat/vb): {fee} sats")
        
        if tx_type['type'] != '传统 P2PKH':
            savings = (legacy_p2pkh['vsize'] - tx_type['vsize']) / legacy_p2pkh['vsize'] * 100
            fee_savings = (legacy_p2pkh['vsize'] - tx_type['vsize']) * fee_rate
            print(f"  节省: {savings:.1f}% ({fee_savings} sats)")
        
        print()
    
    print("="*60)
    print("建议")
    print("="*60)
    print("✓ 使用原生 SegWit (P2WPKH) 以获得最低手续费")
    print("✓ 仅为了向后兼容使用 P2SH 包装")
    print("✓ 原生 SegWit 比传统约便宜 40%")
    print()

if __name__ == "__main__":
    compare_transaction_types()
```

### 示例 4：验证 SegWit 地址编码

```python
from bitcoinutils.keys import P2wpkhAddress, PublicKey
import hashlib

def verify_segwit_address():
    """
    手动验证 P2WPKH 地址编码
    """
    # 公钥
    pubkey_hex = '021f8f6e70fcb96c29e6c87b47ac21e5e9c2e9e5f5f5f5f5f5f5f5f5'
    pubkey = PublicKey(pubkey_hex)
    
    # 计算 Hash160 (SHA256 → RIPEMD160)
    pubkey_bytes = bytes.fromhex(pubkey_hex)
    sha256_hash = hashlib.sha256(pubkey_bytes).digest()
    hash160 = hashlib.new('ripemd160', sha256_hash).digest()
    
    print("="*60)
    print("P2WPKH 地址编码验证")
    print("="*60)
    print()
    print(f"公钥:    {pubkey_hex}")
    print(f"SHA256:        {sha256_hash.hex()}")
    print(f"Hash160:       {hash160.hex()}")
    print()
    
    # 创建 P2WPKH 地址
    p2wpkh_addr = P2wpkhAddress.from_hash(hash160.hex())
    
    print("="*60)
    print("P2WPKH 地址")
    print("="*60)
    print(f"地址: {p2wpkh_addr.to_string()}")
    print()
    
    # 锁定脚本
    locking_script = p2wpkh_addr.to_script_pub_key()
    
    print("="*60)
    print("锁定脚本 (ScriptPubKey)")
    print("="*60)
    print(f"十六进制: {locking_script.to_hex()}")
    print()
    print("分解：")
    print(f"  00  = OP_0 (见证版本)")
    print(f"  14  = PUSH 20 字节")
    print(f"  {hash160.hex()} = Hash160(pubkey)")
    print()
    
    return p2wpkh_addr

if __name__ == "__main__":
    addr = verify_segwit_address()
```

---

## 🧪 实践任务

### 任务 1：构建你的第一个原生 SegWit 交易

**目标**：在测试网上创建并广播完整的 P2WPKH 交易。

**步骤：**

1. 生成一个 SegWit 私钥
2. 派生 P2WPKH 地址 (bc1q...)
3. 向你的地址获取测试网币
4. 构建一个发送到另一个 SegWit 地址的交易
5. 使用见证数据签名（不是 scriptSig）
6. 广播并在浏览器上验证

**验证清单：**
- [ ] 地址以 `tb1q` 开头（测试网 SegWit）
- [ ] 交易 ScriptSig 为空
- [ ] 见证包含 [signature, public_key]
- [ ] 交易成功确认

### 任务 2：交易可延展性演示

**挑战**：理解为什么传统交易可延展但 SegWit 不可延展。

**部分 A**：给定这个传统交易：
```
TXID = hash(version + inputs + outputs + locktime)
                  ^^^^^^
                  包括签名！
```

**问题**：如果你修改签名的 DER 编码（例如，添加零填充），是否：
1. 签名仍然密码学有效？
2. TXID 会改变？
3. 为什么这对闪电网络是个问题？

**部分 B**：给定一个 SegWit 交易：
```
TXID = hash(version + inputs + outputs + locktime)
                          (排除见证)
```

**问题**：签名修改能否改变 TXID？为什么？

### 任务 3：权重单位计算

**给定：**
- 传统 P2PKH 交易：250 字节
- SegWit P2WPKH 交易：110 基础字节 + 107 见证字节

**计算：**
1. 传统权重单位
2. SegWit 权重单位
3. SegWit 虚拟大小 (vsize)
4. 节省百分比
5. 在 20 sats/vbyte 时的手续费节省

**展示你的计算过程！**

---

## 📘 阅读材料

### 必读材料
- **Mastering Taproot** - 第 4 章：构建 SegWit 交易（第 42-55 页）

### BIP 参考
- **BIP 141**：隔离见证（共识层） - https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki
- **BIP 143**：版本 0 见证程序的交易签名验证
- **BIP 144**：隔离见证（对等服务）
- **BIP 173**：原生 v0-16 见证输出的 Bech32 地址格式

### 补充资源
- **Bitcoin Optech**：SegWit 优势 - https://bitcoinops.org/en/topics/segregated-witness/
- **Jimmy Song**："理解 SegWit" 文章系列
- **Pieter Wuille**：在 Scaling Bitcoin 上的 SegWit 演示

---

## 💬 讨论话题

### 问题 1：可延展性影响
**"在 SegWit 之前，交易所必须等待 6 个确认才能记入存款。使用 SegWit，一些交易所在 1-2 个确认后就记入。为什么可延展性需要额外的确认，SegWit 如何使早期记入变得安全？"**

*提示：思考 TXID 变化和子交易。*

### 问题 2：见证折扣理由
**"为什么 BIP 141 给见证数据 75% 折扣（基础 × 4 乘数），而不是平等对待所有字节？这种激励结构鼓励了哪些区块链属性？"**

*提示：考虑 UTXO 集增长 vs. 签名数据存储。*

### 问题 3：SegWit 采用
**"SegWit 于 2017 年 8 月激活。为什么采用率需要数年才能达到 >80%？钱包开发者和交易所的激励和障碍是什么？"**

---

## 🎙️ 讲师笔记

欢迎见证革命！

SegWit 经常被误解为"只是"一个可延展性修复，但它远不止如此。它是比特币处理授权数据的**完全重新设计**，它使我们现在看到的整个第 2 层生态系统成为可能。

关键洞察：**将授权数据与交易数据分离**。这个简单的架构变更带来了巨大改进：

1. **固定 TXID** → 闪电网络成为可能
2. **权重单位** → 高效脚本的经济激励
3. **版本字节系统** → 到 Taproot 的清晰升级路径
4. **见证结构** → 脚本树的基础

当你追踪 P2WPKH 执行时，注意它与 P2PKH **完全相同**——只是数据来自不同的地方。Bitcoin Core 识别 `OP_0 <20字节>` 模式并说"哦，这是 P2WPKH，像 P2PKH 一样执行，但从见证中提取数据。"

这种模式识别在 Taproot 中变得更加重要。当节点看到 `OP_1 <32字节>` 时，它知道"这是 Taproot，执行见证 v1 的特殊规则。"

权重单位计算可能看起来是任意的（为什么 × 4？），但这是精妙的经济学。通过使见证数据更便宜，SegWit 激励将授权数据移出基础交易，这使 UTXO 集更小，网络更具可扩展性。Taproot 通过签名聚合进一步放大了这个优势。

下周：**Taproot 介绍**！我们将看到 Schnorr 签名如何实现密钥聚合，以及密钥调整如何为 Taproot 的隐私魔法奠定基础。

这就是一切汇聚的地方！

—Aaron

---

## 📌 关键要点

✅ SegWit 通过从 TXID 中排除见证来解决交易可延展性  
✅ 见证隔离 = 授权与交易结构分离  
✅ P2WPKH = SegWit v0，与 P2PKH 执行相同  
✅ 权重单位：(基础 × 4) + 见证 = 见证 75% 折扣  
✅ Bech32 地址 (bc1q) 是原生 SegWit，优于 Base58  
✅ 固定 TXID 使闪电网络和第 2 层协议成为可能  
✅ SegWit 是使 Taproot 成为可能的基础

---

**上一课**：[← 第 3 课：P2SH 多重签名](./lesson-03-p2sh-multisig.md)  
**下一课**：[第 5 课：Taproot 介绍：Schnorr 和密钥调整 →](./lesson-05-taproot-schnorr.md)

---

*本课程是 Chaincode Labs (2026) Taproot 脚本工程队列的一部分*

