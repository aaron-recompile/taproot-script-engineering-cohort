# 第 8 课：四叶子 Taproot 脚本树

**周次**: 8  
**时长**: 自主学习（预计 6-7 小时）  
**先修要求**: 第 7 课（双叶子 Taproot 脚本树）

---

## 🎯 学习目标

完成本课后，你将能够：

1. 构建平衡的四叶子 Merkle 树，具有两级结构
2. 使用 OP_CHECKSIGADD 实现 2-of-2 多重签名（Tapscript 创新）
3. 使用多级 Merkle 证明构建扩展控制块（97 字节）
4. 将时间锁（CSV）与脚本路径结合
5. 理解多重签名脚本的见证栈排序
6. 调试复杂的 Merkle 证明验证失败
7. 比较五种不同花费路径的效率

---

## 🧩 核心概念

### 从理论到实践的飞跃

在前面的课程中，我们掌握了 Taproot 基础和双叶子树。然而，**真正的企业级应用需要更复杂的逻辑**。四叶子脚本树代表了当前 Taproot 技术在实际应用中的主流复杂度。

**为什么四叶子脚本树如此重要？**

大多数 Taproot 应用停留在简单的密钥路径花费，这最大化隐私但使 Taproot 的大部分智能合约潜力未开发。四叶子树展示了几个关键能力：

### 真实世界应用场景

**四叶子树能够实现：**

| 场景 | 描述 |
|------|------|
| **钱包恢复** | 具有时间锁 + 多重签名 + 紧急路径的渐进式访问控制 |
| **闪电网络通道** | 针对不同参与者集合的多种合作关闭场景 |
| **原子交换** | 具有各种回退条件的哈希时间锁定合约 |
| **继承规划** | 基于时间的访问和多受益人选项 |
| **企业金库** | 分层授权（立即、时间延迟、紧急、多重签名） |

### 技术优势

1. **选择性披露**：只暴露执行的脚本；其他脚本保持隐藏
2. **手续费效率**：比等效的传统多条件脚本更小
3. **灵活逻辑**：在单一承诺内的多个执行路径
4. **隐私保护**：未使用的路径永远不会在链上揭示

### 四叶子 Merkle 树结构

与双叶子树（单分支层级）不同，**四叶子树具有平衡的两级 Merkle 结构**：

```
                    Merkle 根
                    /          \
              Branch 0        Branch 1
              /      \        /      \
         Script 0  Script 1  Script 2  Script 3
        (哈希锁)(多重签名)  (CSV)    (简单)
```

**树构建公式：**

```python
# 级别 1：计算各个叶子哈希
TapLeaf_0 = Tagged_Hash("TapLeaf", 0xc0 + len(script0) + script0)
TapLeaf_1 = Tagged_Hash("TapLeaf", 0xc0 + len(script1) + script1)
TapLeaf_2 = Tagged_Hash("TapLeaf", 0xc0 + len(script2) + script2)
TapLeaf_3 = Tagged_Hash("TapLeaf", 0xc0 + len(script3) + script3)

# 级别 2：计算分支哈希（在每个对内排序）
Branch_0 = Tagged_Hash("TapBranch", sorted([TapLeaf_0, TapLeaf_1]))
Branch_1 = Tagged_Hash("TapBranch", sorted([TapLeaf_2, TapLeaf_3]))

# 级别 3：计算 Merkle 根
Merkle_Root = Tagged_Hash("TapBranch", sorted([Branch_0, Branch_1]))

# 最终：计算输出密钥
tweak = Tagged_Hash("TapTweak", internal_pubkey + Merkle_Root)
output_key = internal_pubkey + tweak × G
```

### 真实案例研究：在测试网上完整验证

让我们分析一个在测试网上验证的实际四叶子树：

**共享 Taproot 地址：**
```
tb1pjfdm902y2adr08qnn4tahxjvp6x5selgmvzx63yfqk2hdey02yvqjcr29q
```

**特性**：使用同一地址的五种不同花费方法！

**四个脚本路径：**

1. **脚本 0（SHA256 哈希锁）**：任何知道 "helloworld" 的人都可以花费
   - 实现用于原子交换的哈希锁模式
   - 见证：`[preimage, script, control_block]`

2. **脚本 1（2-of-2 多重签名）**：需要 Alice 和 Bob 合作
   - 使用 Tapscript 高效的 **OP_CHECKSIGADD**（不是 OP_CHECKMULTISIG！）
   - 见证：`[bob_sig, alice_sig, script, control_block]`

3. **脚本 2（CSV 时间锁）**：Bob 在等待 2 个区块后可以花费
   - 实现相对时间锁
   - 见证：`[bob_sig, script, control_block]`
   - **关键**：交易输入必须设置自定义序列值

4. **脚本 3（简单签名）**：Bob 可以立即用签名花费
   - 最简单的脚本路径
   - 见证：`[bob_sig, script, control_block]`

5. **密钥路径**：Alice 使用调整后的私钥获得最大隐私
   - 看起来像普通的单签名交易
   - 见证：`[alice_sig]`（只有 64 字节！）

### OP_CHECKSIGADD：Tapscript 创新

传统比特币多重签名使用 **OP_CHECKMULTISIG**，这有几个问题：
- 复杂的栈操作
- 差一错误（必须推送虚拟值）
- 检查所有签名组合（效率低）
- 需要 33 字节压缩公钥

**Tapscript 引入了 OP_CHECKSIGADD**，它：
- ✅ 使用简单的计数器机制
- ✅ 逐个验证签名，失败时停止
- ✅ 原生支持 32 字节 x-only 公钥
- ✅ 更干净的栈操作，无需虚拟值

**OP_CHECKSIGADD 脚本模式：**

```python
Script([
    "OP_0",                      # 初始化计数器为 0
    alice_pub.to_x_only_hex(),   # Alice 的 32 字节公钥
    "OP_CHECKSIGADD",            # 验证 Alice，递增计数器
    bob_pub.to_x_only_hex(),     # Bob 的 32 字节公钥
    "OP_CHECKSIGADD",            # 验证 Bob，递增计数器
    "OP_2",                      # 所需签名数量
    "OP_EQUAL"                   # 检查计数器 == 2
])
```

**工作原理：**
1. 计数器从 0 开始
2. 对于每个签名：验证并在有效时递增计数器
3. 将最终计数器与所需数量比较
4. 如果计数器 == 所需数量，脚本成功

### 扩展控制块（97 字节）

对于**四叶子树**，控制块扩展到**97 字节**以包含**两级 Merkle 证明**：

```
┌─────┬────────────────────┬────────────────────┬────────────────────┐
│字节 │    字节 2-33       │    字节 34-65      │    字节 66-97      │
│  1  │                    │                    │                    │
├─────┼────────────────────┼────────────────────┼────────────────────┤
│ c0  │ 内部公钥           │ 兄弟节点 1 哈希    │ 兄弟节点 2 哈希    │
│     │    (32 字节)       │   (32 字节)        │   (32 字节)        │
└─────┴────────────────────┴────────────────────┴────────────────────┘

字节 1: 叶子版本 (c0) + 奇偶位
字节 2-33: 内部公钥
字节 34-65: 第一个兄弟节点哈希（同级）
字节 66-97: 第二个兄弟节点哈希（父级）
```

**每个脚本的 Merkle 证明路径：**

```python
# 当花费脚本 0（哈希锁）时：
# 控制块包含：
#   - 兄弟节点 1：脚本 1 的 TapLeaf 哈希（同一分支）
#   - 兄弟节点 2：Branch 1 的 TapBranch 哈希（父级）

# 当花费脚本 1（多重签名）时：
# 控制块包含：
#   - 兄弟节点 1：脚本 0 的 TapLeaf 哈希
#   - 兄弟节点 2：Branch 1 的 TapBranch 哈希

# 当花费脚本 2（CSV）时：
# 控制块包含：
#   - 兄弟节点 1：脚本 3 的 TapLeaf 哈希
#   - 兄弟节点 2：Branch 0 的 TapBranch 哈希

# 当花费脚本 3（简单签名）时：
# 控制块包含：
#   - 兄弟节点 1：脚本 2 的 TapLeaf 哈希
#   - 兄弟节点 2：Branch 0 的 TapBranch 哈希
```

**美妙之处**：每个脚本通过仅在每一级揭示其兄弟节点来证明其合法性，而不是整个树！

### 控制块大小对比

| 树类型 | 控制块大小 | 结构 |
|--------|-----------|------|
| **单叶子** | 33 字节 | `[版本+奇偶位] + [内部公钥]` |
| **双叶子** | 65 字节 | `+ [兄弟节点哈希]` |
| **四叶子** | 97 字节 | `+ [兄弟节点哈希] + [父级兄弟节点哈希]` |
| **八叶子** | 129 字节 | `+ [兄弟节点] + [父级兄弟节点] + [祖父级兄弟节点]` |

随着脚本树深度增加，控制块**线性增长**（每级 32 字节），但它仍然比传统的多重签名脚本更高效得多。

---

## 💻 示例代码

### 示例 1：构建四叶子 Taproot 树

```python
from bitcoinutils.setup import setup
from bitcoinutils.keys import PrivateKey
from bitcoinutils.script import Script
from bitcoinutils.utils import Sequence, TYPE_RELATIVE_TIMELOCK
import hashlib

def build_four_leaf_taproot():
    """
    提交阶段：构建具有多个条件的四叶子 Taproot 树
    """
    setup('testnet')
    
    print("="*70)
    print("提交阶段：四叶子 Taproot 树构建")
    print("="*70)
    print()
    
    # 步骤 1：设置密钥
    alice_priv = PrivateKey.from_wif("cT3tJP7BjwL25nQ9rHQuSCLugr3Vs5XfFKsTs7j5gHDgULyMmm1y")
    bob_priv = PrivateKey.from_wif("cSNdLFDf3wjx1rswNL2jKykbVkC6o56o5nYZi4FUkWKjFn2Q5DSG")
    alice_pub = alice_priv.get_public_key()
    bob_pub = bob_priv.get_public_key()
    
    print("步骤 1：密钥设置")
    print("-" * 70)
    print(f"Alice (内部密钥): {alice_pub.to_hex()}")
    print(f"Bob (脚本路径):   {bob_pub.to_hex()}")
    print()
    
    # 步骤 2：构建脚本 0 - SHA256 哈希锁
    preimage = "helloworld"
    hash0 = hashlib.sha256(preimage.encode('utf-8')).hexdigest()
    script0 = Script([
        'OP_SHA256',
        hash0,
        'OP_EQUALVERIFY',
        'OP_TRUE'
    ])
    
    print("步骤 2：脚本 0 - SHA256 哈希锁")
    print("-" * 70)
    print(f"原像: {preimage}")
    print(f"哈希:     {hash0}")
    print(f"脚本:   {script0.to_hex()}")
    print()
    
    # 步骤 3：构建脚本 1 - 2-of-2 多重签名（OP_CHECKSIGADD）
    script1 = Script([
        "OP_0",                      # 初始化计数器
        alice_pub.to_x_only_hex(),   # Alice 的 x-only 公钥
        "OP_CHECKSIGADD",            # 验证 + 递增
        bob_pub.to_x_only_hex(),     # Bob 的 x-only 公钥
        "OP_CHECKSIGADD",            # 验证 + 递增
        "OP_2",                      # 所需数量
        "OP_EQUAL"                   # 检查相等性
    ])
    
    print("步骤 3：脚本 1 - 2-of-2 多重签名（OP_CHECKSIGADD）")
    print("-" * 70)
    print(f"脚本: {script1.to_hex()}")
    print("⚡ 使用 OP_CHECKSIGADD，不是 OP_CHECKMULTISIG！")
    print()
    
    # 步骤 4：构建脚本 2 - CSV 时间锁
    relative_blocks = 2
    seq = Sequence(TYPE_RELATIVE_TIMELOCK, relative_blocks)
    script2 = Script([
        seq.for_script(),            # 推送序列值
        "OP_CHECKSEQUENCEVERIFY",    # 验证相对时间锁
        "OP_DROP",                   # 清理栈
        bob_pub.to_x_only_hex(),     # Bob 的公钥
        "OP_CHECKSIG"                # 验证签名
    ])
    
    print("步骤 4：脚本 2 - CSV 时间锁（2 个区块）")
    print("-" * 70)
    print(f"时间锁: {relative_blocks} 个区块")
    print(f"脚本:   {script2.to_hex()}")
    print()
    
    # 步骤 5：构建脚本 3 - 简单签名
    script3 = Script([
        bob_pub.to_x_only_hex(),
        "OP_CHECKSIG"
    ])
    
    print("步骤 5：脚本 3 - 简单签名")
    print("-" * 70)
    print(f"脚本: {script3.to_hex()}")
    print()
    
    # 步骤 6：构建树结构（嵌套列表格式）
    # [[左分支], [右分支]]
    tree = [[script0, script1], [script2, script3]]
    
    print("步骤 6：树结构")
    print("-" * 70)
    print("Merkle 树:")
    print("              根")
    print("             /    \\")
    print("       Branch0    Branch1")
    print("        /  \\        /  \\")
    print("    S0    S1      S2    S3")
    print("   哈希  多重     CSV   签名")
    print()
    
    # 步骤 7：生成 Taproot 地址
    taproot_address = alice_pub.get_taproot_address(tree)
    
    print("步骤 7：Taproot 地址已生成")
    print("-" * 70)
    print(f"地址: {taproot_address.to_string()}")
    print()
    
    print("="*70)
    print("从此地址花费的五种方式")
    print("="*70)
    print("1. 密钥路径: Alice 使用内部密钥签名（最大隐私）")
    print("2. 脚本 0: 任何有 'helloworld' 原像的人")
    print("3. 脚本 1: Alice + Bob 多重签名（OP_CHECKSIGADD）")
    print("4. 脚本 2: Bob 在 2 个区块后（CSV 时间锁）")
    print("5. 脚本 3: Bob 立即签名")
    print()
    
    return taproot_address, tree, alice_priv, bob_priv

# 执行树构建
address, tree, alice_key, bob_key = build_four_leaf_taproot()
```

### 示例 2：脚本 0 - 哈希锁路径花费

```python
from bitcoinutils.transactions import Transaction, TxInput, TxOutput, TxWitnessInput
from bitcoinutils.script import ControlBlock
from bitcoinutils.utils import to_satoshis

def spend_hashlock_path(tree, alice_pub, taproot_address):
    """
    脚本路径 0：使用原像的哈希锁花费
    """
    setup('testnet')
    
    print("="*70)
    print("脚本路径 0：哈希锁花费")
    print("="*70)
    print()
    
    # UTXO 信息
    commit_txid = "245563c5aa4c6d32fc34eed2f182b5ed76892d13370f067dc56f34616b66c468"
    input_amount = 0.00001200  # 1200 sats
    output_amount = 0.00000666  # 666 sats
    
    print("交易详情:")
    print("-" * 70)
    print(f"输入:  {input_amount} BTC")
    print(f"输出: {output_amount} BTC")
    print(f"手续费:    {input_amount - output_amount} BTC")
    print()
    
    # 构建交易
    txin = TxInput(commit_txid, 0)
    txout = TxOutput(
        to_satoshis(output_amount),
        alice_pub.get_taproot_address().to_script_pub_key()
    )
    tx = Transaction([txin], [txout], has_segwit=True)
    
    # 为脚本 0 构建控制块
    control_block = ControlBlock(
        alice_pub,
        tree,
        0,  # 脚本索引 0（哈希锁）
        is_odd=taproot_address.is_odd()
    )
    
    print("控制块构建:")
    print("-" * 70)
    print(f"长度: {len(bytes.fromhex(control_block.to_hex()))} 字节")
    print(f"脚本索引: 0 (哈希锁)")
    print()
    
    # 解码控制块
    cb_bytes = bytes.fromhex(control_block.to_hex())
    print("控制块分解:")
    print(f"  字节 1:      版本 + 奇偶位")
    print(f"  字节 2-33:  内部公钥（Alice）")
    print(f"  字节 34-65: 兄弟节点 1（脚本 1 TapLeaf）")
    print(f"  字节 66-97: 兄弟节点 2（Branch 1 TapBranch）")
    print()
    
    # 构建见证
    preimage = "helloworld"
    preimage_hex = preimage.encode('utf-8').hex()
    script0 = tree[0][0]  # 左分支中的第一个脚本
    
    witness = TxWitnessInput([
        preimage_hex,
        script0.to_hex(),
        control_block.to_hex()
    ])
    
    tx.witnesses.append(witness)
    
    print("见证栈:")
    print("-" * 70)
    print(f"[0] 原像: {preimage_hex}")
    print(f"[1] 脚本:   {script0.to_hex()}")
    print(f"[2] 控制块:  {control_block.to_hex()[:40]}...")
    print()
    
    print(f"交易 ID: {tx.get_txid()}")
    print(f"大小: {tx.get_size()} 字节")
    print(f"虚拟大小: {tx.get_vsize()} vbytes")
    
    return tx

# 执行哈希锁花费
# tx_hash = spend_hashlock_path(tree, alice_pub, address)
```

### 示例 3：脚本 1 - 多重签名路径（OP_CHECKSIGADD）

```python
def spend_multisig_path(tree, alice_priv, bob_priv, taproot_address):
    """
    脚本路径 1：使用 OP_CHECKSIGADD 的 2-of-2 多重签名
    """
    setup('testnet')
    
    print("="*70)
    print("脚本路径 1：多重签名（OP_CHECKSIGADD）")
    print("="*70)
    print()
    
    alice_pub = alice_priv.get_public_key()
    bob_pub = bob_priv.get_public_key()
    
    # UTXO 信息
    commit_txid = "1ed5a3e97a6d3bc0493acc2aac15011cd99000b52e932724766c3d277d76daac"
    input_amount = 0.00001400  # 1400 sats
    output_amount = 0.00000668  # 668 sats
    
    print("交易详情:")
    print("-" * 70)
    print(f"输入:  {input_amount} BTC")
    print(f"输出: {output_amount} BTC")
    print()
    
    # 构建交易
    txin = TxInput(commit_txid, 0)
    txout = TxOutput(
        to_satoshis(output_amount),
        alice_pub.get_taproot_address().to_script_pub_key()
    )
    tx = Transaction([txin], [txout], has_segwit=True)
    
    # 脚本 1 的控制块
    control_block = ControlBlock(
        alice_pub,
        tree,
        1,  # 脚本索引 1（多重签名）
        is_odd=taproot_address.is_odd()
    )
    
    script1 = tree[0][1]  # 左分支中的第二个脚本
    
    # 使用两个密钥签名（脚本路径模式）
    sig_alice = alice_priv.sign_taproot_input(
        tx, 0,
        [taproot_address.to_script_pub_key()],
        [to_satoshis(input_amount)],
        script_path=True,
        tapleaf_script=script1,
        tweak=False
    )
    
    sig_bob = bob_priv.sign_taproot_input(
        tx, 0,
        [taproot_address.to_script_pub_key()],
        [to_satoshis(input_amount)],
        script_path=True,
        tapleaf_script=script1,
        tweak=False
    )
    
    print("生成的签名:")
    print("-" * 70)
    print(f"Alice: {sig_alice.hex()[:40]}...")
    print(f"Bob:   {sig_bob.hex()[:40]}...")
    print()
    
    # 关键：见证栈顺序很重要！
    # Bob 的签名在前，第二个被消费
    # Alice 的签名在后，第一个被消费
    witness = TxWitnessInput([
        sig_bob,                  # 栈底，第二个被消费
        sig_alice,                # 栈顶，第一个被消费
        script1.to_hex(),
        control_block.to_hex()
    ])
    
    tx.witnesses.append(witness)
    
    print("⚠️  见证栈顺序很关键:")
    print("-" * 70)
    print("[0] Bob 的签名   (OP_CHECKSIGADD 第二个消费)")
    print("[1] Alice 的签名 (OP_CHECKSIGADD 第一个消费)")
    print("[2] 脚本")
    print("[3] 控制块")
    print()
    
    print(f"交易 ID: {tx.get_txid()}")
    
    return tx

# 执行多重签名花费
# tx_multi = spend_multisig_path(tree, alice_key, bob_key, address)
```

### 示例 4：OP_CHECKSIGADD 栈执行可视化

```python
def visualize_op_checksigadd_execution():
    """
    可视化 OP_CHECKSIGADD 多重签名的完整栈执行
    """
    print("="*70)
    print("OP_CHECKSIGADD 栈执行追踪")
    print("="*70)
    print()
    
    print("脚本: OP_0 [Alice_公钥] OP_CHECKSIGADD [Bob_公钥] OP_CHECKSIGADD OP_2 OP_EQUAL")
    print()
    
    print("初始状态：栈上的见证数据")
    print("-" * 70)
    print("│ sig_alice │  ← 栈顶，第一个被消费")
    print("│ sig_bob   │  ← 第二个被消费")
    print("└───────────┘")
    print()
    
    print("步骤 1: OP_0 - 初始化计数器")
    print("-" * 70)
    print("│ 0         │  ← 计数器 = 0")
    print("│ sig_alice │")
    print("│ sig_bob   │")
    print("└───────────┘")
    print()
    
    print("步骤 2: [Alice_公钥] - 推送 Alice 的公钥")
    print("-" * 70)
    print("│ alice_pk  │  ← Alice 的 32 字节 x-only 公钥")
    print("│ 0         │  ← 计数器")
    print("│ sig_alice │")
    print("│ sig_bob   │")
    print("└───────────┘")
    print()
    
    print("步骤 3: OP_CHECKSIGADD - 验证 Alice 并递增")
    print("-" * 70)
    print("过程:")
    print("  • 弹出 alice_pk")
    print("  • 弹出 sig_alice（从下层）")
    print("  • 验证: schnorr_verify(sig_alice, alice_pk, sighash)")
    print("  • 弹出计数器 0")
    print("  • 验证成功: 推送 (0 + 1 = 1)")
    print()
    print("│ 1         │  ← 计数器递增到 1")
    print("│ sig_bob   │")
    print("└───────────┘")
    print()
    
    print("步骤 4: [Bob_公钥] - 推送 Bob 的公钥")
    print("-" * 70)
    print("│ bob_pk    │  ← Bob 的 32 字节 x-only 公钥")
    print("│ 1         │  ← 计数器")
    print("│ sig_bob   │")
    print("└───────────┘")
    print()
    
    print("步骤 5: OP_CHECKSIGADD - 验证 Bob 并递增")
    print("-" * 70)
    print("过程:")
    print("  • 弹出 bob_pk")
    print("  • 弹出 sig_bob")
    print("  • 验证: schnorr_verify(sig_bob, bob_pk, sighash)")
    print("  • 弹出计数器 1")
    print("  • 验证成功: 推送 (1 + 1 = 2)")
    print()
    print("│ 2         │  ← 计数器递增到 2")
    print("└───────────┘")
    print()
    
    print("步骤 6: OP_2 - 推送所需签名数量")
    print("-" * 70)
    print("│ 2         │  ← 所需数量")
    print("│ 2         │  ← 实际数量")
    print("└───────────┘")
    print()
    
    print("步骤 7: OP_EQUAL - 检查相等性")
    print("-" * 70)
    print("过程:")
    print("  • 弹出两个 2")
    print("  • 比较: 2 == 2 是 TRUE")
    print("  • 推送 1（成功）")
    print()
    print("│ 1         │  ← 脚本成功 ✓")
    print("└───────────┘")
    print()
    
    print("="*70)
    print("OP_CHECKSIGADD vs OP_CHECKMULTISIG")
    print("="*70)
    print()
    print("OP_CHECKSIGADD 优势:")
    print("  ✓ 简单的计数器机制")
    print("  ✓ 逐个验证，失败时停止")
    print("  ✓ 支持 32 字节 x-only 公钥")
    print("  ✓ 不需要虚拟值")
    print("  ✓ 更干净的栈操作")
    print()
    print("OP_CHECKMULTISIG 问题:")
    print("  ✗ 复杂的栈操作")
    print("  ✗ 必须检查所有签名组合")
    print("  ✗ 需要 33 字节压缩公钥")
    print("  ✗ 差一错误（虚拟值）")

visualize_op_checksigadd_execution()
```

### 示例 5：四叶子树的控制块验证

```python
def verify_four_leaf_control_block():
    """
    验证控制块并为四叶子树重构 Merkle 根
    """
    print("="*70)
    print("四叶子控制块验证")
    print("="*70)
    print()
    
    # 来自测试网交易的真实控制块（脚本 1 - 多重签名）
    cb_hex = "c050be5fc44ec580c387bf45df275aaa8b27e2d7716af31f10eeed357d126bb4d3fe78d8523ce9603014b28739a51ef826f791aa17511e617af6dc96a8f10f659eda55197526f26fa309563b7a3551ca945c046e5b7ada957e59160d4d27f299e3"
    
    cb_bytes = bytes.fromhex(cb_hex)
    
    print(f"控制块长度: {len(cb_bytes)} 字节")
    print()
    
    # 解析控制块
    version_parity = cb_bytes[0]
    leaf_version = version_parity & 0xfe
    parity = version_parity & 0x01
    internal_pubkey = cb_bytes[1:33]
    sibling_1 = cb_bytes[33:65]
    sibling_2 = cb_bytes[65:97]
    
    print("解析的控制块:")
    print("-" * 70)
    print(f"叶子版本:   0x{leaf_version:02x} (Tapscript v0)")
    print(f"奇偶位:     {parity} ({'奇数' if parity else '偶数'} y 坐标)")
    print(f"内部密钥:   {internal_pubkey.hex()}")
    print(f"  → Alice 的内部公钥")
    print()
    print(f"兄弟节点 1:      {sibling_1.hex()}")
    print(f"  → 脚本 0（哈希锁）TapLeaf 哈希")
    print()
    print(f"兄弟节点 2:      {sibling_2.hex()}")
    print(f"  → Branch 1（脚本 2+3）TapBranch 哈希")
    print()
    
    print("="*70)
    print("MERKLE 证明可视化")
    print("="*70)
    print()
    print("当花费脚本 1（多重签名）时，验证者:")
    print()
    print("1. 计算脚本 1 的 TapLeaf 哈希")
    print("   TapLeaf_1 = H('TapLeaf' || version || len || script1)")
    print()
    print("2. 使用兄弟节点 1 计算 Branch 0")
    print("   Branch_0 = H('TapBranch' || sort(TapLeaf_0, TapLeaf_1))")
    print()
    print("3. 使用兄弟节点 2（Branch 1）计算 Merkle 根")
    print("   Merkle_Root = H('TapBranch' || sort(Branch_0, Branch_1))")
    print()
    print("4. 计算调整值和输出密钥")
    print("   tweak = H('TapTweak' || internal_key || Merkle_Root)")
    print("   output_key = internal_key + tweak × G")
    print()
    print("5. 将 output_key 与地址比较")
    print("   如果匹配：脚本 1 被证明是承诺树的一部分 ✓")
    print()
    
    print("🔒 保持隐藏的内容:")
    print("  • 脚本 0 的实际内容（只揭示了哈希）")
    print("  • 脚本 2 和 3 的实际内容")
    print("  • 密钥路径花费是否可能")

verify_four_leaf_control_block()
```

---

## 🧪 实践任务

### 任务 1：构建你自己的四叶子企业合约

**目标**：为企业金库创建完整的四叶子 Taproot 树。

**场景**：设计具有以下条件的金库：
- **脚本 0**：CEO 可以立即花费
- **脚本 1**：2-of-3 董事会多重签名
- **脚本 2**：任何董事会成员在 30 天后（CSV）
- **脚本 3**：3-of-3 紧急恢复

**步骤：**

1. 为 CEO 和 3 个董事会成员生成密钥
2. 构建所有四个脚本
3. 构建树结构
4. 生成 Taproot 地址
5. 在测试网上资助并测试至少 2 个花费路径

**代码模板：**

```python
# CEO 立即花费
script0 = Script([ceo_pub.to_x_only_hex(), 'OP_CHECKSIG'])

# 2-of-3 董事会多重签名（使用 OP_CHECKSIGADD）
script1 = Script([
    'OP_0',
    board1_pub.to_x_only_hex(), 'OP_CHECKSIGADD',
    board2_pub.to_x_only_hex(), 'OP_CHECKSIGADD',
    board3_pub.to_x_only_hex(), 'OP_CHECKSIGADD',
    'OP_2', 'OP_EQUAL'  # 2-of-3
])

# 设计脚本 2 和 3，构建树，测试！
```

**交付成果：**
- 2+ 个花费路径的交易 ID
- 控制块大小分析
- 讨论：在紧急情况下你会使用哪个路径？

### 任务 2：调试 OP_CHECKSIGADD 签名顺序

**挑战**：学生的多重签名交易失败了。找出错误。

**错误代码：**

```python
# 错误代码 - 多重签名失败！
def broken_multisig():
    # ... 交易设置 ...
    
    sig_alice = alice_priv.sign_taproot_input(...)
    sig_bob = bob_priv.sign_taproot_input(...)
    
    # 错误：见证顺序错误
    witness = TxWitnessInput([
        sig_alice,  # Alice 在前
        sig_bob,    # Bob 在后
        script1.to_hex(),
        control_block.to_hex()
    ])
```

**你的任务：**
1. 解释为什么见证顺序是错误的
2. 追踪栈执行，显示它在哪里失败
3. 修复代码
4. 解释 3-of-3 多重签名的正确见证排序

**提示**：思考 OP_CHECKSIGADD 先消费哪个签名。

### 任务 3：Merkle 根重构

**目标**：从控制块手动重构 Merkle 根。

**给定：**
- 来自脚本 1 花费的控制块（97 字节）
- 脚本 1 内容（多重签名脚本）

**任务：**

1. 解析控制块
2. 提取两个兄弟节点哈希
3. 计算脚本 1 的 TapLeaf 哈希
4. 使用兄弟节点 1 计算 Branch 0 哈希
5. 使用兄弟节点 2 计算 Merkle 根
6. 验证结果是否与预期地址匹配

**奖励**：编写代码自动化此验证，适用于任何脚本。

---

## 📘 阅读材料

### 必读材料
- **Mastering Taproot** - 第 8 章：四叶子脚本树（第 109-127 页）
- 重点：
  - 平衡 Merkle 树结构（第 110-113 页）
  - OP_CHECKSIGADD 实现（第 117-121 页）
  - 扩展控制块（第 122-124 页）
  - 多级 Merkle 证明（第 124-127 页）

### BIP 参考
- **BIP 342**：Tapscript 规范
  - 章节：OP_CHECKSIGADD 操作码
  - 章节：签名验证规则
- **BIP 341**：Taproot 规范
  - 章节：多级 Merkle 证明验证

### 补充资源
- **Bitcoin Optech**：Tapscript 改进
- **OP_CHECKSIGADD 深度解析**：理解计数器机制
- **真实示例**：分析主网上的四叶子花费

---

## 💬 讨论话题

### 问题 1：OP_CHECKSIGADD 设计选择

**"为什么 Tapscript 引入 OP_CHECKSIGADD 而不是改进 OP_CHECKMULTISIG？这个改变反映了什么基本设计哲学？OP_CHECKSIGADD 能否用于实现 3-of-5 多重签名？脚本会是什么样子？"**

*提示：思考增量验证与检查所有组合。*

### 问题 2：控制块增长

**"对于四叶子树，控制块是 97 字节。对于八叶子（三级），它们将是 129 字节。在什么树深度下，控制块开销超过收益？考虑手续费成本和隐私。设计一个选择树深度的决策框架。"**

*考虑：脚本复杂度、路径数量、每个路径的可能性。*

### 问题 3：真实世界应用

**"为特定用例（闪电通道、继承、企业金库等）设计一个真实的四叶子 Taproot 合约。解释：(1) 为什么需要四叶子，(2) 每个脚本做什么，(3) 你期望最常使用哪个路径，(4) 隐私如何被保护。"**

*具体说明脚本、时间锁和威胁模型。*

---

## 🎙️ 讲师笔记

欢迎来到企业级 Taproot 开发！

如果你已经掌握了第 5-7 课，你已经准备好了。四叶子树可能看起来很复杂，但它们只是**两个双叶子树的组合**。原理相同；Merkle 证明只是更深一层。

**本课的关键洞察：**

1. **OP_CHECKSIGADD 是游戏改变者**：这个操作码使多重签名在 Tapscript 中变得优雅。不再需要虚拟值，没有差一错误，并且与 x-only 公钥完美配合。

2. **见证顺序很关键**：对于多重签名，签名必须按与消费方式相反的顺序排列。见证中的最后一个签名首先被 OP_CHECKSIGADD 消费。如果搞错了，你的交易就会失败。

3. **控制块线性增长**：随着深度增加，33 → 65 → 97 → 129 字节。每级增加 32 字节用于该级的兄弟节点哈希。

4. **五种花费路径，一个地址**：本课的案例研究显示了从同一地址花费的五种完全不同的方式。这就是 Taproot 的力量在行动。

**常见陷阱：**

- **多重签名中见证顺序错误**：当脚本期望 Alice 在前时，Bob 的签名在 Alice 的签名之前
- **忘记 CSV 序列**：脚本 2（CSV）需要在交易输入上设置 `sequence`
- **索引混淆**：使用嵌套列表结构 `[[s0, s1], [s2, s3]]`，脚本索引仍然是平面的：0, 1, 2, 3

**调试提示**：当四叶子花费失败时，检查：
1. 控制块是否正好是 97 字节？
2. 它是否包含正确的两个兄弟节点哈希？
3. 你能手动重构 Merkle 根吗？
4. 对于多重签名：见证顺序是否正确？

下一课，我们将探索**高级 Taproot 模式**：将时间锁与哈希锁（HTLC）结合、原子交换和闪电网络集成。但首先掌握四叶子树——它们是一切高级内容的基础。

花时间理解 OP_CHECKSIGADD 栈执行。理解计数器如何递增对于调试多重签名脚本至关重要。

在 Discord 上见！

—Aaron

---

## 📌 关键要点

✅ 四叶子树具有两级 Merkle 结构（平衡二叉树）  
✅ OP_CHECKSIGADD 优于 OP_CHECKMULTISIG（用于 Tapscript）  
✅ 控制块扩展到 97 字节（包含两个兄弟节点哈希）  
✅ 见证顺序很关键：最后一个签名首先被消费  
✅ CSV 时间锁需要在交易输入上设置 `sequence`  
✅ 五种花费路径可能：4 个脚本路径 + 1 个密钥路径  
✅ 每个路径只揭示其 Merkle 证明，而不是整个树  
✅ 隐私被保护：未使用的路径永远不会在链上揭示

---

**下一课**：[第 9 课：高级 Taproot 模式 →](./lesson-09-advanced-patterns.md)

---

*本课程是 Chaincode Labs (2026) Taproot 脚本工程队列的一部分*

