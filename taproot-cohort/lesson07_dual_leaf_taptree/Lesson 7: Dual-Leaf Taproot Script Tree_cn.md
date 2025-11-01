# 第 7 课：双叶子 Taproot 脚本树

**周次**: 7  
**时长**: 自主学习（预计 5-6 小时）  
**先修要求**: 第 6 课（单叶子 Taproot 脚本树）

---

## 🎯 学习目标

完成本课后，你将能够：

1. 构建具有 Merkle 结构的双叶子 Taproot 脚本树
2. 正确计算 TapLeaf 和 TapBranch 哈希
3. 使用 Merkle 证明构建控制块（双叶子为 65 字节）
4. 从同一地址实现多个花费路径
5. 理解用于 Merkle 根计算的字典序排序
6. 调试 Merkle 证明验证失败
7. 比较不同花费路径之间的隐私和效率权衡

---

## 🧩 核心概念

### 从单叶子到双叶子：Taproot 的真正力量

在第 6 课中，我们掌握了单叶子 Taproot 脚本路径，其中一个脚本条件被承诺到地址中。但 Taproot 的真正力量在于其**多分支脚本树架构**——能够在单个地址内优雅地组织多个不同的花费条件。

**演进：**

```
单叶子树:             双叶子树:
                      
   TapLeaf 哈希        Merkle 根
        │                   / \
   [哈希锁]           TapLeaf A   TapLeaf B
                     [哈希锁]  [Bob 脚本]
```

### 业务场景：Alice 的双路径托管

想象这个业务场景：**Alice 想要创建一个数字托管合约**，支持基于秘密信息（哈希锁）的自动解锁，并为 Bob 提供直接私钥控制权限。

**在传统比特币中**，这需要：
- 复杂的多重签名脚本，或
- 多个独立地址，或
- 暴露所有条件的 P2SH

**Taproot 的双叶子脚本树优雅地将这些整合到单个地址：**

1. **脚本路径 1**：哈希锁脚本 - 任何知道 "helloworld" 的人都可以花费
2. **脚本路径 2**：Bob 脚本 - 只有 Bob 的私钥持有者可以花费
3. **密钥路径**：Alice 作为内部密钥持有者可以直接花费（最大隐私）

**优雅之处**：外部观察者无法区分这是简单支付还是复杂的三路径条件合约。**只有在实际花费时，使用的路径才会被选择性地揭示。**

### 双叶子脚本树的 Merkle 结构

与直接将 TapLeaf 哈希用作 Merkle 根的单叶子树不同，**双叶子脚本树需要构建真正的 Merkle 树**。

**Merkle 树结构：**

```
                    Merkle 根
                    (TapBranch)
                    /          \
              TapLeaf A      TapLeaf B
           [哈希脚本]   [Bob 脚本]
```

**技术实现关键点：**

1. **TapLeaf 哈希计算**：每个脚本分别计算其 TapLeaf 哈希
2. **字典序排序**：在组合之前对两个 TapLeaf 哈希进行排序
3. **TapBranch 哈希计算**：排序后计算 TapBranch 哈希
4. **控制块构建**：每个脚本需要包含其兄弟节点哈希作为 Merkle 证明

**公式：**

```python
# 步骤 1：计算各个叶子哈希
TapLeaf_A = Tagged_Hash("TapLeaf", 0xc0 + len(script_A) + script_A)
TapLeaf_B = Tagged_Hash("TapLeaf", 0xc0 + len(script_B) + script_B)

# 步骤 2：字典序排序（关键！）
if TapLeaf_A < TapLeaf_B:
    sorted_leaves = TapLeaf_A + TapLeaf_B
else:
    sorted_leaves = TapLeaf_B + TapLeaf_A

# 步骤 3：计算 Merkle 根
Merkle_Root = Tagged_Hash("TapBranch", sorted_leaves)

# 步骤 4：计算输出密钥
tweak = Tagged_Hash("TapTweak", internal_pubkey + Merkle_Root)
output_key = internal_pubkey + tweak × G
```

### 带 Merkle 证明的控制块（65 字节）

对于**双叶子树**，控制块从 33 字节扩展到**65 字节**以包含 Merkle 证明。

**控制块结构（65 字节）：**

```
┌─────────┬──────────────────────────────────┬──────────────────────────────────┐
│ 字节 1  │         字节 2-33               │         字节 34-65              │
├─────────┼──────────────────────────────────┼──────────────────────────────────┤
│   c0    │ 50be5fc4...126bb4d3             │ 2faaa677...105cf9df             │
├─────────┼──────────────────────────────────┼──────────────────────────────────┤
│版本+奇偶│    内部公钥                      │    兄弟 TapLeaf 哈希            │
└─────────┴──────────────────────────────────┴──────────────────────────────────┘

字节 1: c0 (版本) + 奇偶位
字节 2-33: 内部公钥 (32 字节)
字节 34-65: 兄弟节点哈希 (32 字节) - 这就是 MERKLE 证明！
```

**这为什么是 Merkle 证明？**

通过脚本路径 1（哈希锁）花费时：
- 控制块包含**脚本路径 2 的 TapLeaf 哈希**作为兄弟节点
- 验证者可以重构 Merkle 根：`Hash(Script1_leaf || Script2_leaf)`
- 这证明脚本 1 是承诺树的一部分

通过脚本路径 2（Bob 脚本）花费时：
- 控制块包含**脚本路径 1 的 TapLeaf 哈希**作为兄弟节点
- 验证者可以重构 Merkle 根：`Hash(Script1_leaf || Script2_leaf)`
- 这证明脚本 2 是承诺树的一部分

**美妙之处**：每个路径通过仅揭示其兄弟节点来证明其合法性，而不是整个树！

### 字典序排序（关键！）

**为什么排序？** 为了确保**确定性 Merkle 根**计算，无论脚本顺序如何。

**规则：**

```python
# 错误：不要直接连接
merkle_root = Hash(leaf_a + leaf_b)  # 顺序很重要！

# 正确：先排序
if leaf_a < leaf_b:
    merkle_root = Tagged_Hash("TapBranch", leaf_a + leaf_b)
else:
    merkle_root = Tagged_Hash("TapBranch", leaf_b + leaf_a)
```

**不排序**：不同的叶子顺序会产生不同的 Merkle 根，破坏地址重构。

**排序**：任何脚本顺序都会产生相同的 Merkle 根，确保一致性。

### 真实交易分析

让我们检查来自**同一双叶子地址**的两笔真实测试网交易：

**共享 Taproot 地址：**
```
tb1p93c4wxsr87p88jau7vru83zpk6xl0shf5ynmutd9x0gxwau3tngq9a4w3z
```

**交易 1：哈希脚本路径花费**
- TXID: `b61857a05852482c9d5ffbb8159fc2ba1efa3dd16fe4595f121fc35878a2e430`
- 方法：脚本路径（使用原像 "helloworld"）
- 控制块：包含 Bob 脚本的 TapLeaf 哈希

**交易 2：Bob 脚本路径花费**
- TXID: `185024daff64cea4c82f129aa9a8e97b4622899961452d1d144604e65a70cfe0`
- 方法：脚本路径（使用 Bob 的私钥签名）
- 控制块：包含哈希脚本的 TapLeaf 哈希

**关键观察**：两笔交易使用**完全相同的 Taproot 地址**，证明它们确实来自同一个双叶子脚本树！

### 控制块对比

让我们解码这些交易中的实际控制块：

**哈希脚本路径控制块（65 字节）：**
```
c0 50be5fc44ec580c387bf45df275aaa8b27e2d7716af31f10eeed357d126bb4d3
   2faaa677cb6ad6a74bf7025e4cd03d2a82c7fb8e3c277916d7751078105cf9df

结构:
├─ c0: 叶子版本 (0xc0)
├─ 50be5fc4...126bb4d3: Alice 内部公钥
└─ 2faaa677...105cf9df: Bob 脚本的 TapLeaf 哈希 (兄弟节点!)
```

**Bob 脚本路径控制块（65 字节）：**
```
c0 50be5fc44ec580c387bf45df275aaa8b27e2d7716af31f10eeed357d126bb4d3
   fe78d8523ce9603014b28739a51ef826f791aa17511e617af6dc96a8f10f659e

结构:
├─ c0: 叶子版本 (0xc0)
├─ 50be5fc4...126bb4d3: Alice 内部公钥 (相同!)
└─ fe78d852...8f10f659e: 哈希脚本的 TapLeaf 哈希 (兄弟节点!)
```

**重要观察：**
- 两个控制块使用**相同的内部公钥**
- Merkle 路径部分是**兄弟节点 TapLeaf 哈希**
- 这正是 Merkle 树结构的体现！

---

## 💻 示例代码

### 示例 1：提交阶段 - 构建双叶子 Taproot 地址

```python
from bitcoinutils.setup import setup
from bitcoinutils.keys import PrivateKey
from bitcoinutils.script import Script
import hashlib

def build_hash_lock_script(preimage):
    """构建哈希锁脚本 - 任何有原像的人都可以花费"""
    preimage_hash = hashlib.sha256(preimage.encode('utf-8')).hexdigest()
    return Script([
        'OP_SHA256',
        preimage_hash,
        'OP_EQUALVERIFY',
        'OP_TRUE'
    ])

def build_bob_p2pk_script(bob_public_key):
    """构建支付到公钥脚本 - 只有 Bob 可以花费"""
    return Script([
        bob_public_key.to_x_only_hex(),  # 32 字节 x-only 公钥
        'OP_CHECKSIG'
    ])

def create_dual_leaf_taproot():
    """
    提交阶段：构建具有两个花费条件的双叶子 Taproot 地址
    """
    setup('testnet')
    
    print("="*70)
    print("提交阶段：双叶子 Taproot 地址创建")
    print("="*70)
    print()
    
    # 步骤 1：设置密钥
    # Alice 控制内部密钥（密钥路径花费）
    alice_private = PrivateKey('cRxebG1hY6vVgS9CSLNaEbEJaXkpZvc6nFeqqGT7v6gcW7MbzKNT')
    alice_public = alice_private.get_public_key()
    
    # Bob 有自己的密钥（脚本路径 2）
    bob_private = PrivateKey('cSNdLFDf3wjx1rswNL2jKykbVkC6o56o5nYZi4FUkWKjFn2Q5DSG')
    bob_public = bob_private.get_public_key()
    
    print("步骤 1：密钥设置")
    print("-" * 70)
    print(f"Alice (内部密钥所有者):")
    print(f"  私钥: {alice_private.to_wif()}")
    print(f"  公钥:  {alice_public.to_hex()}")
    print()
    print(f"Bob (脚本路径 2 控制者):")
    print(f"  私钥: {bob_private.to_wif()}")
    print(f"  公钥:  {bob_public.to_hex()}")
    print()
    
    # 步骤 2：构建脚本 1 - 哈希锁
    preimage = "helloworld"
    hash_script = build_hash_lock_script(preimage)
    
    print("步骤 2：脚本 1 - 哈希锁")
    print("-" * 70)
    print(f"原像（秘密）: {preimage}")
    print(f"SHA256 哈希:       {hashlib.sha256(preimage.encode()).hexdigest()}")
    print(f"脚本:            {hash_script}")
    print(f"序列化:        {hash_script.to_hex()}")
    print()
    
    # 步骤 3：构建脚本 2 - Bob 的 P2PK
    bob_script = build_bob_p2pk_script(bob_public)
    
    print("步骤 3：脚本 2 - Bob 的 P2PK")
    print("-" * 70)
    print(f"Bob 的 x-only 公钥: {bob_public.to_x_only_hex()}")
    print(f"脚本:              {bob_script}")
    print(f"序列化:          {bob_script.to_hex()}")
    print()
    
    # 步骤 4：构建双叶子脚本树
    # 重要：使用平面列表结构，不要嵌套
    all_leafs = [hash_script, bob_script]
    
    print("步骤 4：脚本树结构")
    print("-" * 70)
    print("平面结构: [hash_script, bob_script]")
    print(f"  索引 0: 哈希锁脚本")
    print(f"  索引 1: Bob 的 P2PK 脚本")
    print()
    
    # 步骤 5：生成 Taproot 地址
    taproot_address = alice_public.get_taproot_address(all_leafs)
    
    print("步骤 5：Taproot 地址已生成")
    print("-" * 70)
    print(f"地址: {taproot_address.to_string()}")
    print(f"类型:    Bech32m (tb1p...)")
    print()
    
    # 步骤 6：理解承诺的内容
    print("="*70)
    print("此地址中承诺的内容")
    print("="*70)
    print()
    print("Merkle 根承诺到:")
    print("  ✓ 哈希锁脚本 (脚本路径 1)")
    print("  ✓ Bob 的 P2PK 脚本 (脚本路径 2)")
    print("  ✓ 内部密钥 (Alice 的密钥，用于密钥路径)")
    print()
    print("外部观察者看到:")
    print("  • 只是一个普通的 Taproot 地址")
    print("  • 无法判断存在多少个脚本路径")
    print("  • 无法判断条件是什么")
    print()
    print("从此地址花费的三种方式:")
    print("  1. 密钥路径: Alice 使用内部密钥签名（最大隐私）")
    print("  2. 脚本路径 1: 任何有 'helloworld' 原像的人")
    print("  3. 脚本路径 2: Bob 使用他的密钥签名")
    print()
    
    return taproot_address, all_leafs, alice_private, bob_private

# 执行提交阶段
address, leafs, alice_key, bob_key = create_dual_leaf_taproot()
```

### 示例 2：揭示阶段 - 哈希脚本路径花费（脚本 1）

```python
from bitcoinutils.transactions import Transaction, TxInput, TxOutput, TxWitnessInput
from bitcoinutils.script import ControlBlock
from bitcoinutils.utils import to_satoshis

def hash_script_path_spending():
    """
    揭示阶段：通过哈希脚本路径花费（任何有原像的人）
    """
    setup('testnet')
    
    print("="*70)
    print("揭示阶段：哈希脚本路径花费")
    print("="*70)
    print()
    
    # 步骤 1：重建相同的脚本树（必须与提交匹配！）
    alice_private = PrivateKey('cRxebG1hY6vVgS9CSLNaEbEJaXkpZvc6nFeqqGT7v6gcW7MbzKNT')
    alice_public = alice_private.get_public_key()
    
    bob_private = PrivateKey('cSNdLFDf3wjx1rswNL2jKykbVkC6o56o5nYZi4FUkWKjFn2Q5DSG')
    bob_public = bob_private.get_public_key()
    
    # 重建相同的脚本树
    preimage = "helloworld"
    hash_script = build_hash_lock_script(preimage)
    bob_script = build_bob_p2pk_script(bob_public)
    
    all_leafs = [hash_script, bob_script]
    taproot_address = alice_public.get_taproot_address(all_leafs)
    
    print("步骤 1：重新创建脚本树")
    print("-" * 70)
    print(f"Taproot 地址: {taproot_address.to_string()}")
    print("(必须与提交阶段完全匹配！)")
    print()
    
    # 步骤 2：构建交易
    # 此 UTXO 来自资助地址
    commit_txid = "f02c055369812944390ca6a232190ec0db83e4b1b623c452a269408bf8282d66"
    input_amount = 0.00001200  # 1200 sats
    output_amount = 0.00001034  # 1034 sats (166 sats 手续费)
    
    print("步骤 2：交易详情")
    print("-" * 70)
    print(f"输入:  {input_amount} BTC ({int(input_amount * 100_000_000)} sats)")
    print(f"输出: {output_amount} BTC ({int(output_amount * 100_000_000)} sats)")
    print(f"手续费:    {input_amount - output_amount} BTC ({int((input_amount - output_amount) * 100_000_000)} sats)")
    print()
    
    txin = TxInput(commit_txid, 0)
    txout = TxOutput(
        to_satoshis(output_amount),
        alice_public.get_taproot_address().to_script_pub_key()
    )
    tx = Transaction([txin], [txout], has_segwit=True)
    
    # 步骤 3：为哈希脚本（索引 0）构建控制块
    # 关键：控制块包含兄弟节点哈希（Bob 的脚本）
    control_block = ControlBlock(
        alice_public,           # 内部公钥
        all_leafs,              # 脚本树结构
        0,                      # 哈希脚本索引（我们要花费这个）
        is_odd=taproot_address.is_odd()
    )
    
    print("步骤 3：控制块构建")
    print("-" * 70)
    print(f"控制块: {control_block.to_hex()}")
    print(f"长度:        {len(bytes.fromhex(control_block.to_hex()))} 字节")
    print()
    
    # 解码控制块
    cb_hex = control_block.to_hex()
    print("控制块分解:")
    print(f"  字节 1:      {cb_hex[:2]} (版本 + 奇偶位)")
    print(f"  字节 2-33:  {cb_hex[2:66]} (内部公钥)")
    print(f"  字节 34-65: {cb_hex[66:]} (兄弟节点: Bob 脚本的 TapLeaf 哈希)")
    print()
    print("⚡ 兄弟节点哈希证明此脚本是树的一部分！")
    print()
    
    # 步骤 4：准备见证数据
    preimage_hex = preimage.encode('utf-8').hex()
    
    print("步骤 4：见证栈构建")
    print("-" * 70)
    print("见证栈 (从底到顶):")
    print(f"  [0] 原像:      {preimage_hex}")
    print(f"  [1] 哈希脚本:   {hash_script.to_hex()}")
    print(f"  [2] 控制块: {control_block.to_hex()}")
    print()
    
    # 步骤 5：构建见证
    witness = TxWitnessInput([
        preimage_hex,           # [0] 脚本输入：秘密
        hash_script.to_hex(),   # [1] 揭示的脚本
        control_block.to_hex()  # [2] Merkle 证明
    ])
    
    tx.witnesses.append(witness)
    
    # 步骤 6：最终交易
    print("="*70)
    print("最终交易（哈希脚本路径）")
    print("="*70)
    print(f"交易 ID:  {tx.get_txid()}")
    print(f"大小:            {tx.get_size()} 字节")
    print(f"虚拟大小:    {tx.get_vsize()} vbytes")
    print()
    
    print("🎯 揭示的内容:")
    print("   ✓ 哈希锁脚本")
    print("   ✓ 原像 'helloworld'")
    print("   ✓ 内部公钥（通过控制块）")
    print("   ✓ Bob 脚本的 TapLeaf 哈希（作为兄弟节点）")
    print()
    print("🔒 保持隐藏的内容:")
    print("   • Bob 脚本的实际内容（只揭示了哈希）")
    print("   • 密钥路径花费是否可能")
    print("   • 可能存在多少其他脚本")
    
    return tx

# 执行哈希脚本路径花费
tx_hash = hash_script_path_spending()
```

### 示例 3：揭示阶段 - Bob 脚本路径花费（脚本 2）

```python
def bob_script_path_spending():
    """
    揭示阶段：通过 Bob 脚本路径花费（需要 Bob 的签名）
    """
    setup('testnet')
    
    print("="*70)
    print("揭示阶段：Bob 脚本路径花费")
    print("="*70)
    print()
    
    # 步骤 1：相同的脚本树构建
    alice_private = PrivateKey('cRxebG1hY6vVgS9CSLNaEbEJaXkpZvc6nFeqqGT7v6gcW7MbzKNT')
    alice_public = alice_private.get_public_key()
    
    bob_private = PrivateKey('cSNdLFDf3wjx1rswNL2jKykbVkC6o56o5nYZi4FUkWKjFn2Q5DSG')
    bob_public = bob_private.get_public_key()
    
    # 重建脚本树
    preimage_hash = hashlib.sha256("helloworld".encode('utf-8')).hexdigest()
    hash_script = Script(['OP_SHA256', preimage_hash, 'OP_EQUALVERIFY', 'OP_TRUE'])
    bob_script = build_bob_p2pk_script(bob_public)
    
    all_leafs = [hash_script, bob_script]
    taproot_address = alice_public.get_taproot_address(all_leafs)
    
    print("步骤 1：重新创建脚本树")
    print("-" * 70)
    print(f"Taproot 地址: {taproot_address.to_string()}")
    print("(与哈希脚本路径相同的地址！)")
    print()
    
    # 步骤 2：构建交易
    commit_txid = "8caddfad76a5b3a8595a522e24305dc20580ca868ef733493e308ada084a050c"
    input_amount = 0.00001111  # 1111 sats
    output_amount = 0.00000900  # 900 sats (211 sats 手续费)
    
    print("步骤 2：交易详情")
    print("-" * 70)
    print(f"输入:  {input_amount} BTC ({int(input_amount * 100_000_000)} sats)")
    print(f"输出: {output_amount} BTC ({int(output_amount * 100_000_000)} sats)")
    print(f"手续费:    {input_amount - output_amount} BTC")
    print()
    
    txin = TxInput(commit_txid, 1)
    txout = TxOutput(
        to_satoshis(output_amount),
        bob_public.get_taproot_address().to_script_pub_key()
    )
    tx = Transaction([txin], [txout], has_segwit=True)
    
    # 步骤 3：为 Bob 脚本（索引 1）构建控制块
    # 关键：控制块包含兄弟节点哈希（哈希脚本）
    control_block = ControlBlock(
        alice_public,           # 内部公钥（与之前相同！）
        all_leafs,              # 脚本树结构
        1,                      # Bob 脚本索引（我们要花费这个）
        is_odd=taproot_address.is_odd()
    )
    
    print("步骤 3：控制块构建")
    print("-" * 70)
    print(f"控制块: {control_block.to_hex()}")
    print(f"长度:        {len(bytes.fromhex(control_block.to_hex()))} 字节")
    print()
    
    # 解码控制块
    cb_hex = control_block.to_hex()
    print("控制块分解:")
    print(f"  字节 1:      {cb_hex[:2]} (版本 + 奇偶位)")
    print(f"  字节 2-33:  {cb_hex[2:66]} (内部公钥 - 与哈希脚本相同！)")
    print(f"  字节 34-65: {cb_hex[66:]} (兄弟节点: 哈希脚本的 TapLeaf 哈希)")
    print()
    
    # 步骤 4：签名交易（Bob 的签名）
    sig = bob_private.sign_taproot_input(
        tx, 0,
        [taproot_address.to_script_pub_key()],
        [to_satoshis(input_amount)],
        script_path=True,           # 脚本路径花费
        tapleaf_script=bob_script,  # 单数！我们要执行的脚本
        tweak=False                 # 脚本路径不需要调整
    )
    
    print("步骤 4：Bob 的 Schnorr 签名")
    print("-" * 70)
    print(f"签名: {sig.hex()}")
    print(f"长度:    {len(bytes.fromhex(sig.hex()))} 字节 (64 字节)")
    print()
    
    # 步骤 5：构建见证
    print("步骤 5：见证栈构建")
    print("-" * 70)
    print("见证栈 (从底到顶):")
    print(f"  [0] Bob 的签名: {sig.hex()}")
    print(f"  [1] Bob 脚本:      {bob_script.to_hex()}")
    print(f"  [2] 控制块:   {control_block.to_hex()}")
    print()
    
    witness = TxWitnessInput([
        sig,                        # [0] Bob 的签名
        bob_script.to_hex(),        # [1] 揭示的脚本
        control_block.to_hex()      # [2] Merkle 证明
    ])
    
    tx.witnesses.append(witness)
    
    # 步骤 6：最终交易
    print("="*70)
    print("最终交易（BOB 脚本路径）")
    print("="*70)
    print(f"交易 ID:  {tx.get_txid()}")
    print(f"大小:            {tx.get_size()} 字节")
    print(f"虚拟大小:    {tx.get_vsize()} vbytes")
    print()
    
    print("🎯 揭示的内容:")
    print("   ✓ Bob 的 P2PK 脚本")
    print("   ✓ Bob 的 Schnorr 签名")
    print("   ✓ 内部公钥（通过控制块）")
    print("   ✓ 哈希脚本的 TapLeaf 哈希（作为兄弟节点）")
    print()
    print("🔒 保持隐藏的内容:")
    print("   • 哈希脚本的实际内容（只揭示了哈希）")
    print("   • 原像 'helloworld'")
    print("   • 密钥路径花费是否可能")
    
    return tx

# 执行 Bob 脚本路径花费
tx_bob = bob_script_path_spending()
```

### 示例 4：Merkle 证明验证

```python
def verify_merkle_proof():
    """
    验证控制块正确证明脚本成员资格
    """
    print("="*70)
    print("MERKLE 证明验证")
    print("="*70)
    print()
    
    # 来自实际交易的数据
    hash_control_block = "c050be5fc44ec580c387bf45df275aaa8b27e2d7716af31f10eeed357d126bb4d32faaa677cb6ad6a74bf7025e4cd03d2a82c7fb8e3c277916d7751078105cf9df"
    hash_script_hex = "a820936a185caaa266bb9cbe981e9e05cb78cd732b0b3280eb944412bb6f8f8f07af8851"
    
    bob_control_block = "c050be5fc44ec580c387bf45df275aaa8b27e2d7716af31f10eeed357d126bb4d3fe78d8523ce9603014b28739a51ef826f791aa17511e617af6dc96a8f10f659e"
    bob_script_hex = "2084b5951609b76619a1ce7f48977b4312ebe226987166ef044bfb374ceef63af5ac"
    
    def tagged_hash(tag, data):
        """BIP 340 标记哈希"""
        tag_hash = hashlib.sha256(tag.encode()).digest()
        return hashlib.sha256(tag_hash + tag_hash + data).digest()
    
    def parse_control_block(cb_hex):
        """解析控制块组件"""
        cb_bytes = bytes.fromhex(cb_hex)
        leaf_version = cb_bytes[0] & 0xfe
        parity = cb_bytes[0] & 0x01
        internal_pubkey = cb_bytes[1:33]
        sibling_hash = cb_bytes[33:65]
        return leaf_version, parity, internal_pubkey, sibling_hash
    
    # 解析两个控制块
    hash_version, hash_parity, hash_internal_key, hash_sibling = parse_control_block(hash_control_block)
    bob_version, bob_parity, bob_internal_key, bob_sibling = parse_control_block(bob_control_block)
    
    print("步骤 1：解析控制块")
    print("-" * 70)
    print(f"哈希脚本 CB 内部密钥: {hash_internal_key.hex()}")
    print(f"Bob 脚本 CB 内部密钥:  {bob_internal_key.hex()}")
    print(f"内部密钥匹配: {hash_internal_key == bob_internal_key} ✓")
    print()
    
    # 计算 TapLeaf 哈希
    print("步骤 2：计算 TapLeaf 哈希")
    print("-" * 70)
    
    hash_script_bytes = bytes.fromhex(hash_script_hex)
    hash_tapleaf = tagged_hash("TapLeaf", 
                               bytes([hash_version]) + 
                               bytes([len(hash_script_bytes)]) + 
                               hash_script_bytes)
    
    bob_script_bytes = bytes.fromhex(bob_script_hex)
    bob_tapleaf = tagged_hash("TapLeaf",
                              bytes([bob_version]) + 
                              bytes([len(bob_script_bytes)]) + 
                              bob_script_bytes)
    
    print(f"哈希脚本 TapLeaf: {hash_tapleaf.hex()}")
    print(f"Bob 脚本 TapLeaf:  {bob_tapleaf.hex()}")
    print()
    
    # 验证兄弟节点关系
    print("步骤 3：验证兄弟节点关系")
    print("-" * 70)
    print(f"哈希 CB 的兄弟节点匹配 Bob TapLeaf: {hash_sibling.hex() == bob_tapleaf.hex()} ✓")
    print(f"Bob CB 的兄弟节点匹配哈希 TapLeaf: {bob_sibling.hex() == hash_tapleaf.hex()} ✓")
    print()
    print("这证明两个脚本都是同一树的一部分！")
    print()
    
    # 计算 Merkle 根
    print("步骤 4：计算 Merkle 根")
    print("-" * 70)
    
    # 字典序排序（关键！）
    if hash_tapleaf < bob_tapleaf:
        print("排序: 哈希 TapLeaf < Bob TapLeaf")
        merkle_root = tagged_hash("TapBranch", hash_tapleaf + bob_tapleaf)
    else:
        print("排序: Bob TapLeaf < 哈希 TapLeaf")
        merkle_root = tagged_hash("TapBranch", bob_tapleaf + hash_tapleaf)
    
    print(f"Merkle 根: {merkle_root.hex()}")
    print()
    
    # 计算调整值
    print("步骤 5：计算输出密钥调整值")
    print("-" * 70)
    tweak = tagged_hash("TapTweak", hash_internal_key + merkle_root)
    print(f"调整值: {tweak.hex()}")
    print()
    
    print("="*70)
    print("验证完成")
    print("="*70)
    print()
    print("✓ 两个控制块都包含正确的兄弟节点哈希")
    print("✓ 可以从任一路径重构 Merkle 根")
    print("✓ 这证明两个脚本都被承诺到地址")
    print("✓ 验证者可以信任任一花费路径都是合法的")

verify_merkle_proof()
```

---

## 🧪 实践任务

### 任务 1：构建你自己的双叶子合约

**目标**：创建具有自定义条件的完整双叶子 Taproot 合约。

**场景**：创建 Alice 和 Bob 之间的托管：
- **脚本 1**：时间锁 - Alice 可以在 10 个区块后收回
- **脚本 2**：Bob 可以使用他的签名立即花费

**步骤：**

1. 创建两个脚本
2. 构建双叶子树
3. 生成 Taproot 地址
4. 在测试网上资助它
5. 测试两个花费路径

**代码模板：**

```python
# 脚本 1：Alice 的时间锁定恢复
timelock_script = Script([
    alice_public.to_x_only_hex(),
    'OP_CHECKSIGVERIFY',
    10,  # 10 个区块
    'OP_CHECKSEQUENCEVERIFY'
])

# 脚本 2：Bob 的立即花费
bob_script = Script([
    bob_public.to_x_only_hex(),
    'OP_CHECKSIG'
])

# 构建树并测试！
```

**交付成果：**
- 两个花费路径的交易 ID
- 在 mempool.space 上的交易截图
- 分析：哪个路径更昂贵？为什么？

### 任务 2：调试 Merkle 证明失败

**挑战**：学生的双叶子交易失败了。找到并修复错误。

**错误代码：**

```python
# 错误代码 - 找出错误！
def broken_dual_leaf():
    # 脚本创建正确
    leafs = [script_a, script_b]
    address = internal_key.get_taproot_address(leafs)
    
    # 花费 script_a
    control_block = ControlBlock(
        internal_key,
        leafs,
        1,  # 这里错误？
        is_odd=address.is_odd()
    )
    
    witness = TxWitnessInput([
        input_data,
        script_a.to_hex(),  # 使用 script_a
        control_block.to_hex()
    ])
```

**你的任务：**
1. 识别错误（提示：索引不匹配）
2. 解释为什么它会导致 Merkle 证明失败
3. 修复并在测试网上测试
4. 解释验证者在失败时看到什么

### 任务 3：Merkle 根计算

**目标**：手动计算并验证 Merkle 根。

**给定：**
- 哈希脚本 TapLeaf: `fe78d8523ce9603014b28739a51ef826f791aa17511e617af6dc96a8f10f659e`
- Bob 脚本 TapLeaf: `2faaa677cb6ad6a74bf7025e4cd03d2a82c7fb8e3c277916d7751078105cf9df`

**任务：**

1. 对这些哈希进行字典序排序
2. 手动计算 TapBranch 哈希
3. 与实际链上数据验证
4. 解释为什么排序是必要的

**奖励**：计算调整值和输出密钥。

---

## 📘 阅读材料

### 必读材料
- **Mastering Taproot** - 第 7 章：双叶子脚本树（第 91-108 页）
- 重点：
  - Merkle 树结构（第 92-94 页）
  - TapBranch 哈希计算（第 95-97 页）
  - 带兄弟节点哈希的控制块（第 97-99 页）
  - 字典序排序（第 105 页）

### BIP 参考
- **BIP 341**：Taproot 规范
  - 章节："构建和花费 Taproot 输出"
  - 重点：Merkle 证明验证
- **BIP 340**：标记哈希函数
  - 理解 "TapLeaf" 和 "TapBranch" 标签

### 补充资源
- **Bitcoin Optech**：Taproot Merkle 树
- **可视化工具**：使用 RootScope 可视化双叶子树
- **真实示例**：分析测试网上的实际双叶子花费

---

## 💬 讨论话题

### 问题 1：Merkle 证明安全性

**"在双叶子树中，通过脚本 1 花费时，控制块揭示了脚本 2 的 TapLeaf 哈希（但不是其内容）。这如何在隐私和安全性之间取得平衡？观察者获得什么信息，什么仍然隐藏？攻击者能否利用揭示的哈希？"**

*提示：考虑哈希是单向的，思考哈希承诺的内容。*

### 问题 2：字典序排序的必要性

**"为什么必须在计算 TapBranch 哈希之前对 TapLeaf 哈希进行字典序排序？如果我们跳过排序会发生什么？设计一个场景，其中缺乏排序会导致实际问题。"**

*考虑：如果 Alice 和 Bob 以不同顺序创建树会发生什么？*

### 问题 3：隐私退化

**"比较不同花费方法的隐私：(1) 密钥路径，(2) 哈希脚本路径，(3) Bob 脚本路径。从最私密到最不私密进行排名并解释。如果你在设计合约，你会如何激励密钥路径的使用？"**

*思考：每种情况下揭示什么，什么永远保持隐藏？*

---

## 🎙️ 讲师笔记

欢迎学习双叶子树——Taproot 真正有趣的地方！

在第 6 课中，你学习了单叶子树，其中 TapLeaf 哈希直接成为 Merkle 根。**现在我们正在构建真正的 Merkle 树**，带有分支，这就是魔法发生的地方。

**本课的关键洞察：**

1. **Merkle 证明很优雅**：我们不是揭示所有脚本，而是只揭示兄弟节点哈希。验证者可以重构 Merkle 根并验证我们的脚本已被承诺。

2. **字典序排序是不可协商的**：没有它，创建同一树的不同方可能会得到不同的 Merkle 根。排序确保确定性。

3. **控制块线性增长**：33 字节（单叶子）→ 65 字节（双叶子）→ 97 字节（四叶子）。每个兄弟节点增加 32 字节。

4. **索引很重要**：构建控制块时，脚本索引必须与你实际花费的脚本匹配。第一个脚本索引 0，第二个脚本索引 1。

**常见陷阱：**

- **索引不匹配**：控制块说索引 1，但见证使用脚本 0 → 失败
- **忘记排序**：计算 TapBranch 时不排序 → 错误的 Merkle 根 → 失败
- **控制块中的兄弟节点错误**：手动构建控制块并使用错误的哈希 → Merkle 证明失败

**调试提示**：当双叶子花费失败时，检查：
1. 脚本是否与提交阶段顺序相同？
2. 控制块索引是否正确？
3. 控制块中的兄弟节点哈希是否匹配另一个脚本的 TapLeaf？

下一课，我们将扩展到**四叶子树**，这需要多级 Merkle 证明和扩展的控制块。如果你掌握了双叶子，四叶子将非常简单！

花时间理解 Merkle 证明验证代码。理解验证者如何重构树对于调试和欣赏 Taproot 的隐私保证至关重要。

在 Discord 上见！

—Aaron

---

## 📌 关键要点

✅ 双叶子树使用带 TapBranch 哈希的真正 Merkle 结构  
✅ 控制块扩展到 65 字节（添加 32 字节兄弟节点哈希）  
✅ TapLeaf 哈希的字典序排序确保确定性 Merkle 根  
✅ 每个花费路径只揭示其兄弟节点的哈希，而不是内容  
✅ Merkle 证明允许在不揭示未使用脚本的情况下进行验证  
✅ 脚本索引（0 或 1）必须与正在花费的脚本匹配  
✅ 两个花费路径使用相同的 Taproot 地址  
✅ 相同的内部密钥出现在两个控制块中

---

**下一课**：[第 8 课：四叶子 Taproot 脚本树 →](./lesson-08-four-leaf-taptree.md)

---

*本课程是 Chaincode Labs (2026) Taproot 脚本工程队列的一部分*

