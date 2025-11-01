# Lesson 8: Four-Leaf Taproot Script Tree

**Week**: 8  
**Duration**: Self-paced (estimated 6-7 hours)  
**Prerequisites**: Lesson 7 (Dual-Leaf Taproot Script Tree)

---

## 🎯 Learning Objectives

By the end of this lesson, you will be able to:

1. Build balanced four-leaf Merkle trees with two-level structure
2. Implement 2-of-2 multisig using OP_CHECKSIGADD (Tapscript innovation)
3. Construct extended control blocks (97 bytes) with multi-level Merkle proofs
4. Combine time locks (CSV) with script paths
5. Understand witness stack ordering for multisig scripts
6. Debug complex Merkle proof verification failures
7. Compare efficiency across five different spending paths

---

## 🧩 Key Concepts

### The Leap from Theory to Practice

In previous lessons, we mastered Taproot fundamentals and dual-leaf trees. However, **true enterprise-level applications require more complex logic**. Four-leaf script trees represent the mainstream complexity of current Taproot technology in practical applications.

**Why are four-leaf script trees so important?**

Most Taproot applications stop at simple key-path spending, which maximizes privacy but leaves most of Taproot's smart contract potential undeveloped. Four-leaf trees demonstrate several key capabilities:

### Real-World Application Scenarios

**Four-leaf trees enable:**

| Scenario | Description |
|----------|-------------|
| **Wallet Recovery** | Progressive access control with timelocks + multisig + emergency paths |
| **Lightning Network Channels** | Multiple cooperative closing scenarios for different participant sets |
| **Atomic Swaps** | Hash time-locked contracts with various fallback conditions |
| **Inheritance Planning** | Time-based access with multi-beneficiary options |
| **Corporate Treasury** | Tiered authorization (immediate, time-delayed, emergency, multisig) |

### Technical Advantages

1. **Selective Disclosure**: Only executed scripts are exposed; other scripts remain hidden
2. **Fee Efficiency**: Smaller than equivalent traditional multi-condition scripts
3. **Flexible Logic**: Multiple execution paths within a single commitment
4. **Privacy Preservation**: Unused paths never revealed on-chain

### Four-Leaf Merkle Tree Structure

Unlike dual-leaf trees (single branch level), **four-leaf trees have a balanced two-level Merkle structure**:

```
                    Merkle Root
                    /          \
              Branch 0        Branch 1
              /      \        /      \
         Script 0  Script 1  Script 2  Script 3
        (Hashlock)(Multisig)  (CSV)    (Simple)
```

**Tree Building Formula:**

```python
# Level 1: Calculate individual leaf hashes
TapLeaf_0 = Tagged_Hash("TapLeaf", 0xc0 + len(script0) + script0)
TapLeaf_1 = Tagged_Hash("TapLeaf", 0xc0 + len(script1) + script1)
TapLeaf_2 = Tagged_Hash("TapLeaf", 0xc0 + len(script2) + script2)
TapLeaf_3 = Tagged_Hash("TapLeaf", 0xc0 + len(script3) + script3)

# Level 2: Calculate branch hashes (sort within each pair)
Branch_0 = Tagged_Hash("TapBranch", sorted([TapLeaf_0, TapLeaf_1]))
Branch_1 = Tagged_Hash("TapBranch", sorted([TapLeaf_2, TapLeaf_3]))

# Level 3: Calculate Merkle root
Merkle_Root = Tagged_Hash("TapBranch", sorted([Branch_0, Branch_1]))

# Final: Calculate output key
tweak = Tagged_Hash("TapTweak", internal_pubkey + Merkle_Root)
output_key = internal_pubkey + tweak × G
```

### Real Case Study: Complete Validation on Testnet

Let's analyze an actual four-leaf tree validated on testnet:

**Shared Taproot Address:**
```
tb1pjfdm902y2adr08qnn4tahxjvp6x5selgmvzx63yfqk2hdey02yvqjcr29q
```

**Feature:** Five different spending methods using the same address!

**Four Script Paths:**

1. **Script 0 (SHA256 Hashlock)**: Anyone who knows "helloworld" can spend
   - Implements hashlock pattern for atomic swaps
   - Witness: `[preimage, script, control_block]`

2. **Script 1 (2-of-2 Multisig)**: Requires cooperation between Alice and Bob
   - Uses Tapscript's efficient **OP_CHECKSIGADD** (not OP_CHECKMULTISIG!)
   - Witness: `[bob_sig, alice_sig, script, control_block]`

3. **Script 2 (CSV Timelock)**: Bob can spend after waiting 2 blocks
   - Implements relative timelock
   - Witness: `[bob_sig, script, control_block]`
   - **Critical**: Transaction input must set custom sequence value

4. **Script 3 (Simple Signature)**: Bob can spend immediately with signature
   - Simplest script path
   - Witness: `[bob_sig, script, control_block]`

5. **Key Path**: Alice uses tweaked private key for maximum privacy
   - Looks like ordinary single-signature transaction
   - Witness: `[alice_sig]` (only 64 bytes!)

### OP_CHECKSIGADD: The Tapscript Innovation

Traditional Bitcoin multisig uses **OP_CHECKMULTISIG**, which has several problems:
- Complex stack operations
- Off-by-one errors (must push dummy value)
- Checks all signature combinations (inefficient)
- Requires 33-byte compressed public keys

**Tapscript introduces OP_CHECKSIGADD**, which:
- ✅ Uses simple counter mechanism
- ✅ Verifies signatures one-by-one, stops on failure
- ✅ Supports 32-byte x-only public keys natively
- ✅ Cleaner stack operations, no dummy values

**OP_CHECKSIGADD Script Pattern:**

```python
Script([
    "OP_0",                      # Initialize counter to 0
    alice_pub.to_x_only_hex(),   # Alice's 32-byte pubkey
    "OP_CHECKSIGADD",            # Verify Alice, increment counter
    bob_pub.to_x_only_hex(),     # Bob's 32-byte pubkey
    "OP_CHECKSIGADD",            # Verify Bob, increment counter
    "OP_2",                      # Required signature count
    "OP_EQUAL"                   # Check counter == 2
])
```

**How it works:**
1. Counter starts at 0
2. For each signature: verify and increment counter if valid
3. Compare final counter with required count
4. Script succeeds if counter == required

### Extended Control Block (97 bytes)

For **four-leaf trees**, the control block expands to **97 bytes** to include **two-level Merkle proof**:

```
┌─────┬────────────────────┬────────────────────┬────────────────────┐
│Byte │    Bytes 2-33      │    Bytes 34-65     │    Bytes 66-97     │
│  1  │                    │                    │                    │
├─────┼────────────────────┼────────────────────┼────────────────────┤
│ c0  │ Internal Pubkey    │ Sibling 1 Hash     │ Sibling 2 Hash     │
│     │    (32 bytes)      │   (32 bytes)       │   (32 bytes)       │
└─────┴────────────────────┴────────────────────┴────────────────────┘

Byte 1: Leaf version (c0) + parity bit
Bytes 2-33: Internal public key
Bytes 34-65: First sibling node hash (same level)
Bytes 66-97: Second sibling node hash (parent level)
```

**Merkle Proof Paths for Each Script:**

```python
# When spending Script 0 (Hashlock):
# Control Block contains:
#   - Sibling 1: Script 1's TapLeaf hash (same branch)
#   - Sibling 2: Branch 1's TapBranch hash (parent level)

# When spending Script 1 (Multisig):
# Control Block contains:
#   - Sibling 1: Script 0's TapLeaf hash
#   - Sibling 2: Branch 1's TapBranch hash

# When spending Script 2 (CSV):
# Control Block contains:
#   - Sibling 1: Script 3's TapLeaf hash
#   - Sibling 2: Branch 0's TapBranch hash

# When spending Script 3 (Simple Sig):
# Control Block contains:
#   - Sibling 1: Script 2's TapLeaf hash
#   - Sibling 2: Branch 0's TapBranch hash
```

**The beauty:** Each script proves its legitimacy by revealing only its sibling at each level, not the entire tree!

### Control Block Size Comparison

| Tree Type | Control Block Size | Structure |
|-----------|-------------------|-----------|
| **Single-leaf** | 33 bytes | `[version+parity] + [internal_pubkey]` |
| **Dual-leaf** | 65 bytes | `+ [sibling_hash]` |
| **Four-leaf** | 97 bytes | `+ [sibling_hash] + [parent_sibling_hash]` |
| **Eight-leaf** | 129 bytes | `+ [sibling] + [parent_sibling] + [grandparent_sibling]` |

As script tree depth increases, control block grows **linearly** (32 bytes per level), but it's still much more efficient than traditional multisig scripts.

---

## 💻 Example Code

### Example 1: Building the Four-Leaf Taproot Tree

```python
from bitcoinutils.setup import setup
from bitcoinutils.keys import PrivateKey
from bitcoinutils.script import Script
from bitcoinutils.utils import Sequence, TYPE_RELATIVE_TIMELOCK
import hashlib

def build_four_leaf_taproot():
    """
    Commit Phase: Build four-leaf Taproot tree with multiple conditions
    """
    setup('testnet')
    
    print("="*70)
    print("COMMIT PHASE: Four-Leaf Taproot Tree Construction")
    print("="*70)
    print()
    
    # Step 1: Set up keys
    alice_priv = PrivateKey.from_wif("cT3tJP7BjwL25nQ9rHQuSCLugr3Vs5XfFKsTs7j5gHDgULyMmm1y")
    bob_priv = PrivateKey.from_wif("cSNdLFDf3wjx1rswNL2jKykbVkC6o56o5nYZi4FUkWKjFn2Q5DSG")
    alice_pub = alice_priv.get_public_key()
    bob_pub = bob_priv.get_public_key()
    
    print("Step 1: Key Setup")
    print("-" * 70)
    print(f"Alice (Internal Key): {alice_pub.to_hex()}")
    print(f"Bob (Script Paths):   {bob_pub.to_hex()}")
    print()
    
    # Step 2: Build Script 0 - SHA256 Hashlock
    preimage = "helloworld"
    hash0 = hashlib.sha256(preimage.encode('utf-8')).hexdigest()
    script0 = Script([
        'OP_SHA256',
        hash0,
        'OP_EQUALVERIFY',
        'OP_TRUE'
    ])
    
    print("Step 2: Script 0 - SHA256 Hashlock")
    print("-" * 70)
    print(f"Preimage: {preimage}")
    print(f"Hash:     {hash0}")
    print(f"Script:   {script0.to_hex()}")
    print()
    
    # Step 3: Build Script 1 - 2-of-2 Multisig (OP_CHECKSIGADD)
    script1 = Script([
        "OP_0",                      # Initialize counter
        alice_pub.to_x_only_hex(),   # Alice's x-only pubkey
        "OP_CHECKSIGADD",            # Verify + increment
        bob_pub.to_x_only_hex(),     # Bob's x-only pubkey
        "OP_CHECKSIGADD",            # Verify + increment
        "OP_2",                      # Required count
        "OP_EQUAL"                   # Check equality
    ])
    
    print("Step 3: Script 1 - 2-of-2 Multisig (OP_CHECKSIGADD)")
    print("-" * 70)
    print(f"Script: {script1.to_hex()}")
    print("⚡ Uses OP_CHECKSIGADD, not OP_CHECKMULTISIG!")
    print()
    
    # Step 4: Build Script 2 - CSV Timelock
    relative_blocks = 2
    seq = Sequence(TYPE_RELATIVE_TIMELOCK, relative_blocks)
    script2 = Script([
        seq.for_script(),            # Push sequence value
        "OP_CHECKSEQUENCEVERIFY",    # Verify relative timelock
        "OP_DROP",                   # Clean stack
        bob_pub.to_x_only_hex(),     # Bob's pubkey
        "OP_CHECKSIG"                # Verify signature
    ])
    
    print("Step 4: Script 2 - CSV Timelock (2 blocks)")
    print("-" * 70)
    print(f"Timelock: {relative_blocks} blocks")
    print(f"Script:   {script2.to_hex()}")
    print()
    
    # Step 5: Build Script 3 - Simple Signature
    script3 = Script([
        bob_pub.to_x_only_hex(),
        "OP_CHECKSIG"
    ])
    
    print("Step 5: Script 3 - Simple Signature")
    print("-" * 70)
    print(f"Script: {script3.to_hex()}")
    print()
    
    # Step 6: Build tree structure (nested list format)
    # [[left branch], [right branch]]
    tree = [[script0, script1], [script2, script3]]
    
    print("Step 6: Tree Structure")
    print("-" * 70)
    print("Merkle Tree:")
    print("              Root")
    print("             /    \\")
    print("       Branch0    Branch1")
    print("        /  \\        /  \\")
    print("    S0    S1      S2    S3")
    print("   Hash  Multi   CSV   Sig")
    print()
    
    # Step 7: Generate Taproot address
    taproot_address = alice_pub.get_taproot_address(tree)
    
    print("Step 7: Taproot Address Generated")
    print("-" * 70)
    print(f"Address: {taproot_address.to_string()}")
    print()
    
    print("="*70)
    print("FIVE WAYS TO SPEND FROM THIS ADDRESS")
    print("="*70)
    print("1. Key Path: Alice signs with internal key (max privacy)")
    print("2. Script 0: Anyone with 'helloworld' preimage")
    print("3. Script 1: Alice + Bob multisig (OP_CHECKSIGADD)")
    print("4. Script 2: Bob after 2 blocks (CSV timelock)")
    print("5. Script 3: Bob immediate signature")
    print()
    
    return taproot_address, tree, alice_priv, bob_priv

# Execute tree construction
address, tree, alice_key, bob_key = build_four_leaf_taproot()
```

### Example 2: Script 0 - Hashlock Path Spending

```python
from bitcoinutils.transactions import Transaction, TxInput, TxOutput, TxWitnessInput
from bitcoinutils.script import ControlBlock
from bitcoinutils.utils import to_satoshis

def spend_hashlock_path(tree, alice_pub, taproot_address):
    """
    Script Path 0: Hashlock spending with preimage
    """
    setup('testnet')
    
    print("="*70)
    print("SCRIPT PATH 0: Hashlock Spending")
    print("="*70)
    print()
    
    # UTXO information
    commit_txid = "245563c5aa4c6d32fc34eed2f182b5ed76892d13370f067dc56f34616b66c468"
    input_amount = 0.00001200  # 1200 sats
    output_amount = 0.00000666  # 666 sats
    
    print("Transaction Details:")
    print("-" * 70)
    print(f"Input:  {input_amount} BTC")
    print(f"Output: {output_amount} BTC")
    print(f"Fee:    {input_amount - output_amount} BTC")
    print()
    
    # Build transaction
    txin = TxInput(commit_txid, 0)
    txout = TxOutput(
        to_satoshis(output_amount),
        alice_pub.get_taproot_address().to_script_pub_key()
    )
    tx = Transaction([txin], [txout], has_segwit=True)
    
    # Construct Control Block for Script 0
    control_block = ControlBlock(
        alice_pub,
        tree,
        0,  # Script index 0 (hashlock)
        is_odd=taproot_address.is_odd()
    )
    
    print("Control Block Construction:")
    print("-" * 70)
    print(f"Length: {len(bytes.fromhex(control_block.to_hex()))} bytes")
    print(f"Script Index: 0 (Hashlock)")
    print()
    
    # Decode control block
    cb_bytes = bytes.fromhex(control_block.to_hex())
    print("Control Block Breakdown:")
    print(f"  Byte 1:      Version + parity")
    print(f"  Bytes 2-33:  Internal pubkey (Alice)")
    print(f"  Bytes 34-65: Sibling 1 (Script 1 TapLeaf)")
    print(f"  Bytes 66-97: Sibling 2 (Branch 1 TapBranch)")
    print()
    
    # Build witness
    preimage = "helloworld"
    preimage_hex = preimage.encode('utf-8').hex()
    script0 = tree[0][0]  # First script in left branch
    
    witness = TxWitnessInput([
        preimage_hex,
        script0.to_hex(),
        control_block.to_hex()
    ])
    
    tx.witnesses.append(witness)
    
    print("Witness Stack:")
    print("-" * 70)
    print(f"[0] Preimage: {preimage_hex}")
    print(f"[1] Script:   {script0.to_hex()}")
    print(f"[2] Control:  {control_block.to_hex()[:40]}...")
    print()
    
    print(f"Transaction ID: {tx.get_txid()}")
    print(f"Size: {tx.get_size()} bytes")
    print(f"vSize: {tx.get_vsize()} vbytes")
    
    return tx

# Execute hashlock spending
# tx_hash = spend_hashlock_path(tree, alice_pub, address)
```

### Example 3: Script 1 - Multisig Path (OP_CHECKSIGADD)

```python
def spend_multisig_path(tree, alice_priv, bob_priv, taproot_address):
    """
    Script Path 1: 2-of-2 Multisig with OP_CHECKSIGADD
    """
    setup('testnet')
    
    print("="*70)
    print("SCRIPT PATH 1: Multisig (OP_CHECKSIGADD)")
    print("="*70)
    print()
    
    alice_pub = alice_priv.get_public_key()
    bob_pub = bob_priv.get_public_key()
    
    # UTXO information
    commit_txid = "1ed5a3e97a6d3bc0493acc2aac15011cd99000b52e932724766c3d277d76daac"
    input_amount = 0.00001400  # 1400 sats
    output_amount = 0.00000668  # 668 sats
    
    print("Transaction Details:")
    print("-" * 70)
    print(f"Input:  {input_amount} BTC")
    print(f"Output: {output_amount} BTC")
    print()
    
    # Build transaction
    txin = TxInput(commit_txid, 0)
    txout = TxOutput(
        to_satoshis(output_amount),
        alice_pub.get_taproot_address().to_script_pub_key()
    )
    tx = Transaction([txin], [txout], has_segwit=True)
    
    # Control Block for Script 1
    control_block = ControlBlock(
        alice_pub,
        tree,
        1,  # Script index 1 (multisig)
        is_odd=taproot_address.is_odd()
    )
    
    script1 = tree[0][1]  # Second script in left branch
    
    # Sign with both keys (Script Path mode)
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
    
    print("Signatures Generated:")
    print("-" * 70)
    print(f"Alice: {sig_alice.hex()[:40]}...")
    print(f"Bob:   {sig_bob.hex()[:40]}...")
    print()
    
    # CRITICAL: Witness stack order matters!
    # Bob signature first, consumed second
    # Alice signature second, consumed first
    witness = TxWitnessInput([
        sig_bob,                  # Stack bottom, consumed second
        sig_alice,                # Stack top, consumed first
        script1.to_hex(),
        control_block.to_hex()
    ])
    
    tx.witnesses.append(witness)
    
    print("⚠️  WITNESS STACK ORDER IS CRITICAL:")
    print("-" * 70)
    print("[0] Bob's signature   (consumed SECOND by OP_CHECKSIGADD)")
    print("[1] Alice's signature (consumed FIRST by OP_CHECKSIGADD)")
    print("[2] Script")
    print("[3] Control Block")
    print()
    
    print(f"Transaction ID: {tx.get_txid()}")
    
    return tx

# Execute multisig spending
# tx_multi = spend_multisig_path(tree, alice_key, bob_key, address)
```

### Example 4: OP_CHECKSIGADD Stack Execution Visualization

```python
def visualize_op_checksigadd_execution():
    """
    Visualize the complete stack execution for OP_CHECKSIGADD multisig
    """
    print("="*70)
    print("OP_CHECKSIGADD STACK EXECUTION TRACE")
    print("="*70)
    print()
    
    print("Script: OP_0 [Alice_PubKey] OP_CHECKSIGADD [Bob_PubKey] OP_CHECKSIGADD OP_2 OP_EQUAL")
    print()
    
    print("Initial State: Witness Data on Stack")
    print("-" * 70)
    print("│ sig_alice │  ← Top, consumed FIRST")
    print("│ sig_bob   │  ← Consumed SECOND")
    print("└───────────┘")
    print()
    
    print("Step 1: OP_0 - Initialize Counter")
    print("-" * 70)
    print("│ 0         │  ← Counter = 0")
    print("│ sig_alice │")
    print("│ sig_bob   │")
    print("└───────────┘")
    print()
    
    print("Step 2: [Alice_PubKey] - Push Alice's Public Key")
    print("-" * 70)
    print("│ alice_pk  │  ← Alice's 32-byte x-only pubkey")
    print("│ 0         │  ← Counter")
    print("│ sig_alice │")
    print("│ sig_bob   │")
    print("└───────────┘")
    print()
    
    print("Step 3: OP_CHECKSIGADD - Verify Alice & Increment")
    print("-" * 70)
    print("Process:")
    print("  • Pop alice_pk")
    print("  • Pop sig_alice (from lower layer)")
    print("  • Verify: schnorr_verify(sig_alice, alice_pk, sighash)")
    print("  • Pop counter 0")
    print("  • Verification SUCCESS: push (0 + 1 = 1)")
    print()
    print("│ 1         │  ← Counter incremented to 1")
    print("│ sig_bob   │")
    print("└───────────┘")
    print()
    
    print("Step 4: [Bob_PubKey] - Push Bob's Public Key")
    print("-" * 70)
    print("│ bob_pk    │  ← Bob's 32-byte x-only pubkey")
    print("│ 1         │  ← Counter")
    print("│ sig_bob   │")
    print("└───────────┘")
    print()
    
    print("Step 5: OP_CHECKSIGADD - Verify Bob & Increment")
    print("-" * 70)
    print("Process:")
    print("  • Pop bob_pk")
    print("  • Pop sig_bob")
    print("  • Verify: schnorr_verify(sig_bob, bob_pk, sighash)")
    print("  • Pop counter 1")
    print("  • Verification SUCCESS: push (1 + 1 = 2)")
    print()
    print("│ 2         │  ← Counter incremented to 2")
    print("└───────────┘")
    print()
    
    print("Step 6: OP_2 - Push Required Signature Count")
    print("-" * 70)
    print("│ 2         │  ← Required count")
    print("│ 2         │  ← Actual count")
    print("└───────────┘")
    print()
    
    print("Step 7: OP_EQUAL - Check Equality")
    print("-" * 70)
    print("Process:")
    print("  • Pop both 2s")
    print("  • Compare: 2 == 2 is TRUE")
    print("  • Push 1 (success)")
    print()
    print("│ 1         │  ← Script SUCCESS ✓")
    print("└───────────┘")
    print()
    
    print("="*70)
    print("OP_CHECKSIGADD vs OP_CHECKMULTISIG")
    print("="*70)
    print()
    print("OP_CHECKSIGADD Advantages:")
    print("  ✓ Simple counter mechanism")
    print("  ✓ Verifies one-by-one, stops on failure")
    print("  ✓ Supports 32-byte x-only pubkeys")
    print("  ✓ No dummy value needed")
    print("  ✓ Cleaner stack operations")
    print()
    print("OP_CHECKMULTISIG Problems:")
    print("  ✗ Complex stack operations")
    print("  ✗ Must check all signature combinations")
    print("  ✗ Requires 33-byte compressed pubkeys")
    print("  ✗ Off-by-one error (dummy value)")

visualize_op_checksigadd_execution()
```

### Example 5: Control Block Verification for Four-Leaf Tree

```python
def verify_four_leaf_control_block():
    """
    Verify control block and reconstruct Merkle root for four-leaf tree
    """
    print("="*70)
    print("FOUR-LEAF CONTROL BLOCK VERIFICATION")
    print("="*70)
    print()
    
    # Real control block from testnet transaction (Script 1 - Multisig)
    cb_hex = "c050be5fc44ec580c387bf45df275aaa8b27e2d7716af31f10eeed357d126bb4d3fe78d8523ce9603014b28739a51ef826f791aa17511e617af6dc96a8f10f659eda55197526f26fa309563b7a3551ca945c046e5b7ada957e59160d4d27f299e3"
    
    cb_bytes = bytes.fromhex(cb_hex)
    
    print(f"Control Block Length: {len(cb_bytes)} bytes")
    print()
    
    # Parse control block
    version_parity = cb_bytes[0]
    leaf_version = version_parity & 0xfe
    parity = version_parity & 0x01
    internal_pubkey = cb_bytes[1:33]
    sibling_1 = cb_bytes[33:65]
    sibling_2 = cb_bytes[65:97]
    
    print("Parsed Control Block:")
    print("-" * 70)
    print(f"Leaf Version:   0x{leaf_version:02x} (Tapscript v0)")
    print(f"Parity Bit:     {parity} ({'odd' if parity else 'even'} y-coordinate)")
    print(f"Internal Key:   {internal_pubkey.hex()}")
    print(f"  → Alice's internal public key")
    print()
    print(f"Sibling 1:      {sibling_1.hex()}")
    print(f"  → Script 0 (Hashlock) TapLeaf hash")
    print()
    print(f"Sibling 2:      {sibling_2.hex()}")
    print(f"  → Branch 1 (Script 2+3) TapBranch hash")
    print()
    
    print("="*70)
    print("MERKLE PROOF VISUALIZATION")
    print("="*70)
    print()
    print("When spending Script 1 (Multisig), verifier:")
    print()
    print("1. Calculates Script 1's TapLeaf hash")
    print("   TapLeaf_1 = H('TapLeaf' || version || len || script1)")
    print()
    print("2. Uses Sibling 1 to calculate Branch 0")
    print("   Branch_0 = H('TapBranch' || sort(TapLeaf_0, TapLeaf_1))")
    print()
    print("3. Uses Sibling 2 (Branch 1) to calculate Merkle Root")
    print("   Merkle_Root = H('TapBranch' || sort(Branch_0, Branch_1))")
    print()
    print("4. Calculates tweak and output key")
    print("   tweak = H('TapTweak' || internal_key || Merkle_Root)")
    print("   output_key = internal_key + tweak × G")
    print()
    print("5. Compares output_key with address")
    print("   If match: Script 1 is proven to be part of committed tree ✓")
    print()
    
    print("🔒 What Remains Hidden:")
    print("  • Script 0's actual content (only hash revealed)")
    print("  • Script 2 and 3's actual contents")
    print("  • Whether key-path spending is possible")

verify_four_leaf_control_block()
```

---

## 🧪 Practice Tasks

### Task 1: Build Your Own Four-Leaf Enterprise Contract

**Objective**: Create a complete four-leaf Taproot tree for a corporate treasury.

**Scenario**: Design a treasury with these conditions:
- **Script 0**: CEO can spend immediately
- **Script 1**: 2-of-3 board multisig
- **Script 2**: Any board member after 30 days (CSV)
- **Script 3**: Emergency recovery with 3-of-3

**Steps:**

1. Generate keys for CEO and 3 board members
2. Build all four scripts
3. Construct the tree structure
4. Generate Taproot address
5. Fund and test at least 2 spending paths on testnet

**Code Template:**

```python
# CEO immediate spend
script0 = Script([ceo_pub.to_x_only_hex(), 'OP_CHECKSIG'])

# 2-of-3 board multisig (use OP_CHECKSIGADD)
script1 = Script([
    'OP_0',
    board1_pub.to_x_only_hex(), 'OP_CHECKSIGADD',
    board2_pub.to_x_only_hex(), 'OP_CHECKSIGADD',
    board3_pub.to_x_only_hex(), 'OP_CHECKSIGADD',
    'OP_2', 'OP_EQUAL'  # 2-of-3
])

# Design scripts 2 and 3, build tree, test!
```

**Deliverables:**
- Transaction IDs for 2+ spending paths
- Analysis of control block sizes
- Discussion: Which path would you use in emergency?

### Task 2: Debug OP_CHECKSIGADD Signature Order

**Challenge**: A student's multisig transaction is failing. Find the error.

**Broken Code:**

```python
# BROKEN CODE - Multisig fails!
def broken_multisig():
    # ... transaction setup ...
    
    sig_alice = alice_priv.sign_taproot_input(...)
    sig_bob = bob_priv.sign_taproot_input(...)
    
    # ERROR: Wrong witness order
    witness = TxWitnessInput([
        sig_alice,  # Alice first
        sig_bob,    # Bob second
        script1.to_hex(),
        control_block.to_hex()
    ])
```

**Your Task:**
1. Explain WHY the witness order is wrong
2. Trace through the stack execution showing where it fails
3. Fix the code
4. Explain the correct witness ordering for 3-of-3 multisig

**Hint:** Think about which signature OP_CHECKSIGADD consumes first.

### Task 3: Merkle Root Reconstruction

**Objective**: Manually reconstruct Merkle root from control block.

**Given:**
- Control block from Script 1 spending (97 bytes)
- Script 1 content (multisig script)

**Tasks:**

1. Parse the control block
2. Extract both sibling hashes
3. Calculate Script 1's TapLeaf hash
4. Calculate Branch 0 hash using Sibling 1
5. Calculate Merkle Root using Sibling 2
6. Verify the result matches expected address

**Bonus**: Write code to automate this verification for any script.

---

## 📘 Reading

### Required Reading
- **Mastering Taproot** - Chapter 8: Four-Leaf Script Tree (pages 109-127)
- Focus on:
  - Balanced Merkle tree structure (pages 110-113)
  - OP_CHECKSIGADD implementation (pages 117-121)
  - Extended control blocks (pages 122-124)
  - Multi-level Merkle proofs (pages 124-127)

### BIP References
- **BIP 342**: Tapscript specification
  - Section: OP_CHECKSIGADD opcode
  - Section: Signature validation rules
- **BIP 341**: Taproot specification
  - Section: Multi-level Merkle proof verification

### Supplementary Resources
- **Bitcoin Optech**: Tapscript improvements
- **OP_CHECKSIGADD Deep Dive**: Understanding the counter mechanism
- **Real Examples**: Analyze four-leaf spends on mainnet

---

## 💬 Discussion Prompts

### Question 1: OP_CHECKSIGADD Design Choice

**"Why did Tapscript introduce OP_CHECKSIGADD instead of improving OP_CHECKMULTISIG? What fundamental design philosophy does this change reflect? Could OP_CHECKSIGADD be used to implement 3-of-5 multisig? How would the script look?"**

*Hint: Think about incremental verification vs. checking all combinations.*

### Question 2: Control Block Growth

**"For a four-leaf tree, control blocks are 97 bytes. For eight-leaf (three levels), they'd be 129 bytes. At what tree depth does the control block overhead outweigh the benefits? Consider both fee costs and privacy. Design a decision framework for choosing tree depth."**

*Consider: Script complexity, number of paths, likelihood of each path.*

### Question 3: Real-World Application

**"Design a real-world four-leaf Taproot contract for a specific use case (Lightning channel, inheritance, corporate treasury, etc.). Explain: (1) Why four leaves are necessary, (2) What each script does, (3) Which path you'd expect to be used most often, (4) How privacy is preserved."**

*Be specific about scripts, timelocks, and threat models.*

---

## 🎙️ Instructor's Note

Welcome to the enterprise-grade level of Taproot development!

If you've mastered Lessons 5-7, you're ready for this. Four-leaf trees might seem complex, but they're just **two dual-leaf trees combined**. The principles are the same; the Merkle proofs are just one level deeper.

**Key insights for this lesson:**

1. **OP_CHECKSIGADD is a game-changer**: This opcode makes multisig elegant in Tapscript. No more dummy values, no off-by-one errors, and it works beautifully with x-only pubkeys.

2. **Witness ordering is critical**: For multisig, signatures must be ordered opposite to how they're consumed. The last signature in the witness is consumed first by OP_CHECKSIGADD. Get this wrong, and your transaction fails.

3. **Control blocks grow linearly**: 33 → 65 → 97 → 129 bytes as depth increases. Each level adds 32 bytes for the sibling hash at that level.

4. **Five spending paths, one address**: This lesson's case study shows five completely different ways to spend from the same address. This is Taproot's power in action.

**Common pitfalls:**

- **Wrong witness order in multisig**: Bob's sig before Alice's sig when script expects Alice first
- **Forgetting CSV sequence**: Script 2 (CSV) requires setting `sequence` on the transaction input
- **Index confusion**: With nested list structure `[[s0, s1], [s2, s3]]`, script indices are still flat: 0, 1, 2, 3

**Debugging tip**: When a four-leaf spend fails, check:
1. Is the control block exactly 97 bytes?
2. Does it contain the correct two sibling hashes?
3. Can you manually reconstruct the Merkle root?
4. For multisig: Is the witness order correct?

Next lesson, we'll explore **advanced Taproot patterns**: combining time locks with hash locks (HTLCs), atomic swaps, and Lightning Network integration. But master four-leaf trees first—they're the foundation for everything advanced.

Take your time with OP_CHECKSIGADD stack execution. Understanding how the counter increments is crucial for debugging multisig scripts.

See you in Discord!

—Aaron

---

## 📌 Key Takeaways

✅ Four-leaf trees have two-level Merkle structure (balanced binary tree)  
✅ OP_CHECKSIGADD is superior to OP_CHECKMULTISIG for Tapscript  
✅ Control blocks expand to 97 bytes (includes two sibling hashes)  
✅ Witness ordering is critical: last signature consumed first  
✅ CSV timelocks require setting `sequence` on transaction input  
✅ Five spending paths possible: 4 script paths + 1 key path  
✅ Each path reveals only its Merkle proof, not entire tree  
✅ Privacy preserved: unused paths never revealed on-chain

---

**Next Lesson**: [Lesson 9: Advanced Taproot Patterns →](./lesson-09-advanced-patterns.md)

---

*This lesson is part of the Taproot Script Engineering Cohort at Chaincode Labs (2026)*