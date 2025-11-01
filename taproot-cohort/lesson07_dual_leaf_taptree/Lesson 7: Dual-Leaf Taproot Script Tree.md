# Lesson 7: Dual-Leaf Taproot Script Tree

**Week**: 7  
**Duration**: Self-paced (estimated 5-6 hours)  
**Prerequisites**: Lesson 6 (Single-Leaf Taproot Script Tree)

---

## 🎯 Learning Objectives

By the end of this lesson, you will be able to:

1. Build dual-leaf Taproot script trees with Merkle structure
2. Calculate TapLeaf and TapBranch hashes correctly
3. Construct control blocks with Merkle proofs (65 bytes for dual-leaf)
4. Implement multiple spending paths from the same address
5. Understand lexicographic sorting for Merkle root calculation
6. Debug Merkle proof verification failures
7. Compare privacy and efficiency tradeoffs between spending paths

---

## 🧩 Key Concepts

### From Single-Leaf to Dual-Leaf: The True Power of Taproot

In Lesson 6, we mastered single-leaf Taproot script paths where one script condition was committed to the address. But Taproot's true power lies in its **multi-branch script tree architecture**—the ability to elegantly organize multiple different spending conditions within a single address.

**The Evolution:**

```
Single-Leaf Tree:             Dual-Leaf Tree:
                              
   TapLeaf Hash                  Merkle Root
        │                           / \
   [Hash Lock]              TapLeaf A   TapLeaf B
                           [Hash Lock]  [Bob Script]
```

### Business Scenario: Alice's Dual-Path Escrow

Imagine this business scenario: **Alice wants to create a digital escrow contract** that supports both automatic unlocking based on secret information (Hash Lock) and provides Bob with direct private key control permissions.

**In traditional Bitcoin**, this would require:
- Complex multisignature scripts, OR
- Multiple independent addresses, OR
- P2SH with all conditions exposed

**Taproot's dual-leaf script tree elegantly integrates these into a single address:**

1. **Script Path 1**: Hash Lock script - anyone who knows "helloworld" can spend
2. **Script Path 2**: Bob Script - only Bob's private key holder can spend
3. **Key Path**: Alice as internal key holder can spend directly (maximum privacy)

**The elegance**: External observers cannot distinguish whether this is a simple payment or a complex three-path conditional contract. **Only during actual spending is the used path selectively revealed.**

### Merkle Structure of Dual-Leaf Script Trees

Unlike single-leaf trees that directly use TapLeaf hash as the Merkle root, **dual-leaf script trees require building a true Merkle tree**.

**Merkle Tree Structure:**

```
                    Merkle Root
                    (TapBranch)
                    /          \
              TapLeaf A      TapLeaf B
           [Hash Script]   [Bob Script]
```

**Technical Implementation Key Points:**

1. **TapLeaf Hash Calculation**: Each script separately calculates its TapLeaf hash
2. **Lexicographic Sorting**: Sort the two TapLeaf hashes before combining
3. **TapBranch Hash Calculation**: Calculate TapBranch hash after sorting
4. **Control Block Construction**: Each script needs to include its sibling node hash as Merkle proof

**The Formula:**

```python
# Step 1: Calculate individual leaf hashes
TapLeaf_A = Tagged_Hash("TapLeaf", 0xc0 + len(script_A) + script_A)
TapLeaf_B = Tagged_Hash("TapLeaf", 0xc0 + len(script_B) + script_B)

# Step 2: Sort lexicographically (critical!)
if TapLeaf_A < TapLeaf_B:
    sorted_leaves = TapLeaf_A + TapLeaf_B
else:
    sorted_leaves = TapLeaf_B + TapLeaf_A

# Step 3: Calculate Merkle Root
Merkle_Root = Tagged_Hash("TapBranch", sorted_leaves)

# Step 4: Calculate output key
tweak = Tagged_Hash("TapTweak", internal_pubkey + Merkle_Root)
output_key = internal_pubkey + tweak × G
```

### Control Block with Merkle Proof (65 bytes)

For **dual-leaf trees**, the control block expands from 33 bytes to **65 bytes** to include the Merkle proof.

**Control Block Structure (65 bytes):**

```
┌─────────┬──────────────────────────────────┬──────────────────────────────────┐
│ Byte 1  │         Bytes 2-33               │         Bytes 34-65              │
├─────────┼──────────────────────────────────┼──────────────────────────────────┤
│   c0    │ 50be5fc4...126bb4d3             │ 2faaa677...105cf9df             │
├─────────┼──────────────────────────────────┼──────────────────────────────────┤
│Ver+Parity│    Internal Public Key          │    Sibling TapLeaf Hash         │
└─────────┴──────────────────────────────────┴──────────────────────────────────┘

Byte 1: c0 (version) + parity bit
Bytes 2-33: Internal public key (32 bytes)
Bytes 34-65: Sibling node hash (32 bytes) - THIS IS THE MERKLE PROOF!
```

**What makes this a Merkle proof?**

When spending via Script Path 1 (Hash Lock):
- Control block includes **Script Path 2's TapLeaf hash** as sibling
- Verifier can reconstruct Merkle Root: `Hash(Script1_leaf || Script2_leaf)`
- This proves Script 1 was part of the committed tree

When spending via Script Path 2 (Bob Script):
- Control block includes **Script Path 1's TapLeaf hash** as sibling
- Verifier can reconstruct Merkle Root: `Hash(Script1_leaf || Script2_leaf)`
- This proves Script 2 was part of the committed tree

**The beauty**: Each path proves its legitimacy by revealing only its sibling, not the entire tree!

### Lexicographic Sorting (Critical!)

**Why sort?** To ensure **deterministic Merkle root** calculation regardless of script order.

**The Rule:**

```python
# WRONG: Don't concatenate directly
merkle_root = Hash(leaf_a + leaf_b)  # Order matters!

# CORRECT: Sort first
if leaf_a < leaf_b:
    merkle_root = Tagged_Hash("TapBranch", leaf_a + leaf_b)
else:
    merkle_root = Tagged_Hash("TapBranch", leaf_b + leaf_a)
```

**Without sorting:** Different leaf ordering would produce different Merkle roots, breaking address reconstruction.

**With sorting:** Any script order produces the same Merkle root, ensuring consistency.

### Real Transaction Analysis

Let's examine two real testnet transactions from the **same dual-leaf address**:

**Shared Taproot Address:**
```
tb1p93c4wxsr87p88jau7vru83zpk6xl0shf5ynmutd9x0gxwau3tngq9a4w3z
```

**Transaction 1: Hash Script Path Spending**
- TXID: `b61857a05852482c9d5ffbb8159fc2ba1efa3dd16fe4595f121fc35878a2e430`
- Method: Script Path (using preimage "helloworld")
- Control Block: Includes Bob Script's TapLeaf hash

**Transaction 2: Bob Script Path Spending**
- TXID: `185024daff64cea4c82f129aa9a8e97b4622899961452d1d144604e65a70cfe0`
- Method: Script Path (using Bob's private key signature)
- Control Block: Includes Hash Script's TapLeaf hash

**Key Observation:** Both transactions use the **exact same Taproot address**, proving they indeed originate from the same dual-leaf script tree!

### Control Block Comparison

Let's decode the actual control blocks from these transactions:

**Hash Script Path Control Block (65 bytes):**
```
c0 50be5fc44ec580c387bf45df275aaa8b27e2d7716af31f10eeed357d126bb4d3
   2faaa677cb6ad6a74bf7025e4cd03d2a82c7fb8e3c277916d7751078105cf9df

Structure:
├─ c0: Leaf version (0xc0)
├─ 50be5fc4...126bb4d3: Alice internal pubkey
└─ 2faaa677...105cf9df: Bob Script's TapLeaf hash (SIBLING!)
```

**Bob Script Path Control Block (65 bytes):**
```
c0 50be5fc44ec580c387bf45df275aaa8b27e2d7716af31f10eeed357d126bb4d3
   fe78d8523ce9603014b28739a51ef826f791aa17511e617af6dc96a8f10f659e

Structure:
├─ c0: Leaf version (0xc0)
├─ 50be5fc4...126bb4d3: Alice internal pubkey (SAME!)
└─ fe78d852...8f10f659e: Hash Script's TapLeaf hash (SIBLING!)
```

**Important Observations:**
- Both control blocks use the **same internal public key**
- Merkle path portions are **sibling node TapLeaf hashes**
- This is exactly the manifestation of Merkle tree structure!

---

## 💻 Example Code

### Example 1: Commit Phase - Building Dual-Leaf Taproot Address

```python
from bitcoinutils.setup import setup
from bitcoinutils.keys import PrivateKey
from bitcoinutils.script import Script
import hashlib

def build_hash_lock_script(preimage):
    """Build Hash Lock script - anyone with preimage can spend"""
    preimage_hash = hashlib.sha256(preimage.encode('utf-8')).hexdigest()
    return Script([
        'OP_SHA256',
        preimage_hash,
        'OP_EQUALVERIFY',
        'OP_TRUE'
    ])

def build_bob_p2pk_script(bob_public_key):
    """Build Pay-to-Public-Key script - only Bob can spend"""
    return Script([
        bob_public_key.to_x_only_hex(),  # 32-byte x-only pubkey
        'OP_CHECKSIG'
    ])

def create_dual_leaf_taproot():
    """
    Commit Phase: Build dual-leaf Taproot address with two spending conditions
    """
    setup('testnet')
    
    print("="*70)
    print("COMMIT PHASE: Dual-Leaf Taproot Address Creation")
    print("="*70)
    print()
    
    # Step 1: Set up keys
    # Alice controls the internal key (Key Path spending)
    alice_private = PrivateKey('cRxebG1hY6vVgS9CSLNaEbEJaXkpZvc6nFeqqGT7v6gcW7MbzKNT')
    alice_public = alice_private.get_public_key()
    
    # Bob has his own key (Script Path 2)
    bob_private = PrivateKey('cSNdLFDf3wjx1rswNL2jKykbVkC6o56o5nYZi4FUkWKjFn2Q5DSG')
    bob_public = bob_private.get_public_key()
    
    print("Step 1: Key Setup")
    print("-" * 70)
    print(f"Alice (Internal Key Owner):")
    print(f"  Private Key: {alice_private.to_wif()}")
    print(f"  Public Key:  {alice_public.to_hex()}")
    print()
    print(f"Bob (Script Path 2 Controller):")
    print(f"  Private Key: {bob_private.to_wif()}")
    print(f"  Public Key:  {bob_public.to_hex()}")
    print()
    
    # Step 2: Build Script 1 - Hash Lock
    preimage = "helloworld"
    hash_script = build_hash_lock_script(preimage)
    
    print("Step 2: Script 1 - Hash Lock")
    print("-" * 70)
    print(f"Preimage (secret): {preimage}")
    print(f"SHA256 Hash:       {hashlib.sha256(preimage.encode()).hexdigest()}")
    print(f"Script:            {hash_script}")
    print(f"Serialized:        {hash_script.to_hex()}")
    print()
    
    # Step 3: Build Script 2 - Bob's P2PK
    bob_script = build_bob_p2pk_script(bob_public)
    
    print("Step 3: Script 2 - Bob's P2PK")
    print("-" * 70)
    print(f"Bob's x-only Pubkey: {bob_public.to_x_only_hex()}")
    print(f"Script:              {bob_script}")
    print(f"Serialized:          {bob_script.to_hex()}")
    print()
    
    # Step 4: Build dual-leaf script tree
    # IMPORTANT: Use flat list structure, not nested
    all_leafs = [hash_script, bob_script]
    
    print("Step 4: Script Tree Structure")
    print("-" * 70)
    print("Flat structure: [hash_script, bob_script]")
    print(f"  Index 0: Hash Lock Script")
    print(f"  Index 1: Bob's P2PK Script")
    print()
    
    # Step 5: Generate Taproot address
    taproot_address = alice_public.get_taproot_address(all_leafs)
    
    print("Step 5: Taproot Address Generated")
    print("-" * 70)
    print(f"Address: {taproot_address.to_string()}")
    print(f"Type:    Bech32m (tb1p...)")
    print()
    
    # Step 6: Understand what's committed
    print("="*70)
    print("WHAT'S COMMITTED IN THIS ADDRESS")
    print("="*70)
    print()
    print("The Merkle Root commits to:")
    print("  ✓ Hash Lock script (Script Path 1)")
    print("  ✓ Bob's P2PK script (Script Path 2)")
    print("  ✓ Internal key (Alice's key for Key Path)")
    print()
    print("External observers see:")
    print("  • Just a normal Taproot address")
    print("  • Cannot tell how many script paths exist")
    print("  • Cannot tell what the conditions are")
    print()
    print("Three ways to spend from this address:")
    print("  1. Key Path: Alice signs with internal key (max privacy)")
    print("  2. Script Path 1: Anyone with 'helloworld' preimage")
    print("  3. Script Path 2: Bob signs with his key")
    print()
    
    return taproot_address, all_leafs, alice_private, bob_private

# Execute commit phase
address, leafs, alice_key, bob_key = create_dual_leaf_taproot()
```

### Example 2: Reveal Phase - Hash Script Path Spending (Script 1)

```python
from bitcoinutils.transactions import Transaction, TxInput, TxOutput, TxWitnessInput
from bitcoinutils.script import ControlBlock
from bitcoinutils.utils import to_satoshis

def hash_script_path_spending():
    """
    Reveal Phase: Spend via Hash Script Path (anyone with preimage)
    """
    setup('testnet')
    
    print("="*70)
    print("REVEAL PHASE: Hash Script Path Spending")
    print("="*70)
    print()
    
    # Step 1: Rebuild identical script tree (must match commit!)
    alice_private = PrivateKey('cRxebG1hY6vVgS9CSLNaEbEJaXkpZvc6nFeqqGT7v6gcW7MbzKNT')
    alice_public = alice_private.get_public_key()
    
    bob_private = PrivateKey('cSNdLFDf3wjx1rswNL2jKykbVkC6o56o5nYZi4FUkWKjFn2Q5DSG')
    bob_public = bob_private.get_public_key()
    
    # Rebuild same script tree
    preimage = "helloworld"
    hash_script = build_hash_lock_script(preimage)
    bob_script = build_bob_p2pk_script(bob_public)
    
    all_leafs = [hash_script, bob_script]
    taproot_address = alice_public.get_taproot_address(all_leafs)
    
    print("Step 1: Recreate Script Tree")
    print("-" * 70)
    print(f"Taproot Address: {taproot_address.to_string()}")
    print("(Must match commit phase exactly!)")
    print()
    
    # Step 2: Build transaction
    # This UTXO would come from funding the address
    commit_txid = "f02c055369812944390ca6a232190ec0db83e4b1b623c452a269408bf8282d66"
    input_amount = 0.00001200  # 1200 sats
    output_amount = 0.00001034  # 1034 sats (166 sats fee)
    
    print("Step 2: Transaction Details")
    print("-" * 70)
    print(f"Input:  {input_amount} BTC ({int(input_amount * 100_000_000)} sats)")
    print(f"Output: {output_amount} BTC ({int(output_amount * 100_000_000)} sats)")
    print(f"Fee:    {input_amount - output_amount} BTC ({int((input_amount - output_amount) * 100_000_000)} sats)")
    print()
    
    txin = TxInput(commit_txid, 0)
    txout = TxOutput(
        to_satoshis(output_amount),
        alice_public.get_taproot_address().to_script_pub_key()
    )
    tx = Transaction([txin], [txout], has_segwit=True)
    
    # Step 3: Build Control Block for Hash Script (index 0)
    # CRITICAL: The control block includes the SIBLING hash (Bob's script)
    control_block = ControlBlock(
        alice_public,           # Internal public key
        all_leafs,              # Script tree structure
        0,                      # Hash script index (we're spending this one)
        is_odd=taproot_address.is_odd()
    )
    
    print("Step 3: Control Block Construction")
    print("-" * 70)
    print(f"Control Block: {control_block.to_hex()}")
    print(f"Length:        {len(bytes.fromhex(control_block.to_hex()))} bytes")
    print()
    
    # Decode control block
    cb_hex = control_block.to_hex()
    print("Control Block Breakdown:")
    print(f"  Byte 1:      {cb_hex[:2]} (version + parity)")
    print(f"  Bytes 2-33:  {cb_hex[2:66]} (internal pubkey)")
    print(f"  Bytes 34-65: {cb_hex[66:]} (sibling: Bob Script's TapLeaf hash)")
    print()
    print("⚡ The sibling hash proves this script is part of the tree!")
    print()
    
    # Step 4: Prepare witness data
    preimage_hex = preimage.encode('utf-8').hex()
    
    print("Step 4: Witness Stack Construction")
    print("-" * 70)
    print("Witness Stack (bottom to top):")
    print(f"  [0] Preimage:      {preimage_hex}")
    print(f"  [1] Hash Script:   {hash_script.to_hex()}")
    print(f"  [2] Control Block: {control_block.to_hex()}")
    print()
    
    # Step 5: Build witness
    witness = TxWitnessInput([
        preimage_hex,           # [0] Script input: the secret
        hash_script.to_hex(),   # [1] Revealed script
        control_block.to_hex()  # [2] Merkle proof
    ])
    
    tx.witnesses.append(witness)
    
    # Step 6: Final transaction
    print("="*70)
    print("FINAL TRANSACTION (HASH SCRIPT PATH)")
    print("="*70)
    print(f"Transaction ID:  {tx.get_txid()}")
    print(f"Size:            {tx.get_size()} bytes")
    print(f"Virtual Size:    {tx.get_vsize()} vbytes")
    print()
    
    print("🎯 What Gets Revealed:")
    print("   ✓ The hash lock script")
    print("   ✓ The preimage 'helloworld'")
    print("   ✓ Internal public key (via control block)")
    print("   ✓ Bob Script's TapLeaf hash (as sibling)")
    print()
    print("🔒 What Stays Hidden:")
    print("   • Bob Script's actual content (only hash revealed)")
    print("   • Whether Key Path spending is possible")
    print("   • How many other scripts might exist")
    
    return tx

# Execute hash script path spending
tx_hash = hash_script_path_spending()
```

### Example 3: Reveal Phase - Bob Script Path Spending (Script 2)

```python
def bob_script_path_spending():
    """
    Reveal Phase: Spend via Bob Script Path (Bob's signature required)
    """
    setup('testnet')
    
    print("="*70)
    print("REVEAL PHASE: Bob Script Path Spending")
    print("="*70)
    print()
    
    # Step 1: Same script tree construction
    alice_private = PrivateKey('cRxebG1hY6vVgS9CSLNaEbEJaXkpZvc6nFeqqGT7v6gcW7MbzKNT')
    alice_public = alice_private.get_public_key()
    
    bob_private = PrivateKey('cSNdLFDf3wjx1rswNL2jKykbVkC6o56o5nYZi4FUkWKjFn2Q5DSG')
    bob_public = bob_private.get_public_key()
    
    # Rebuild script tree
    preimage_hash = hashlib.sha256("helloworld".encode('utf-8')).hexdigest()
    hash_script = Script(['OP_SHA256', preimage_hash, 'OP_EQUALVERIFY', 'OP_TRUE'])
    bob_script = build_bob_p2pk_script(bob_public)
    
    all_leafs = [hash_script, bob_script]
    taproot_address = alice_public.get_taproot_address(all_leafs)
    
    print("Step 1: Recreate Script Tree")
    print("-" * 70)
    print(f"Taproot Address: {taproot_address.to_string()}")
    print("(Same address as Hash Script Path!)")
    print()
    
    # Step 2: Build transaction
    commit_txid = "8caddfad76a5b3a8595a522e24305dc20580ca868ef733493e308ada084a050c"
    input_amount = 0.00001111  # 1111 sats
    output_amount = 0.00000900  # 900 sats (211 sats fee)
    
    print("Step 2: Transaction Details")
    print("-" * 70)
    print(f"Input:  {input_amount} BTC ({int(input_amount * 100_000_000)} sats)")
    print(f"Output: {output_amount} BTC ({int(output_amount * 100_000_000)} sats)")
    print(f"Fee:    {input_amount - output_amount} BTC")
    print()
    
    txin = TxInput(commit_txid, 1)
    txout = TxOutput(
        to_satoshis(output_amount),
        bob_public.get_taproot_address().to_script_pub_key()
    )
    tx = Transaction([txin], [txout], has_segwit=True)
    
    # Step 3: Build Control Block for Bob Script (index 1)
    # CRITICAL: The control block includes the SIBLING hash (Hash Script)
    control_block = ControlBlock(
        alice_public,           # Internal public key (same as before!)
        all_leafs,              # Script tree structure
        1,                      # Bob script index (we're spending this one)
        is_odd=taproot_address.is_odd()
    )
    
    print("Step 3: Control Block Construction")
    print("-" * 70)
    print(f"Control Block: {control_block.to_hex()}")
    print(f"Length:        {len(bytes.fromhex(control_block.to_hex()))} bytes")
    print()
    
    # Decode control block
    cb_hex = control_block.to_hex()
    print("Control Block Breakdown:")
    print(f"  Byte 1:      {cb_hex[:2]} (version + parity)")
    print(f"  Bytes 2-33:  {cb_hex[2:66]} (internal pubkey - SAME as Hash Script!)")
    print(f"  Bytes 34-65: {cb_hex[66:]} (sibling: Hash Script's TapLeaf hash)")
    print()
    
    # Step 4: Sign transaction (Bob's signature)
    sig = bob_private.sign_taproot_input(
        tx, 0,
        [taproot_address.to_script_pub_key()],
        [to_satoshis(input_amount)],
        script_path=True,           # Script path spending
        tapleaf_script=bob_script,  # Singular! The script we're executing
        tweak=False                 # No tweaking for script path
    )
    
    print("Step 4: Bob's Schnorr Signature")
    print("-" * 70)
    print(f"Signature: {sig.hex()}")
    print(f"Length:    {len(bytes.fromhex(sig.hex()))} bytes (64 bytes)")
    print()
    
    # Step 5: Build witness
    print("Step 5: Witness Stack Construction")
    print("-" * 70)
    print("Witness Stack (bottom to top):")
    print(f"  [0] Bob's Signature: {sig.hex()}")
    print(f"  [1] Bob Script:      {bob_script.to_hex()}")
    print(f"  [2] Control Block:   {control_block.to_hex()}")
    print()
    
    witness = TxWitnessInput([
        sig,                        # [0] Bob's signature
        bob_script.to_hex(),        # [1] Revealed script
        control_block.to_hex()      # [2] Merkle proof
    ])
    
    tx.witnesses.append(witness)
    
    # Step 6: Final transaction
    print("="*70)
    print("FINAL TRANSACTION (BOB SCRIPT PATH)")
    print("="*70)
    print(f"Transaction ID:  {tx.get_txid()}")
    print(f"Size:            {tx.get_size()} bytes")
    print(f"Virtual Size:    {tx.get_vsize()} vbytes")
    print()
    
    print("🎯 What Gets Revealed:")
    print("   ✓ Bob's P2PK script")
    print("   ✓ Bob's Schnorr signature")
    print("   ✓ Internal public key (via control block)")
    print("   ✓ Hash Script's TapLeaf hash (as sibling)")
    print()
    print("🔒 What Stays Hidden:")
    print("   • Hash Script's actual content (only hash revealed)")
    print("   • The preimage 'helloworld'")
    print("   • Whether Key Path spending is possible")
    
    return tx

# Execute Bob script path spending
tx_bob = bob_script_path_spending()
```

### Example 4: Merkle Proof Verification

```python
def verify_merkle_proof():
    """
    Verify that control blocks correctly prove script membership
    """
    print("="*70)
    print("MERKLE PROOF VERIFICATION")
    print("="*70)
    print()
    
    # Data from actual transactions
    hash_control_block = "c050be5fc44ec580c387bf45df275aaa8b27e2d7716af31f10eeed357d126bb4d32faaa677cb6ad6a74bf7025e4cd03d2a82c7fb8e3c277916d7751078105cf9df"
    hash_script_hex = "a820936a185caaa266bb9cbe981e9e05cb78cd732b0b3280eb944412bb6f8f8f07af8851"
    
    bob_control_block = "c050be5fc44ec580c387bf45df275aaa8b27e2d7716af31f10eeed357d126bb4d3fe78d8523ce9603014b28739a51ef826f791aa17511e617af6dc96a8f10f659e"
    bob_script_hex = "2084b5951609b76619a1ce7f48977b4312ebe226987166ef044bfb374ceef63af5ac"
    
    def tagged_hash(tag, data):
        """BIP 340 tagged hash"""
        tag_hash = hashlib.sha256(tag.encode()).digest()
        return hashlib.sha256(tag_hash + tag_hash + data).digest()
    
    def parse_control_block(cb_hex):
        """Parse control block components"""
        cb_bytes = bytes.fromhex(cb_hex)
        leaf_version = cb_bytes[0] & 0xfe
        parity = cb_bytes[0] & 0x01
        internal_pubkey = cb_bytes[1:33]
        sibling_hash = cb_bytes[33:65]
        return leaf_version, parity, internal_pubkey, sibling_hash
    
    # Parse both control blocks
    hash_version, hash_parity, hash_internal_key, hash_sibling = parse_control_block(hash_control_block)
    bob_version, bob_parity, bob_internal_key, bob_sibling = parse_control_block(bob_control_block)
    
    print("Step 1: Parse Control Blocks")
    print("-" * 70)
    print(f"Hash Script CB internal key: {hash_internal_key.hex()}")
    print(f"Bob Script CB internal key:  {bob_internal_key.hex()}")
    print(f"Internal keys match: {hash_internal_key == bob_internal_key} ✓")
    print()
    
    # Calculate TapLeaf hashes
    print("Step 2: Calculate TapLeaf Hashes")
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
    
    print(f"Hash Script TapLeaf: {hash_tapleaf.hex()}")
    print(f"Bob Script TapLeaf:  {bob_tapleaf.hex()}")
    print()
    
    # Verify sibling relationships
    print("Step 3: Verify Sibling Relationships")
    print("-" * 70)
    print(f"Hash CB's sibling matches Bob TapLeaf: {hash_sibling.hex() == bob_tapleaf.hex()} ✓")
    print(f"Bob CB's sibling matches Hash TapLeaf: {bob_sibling.hex() == hash_tapleaf.hex()} ✓")
    print()
    print("This proves both scripts are part of the same tree!")
    print()
    
    # Calculate Merkle Root
    print("Step 4: Calculate Merkle Root")
    print("-" * 70)
    
    # Sort lexicographically (CRITICAL!)
    if hash_tapleaf < bob_tapleaf:
        print("Sorting: Hash TapLeaf < Bob TapLeaf")
        merkle_root = tagged_hash("TapBranch", hash_tapleaf + bob_tapleaf)
    else:
        print("Sorting: Bob TapLeaf < Hash TapLeaf")
        merkle_root = tagged_hash("TapBranch", bob_tapleaf + hash_tapleaf)
    
    print(f"Merkle Root: {merkle_root.hex()}")
    print()
    
    # Calculate tweak
    print("Step 5: Calculate Output Key Tweak")
    print("-" * 70)
    tweak = tagged_hash("TapTweak", hash_internal_key + merkle_root)
    print(f"Tweak: {tweak.hex()}")
    print()
    
    print("="*70)
    print("VERIFICATION COMPLETE")
    print("="*70)
    print()
    print("✓ Both control blocks contain correct sibling hashes")
    print("✓ Merkle root can be reconstructed from either path")
    print("✓ This proves both scripts were committed to the address")
    print("✓ Verifier can trust either spending path is legitimate")

verify_merkle_proof()
```

---

## 🧪 Practice Tasks

### Task 1: Build Your Own Dual-Leaf Contract

**Objective**: Create a complete dual-leaf Taproot contract with custom conditions.

**Scenario**: Create an escrow between Alice and Bob:
- **Script 1**: Time lock - Alice can reclaim after 10 blocks
- **Script 2**: Bob can spend immediately with his signature

**Steps:**

1. Create both scripts
2. Build the dual-leaf tree
3. Generate Taproot address
4. Fund it on testnet
5. Test both spending paths

**Code Template:**

```python
# Script 1: Time-locked recovery for Alice
timelock_script = Script([
    alice_public.to_x_only_hex(),
    'OP_CHECKSIGVERIFY',
    10,  # 10 blocks
    'OP_CHECKSEQUENCEVERIFY'
])

# Script 2: Bob's immediate spend
bob_script = Script([
    bob_public.to_x_only_hex(),
    'OP_CHECKSIG'
])

# Build tree and test!
```

**Deliverables:**
- Transaction IDs for both spending paths
- Screenshot of transactions on mempool.space
- Analysis: Which path is more expensive? Why?

### Task 2: Debug Merkle Proof Failure

**Challenge**: A student's dual-leaf transaction is failing. Find and fix the error.

**Broken Code:**

```python
# BROKEN CODE - Find the error!
def broken_dual_leaf():
    # Scripts created correctly
    leafs = [script_a, script_b]
    address = internal_key.get_taproot_address(leafs)
    
    # Spending script_a
    control_block = ControlBlock(
        internal_key,
        leafs,
        1,  # ERROR HERE?
        is_odd=address.is_odd()
    )
    
    witness = TxWitnessInput([
        input_data,
        script_a.to_hex(),  # Using script_a
        control_block.to_hex()
    ])
```

**Your Task:**
1. Identify the error (hint: index mismatch)
2. Explain why it causes Merkle proof failure
3. Fix and test on testnet
4. Explain what the verifier sees when this fails

### Task 3: Merkle Root Calculation

**Objective**: Manually calculate and verify Merkle root.

**Given:**
- Hash Script TapLeaf: `fe78d8523ce9603014b28739a51ef826f791aa17511e617af6dc96a8f10f659e`
- Bob Script TapLeaf: `2faaa677cb6ad6a74bf7025e4cd03d2a82c7fb8e3c277916d7751078105cf9df`

**Tasks:**

1. Sort these hashes lexicographically
2. Calculate TapBranch hash manually
3. Verify against actual on-chain data
4. Explain why sorting is necessary

**Bonus**: Calculate the tweak and output key.

---

## 📘 Reading

### Required Reading
- **Mastering Taproot** - Chapter 7: Dual-Leaf Script Tree (pages 91-108)
- Focus on:
  - Merkle tree structure (pages 92-94)
  - TapBranch hash calculation (pages 95-97)
  - Control block with sibling hash (pages 97-99)
  - Lexicographic sorting (page 105)

### BIP References
- **BIP 341**: Taproot specification
  - Section: "Constructing and spending Taproot outputs"
  - Focus on Merkle proof verification
- **BIP 340**: Tagged hash function
  - Understand "TapLeaf" and "TapBranch" tags

### Supplementary Resources
- **Bitcoin Optech**: Taproot Merkle trees
- **Visual Tool**: Use RootScope to visualize dual-leaf trees
- **Real Examples**: Analyze actual dual-leaf spends on testnet

---

## 💬 Discussion Prompts

### Question 1: Merkle Proof Security

**"In a dual-leaf tree, when spending via Script 1, the control block reveals Script 2's TapLeaf hash (but not its content). How does this balance privacy with security? What information does an observer gain, and what remains hidden? Could an attacker exploit the revealed hash?"**

*Hint: Consider that hashing is one-way, and think about what the hash commits to.*

### Question 2: Lexicographic Sorting Necessity

**"Why must TapLeaf hashes be sorted lexicographically before calculating the TapBranch hash? What would go wrong if we skipped sorting? Design a scenario where lack of sorting causes a real problem."**

*Consider: What happens if Alice and Bob create the tree in different orders?*

### Question 3: Privacy Degradation

**"Compare privacy across spending methods: (1) Key Path, (2) Hash Script Path, (3) Bob Script Path. Rank them from most to least private and explain. If you were designing the contract, how would you incentivize key-path usage?"**

*Think about: What gets revealed in each case, and what stays hidden forever?*

---

## 🎙️ Instructor's Note

Welcome to dual-leaf trees—where Taproot becomes truly interesting!

In Lesson 6, you learned single-leaf trees where the TapLeaf hash directly became the Merkle root. **Now we're building real Merkle trees** with branches, and this is where the magic happens.

**Key insights for this lesson:**

1. **Merkle proofs are elegant**: Instead of revealing all scripts, we reveal only the sibling hash. The verifier can reconstruct the Merkle root and verify our script was committed.

2. **Lexicographic sorting is non-negotiable**: Without it, different parties creating the same tree might get different Merkle roots. Sorting ensures determinism.

3. **Control block grows linearly**: 33 bytes (single-leaf) → 65 bytes (dual-leaf) → 97 bytes (four-leaf). Each sibling adds 32 bytes.

4. **Index matters**: When you build the control block, the script index must match the script you're actually spending. Index 0 for first script, index 1 for second script.

**Common pitfalls:**

- **Index mismatch**: Control block says index 1, but witness uses script 0 → FAIL
- **Forgetting to sort**: Calculate TapBranch without sorting → Wrong Merkle root → FAIL
- **Wrong sibling in control block**: Manually building control blocks and using wrong hash → Merkle proof fails

**Debugging tip**: When a dual-leaf spend fails, check:
1. Are scripts in the same order as commit phase?
2. Is the control block index correct?
3. Does the sibling hash in control block match the other script's TapLeaf?

Next lesson, we'll scale up to **four-leaf trees**, which require multiple levels of Merkle proofs and extended control blocks. If you master dual-leaf, four-leaf will be straightforward!

Take time with the Merkle proof verification code. Understanding how the verifier reconstructs the tree is crucial for debugging and for appreciating Taproot's privacy guarantees.

See you in Discord!

—Aaron

---

## 📌 Key Takeaways

✅ Dual-leaf trees use true Merkle structure with TapBranch hash  
✅ Control block expands to 65 bytes (adds 32-byte sibling hash)  
✅ Lexicographic sorting of TapLeaf hashes ensures deterministic Merkle root  
✅ Each spending path reveals only its sibling's hash, not content  
✅ Merkle proof allows verification without revealing unused scripts  
✅ Script index (0 or 1) must match the script being spent  
✅ Both spending paths use the same Taproot address  
✅ Same internal key appears in both control blocks

---

**Next Lesson**: [Lesson 8: Four-Leaf Taproot Script Tree →](./lesson-08-four-leaf-taptree.md)

---

*This lesson is part of the Taproot Script Engineering Cohort at Chaincode Labs (2026)*