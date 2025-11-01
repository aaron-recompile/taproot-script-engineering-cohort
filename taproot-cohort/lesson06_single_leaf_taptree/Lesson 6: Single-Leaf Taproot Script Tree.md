# Lesson 6: Single-Leaf Taproot Script Tree

**Week**: 6  
**Duration**: Self-paced (estimated 4-5 hours)  
**Prerequisites**: Lesson 5 (Taproot Introduction and Key Tweaking)

---

## 🎯 Learning Objectives

By the end of this lesson, you will be able to:

1. Understand and implement the Commit-Reveal pattern for Taproot contracts
2. Build single-leaf script trees with hash lock scripts
3. Construct and decode control blocks (33 bytes for single-leaf)
4. Implement dual-path spending: key-path vs script-path
5. Explain witness data ordering for script-path spending
6. Debug script-path transaction failures using stack traces
7. Compare privacy and efficiency between spending paths

---

## 🧩 Key Concepts

### Why Script Path Changes Everything

In Lesson 5, we learned about **key-path-only** Taproot—essentially Schnorr signatures with key tweaking. But Taproot's true power comes from **script paths**: the ability to hide complex spending conditions inside the address while maintaining the appearance of a simple payment.

**The Problem with Traditional Scripts:**

Traditional Bitcoin scripts (P2PKH, P2SH) either:
- **Completely expose** all conditions (compromising privacy)
- Require **complex multi-signature** setups (inefficient)
- Are **easily identifiable** by transaction patterns

**Taproot's Script Path Solution:**

> Complex conditional contracts look **identical to ordinary payments** when unspent, revealing only the necessary execution path when needed.

### Business Scenario: Alice's Conditional Payment Contract

Let's start with a concrete example to understand script paths:

**Alice wants to create a conditional payment with these characteristics:**

1. **Conditional Path**: Anyone who knows the secret word "helloworld" can claim the funds
2. **Alternative Path**: Alice can reclaim her funds at any time using her private key
3. **Privacy Requirement**: When unspent, external observers cannot distinguish this from simple payments

**This dual-path design is extremely common in real applications:**

| Application Scenario | Description |
|---------------------|-------------|
| **Digital Goods Sales** | Buyer gets unlock keys after payment; seller retains refund authority |
| **Bounty Tasks** | Puzzle solvers claim reward; unused bounty can be reclaimed by publisher |
| **Conditional Escrow** | Automatically releases under specific conditions; owner can reclaim otherwise |
| **Educational Incentives** | Students claim reward upon correct answers; teachers retain management control |

### Taproot Dual-Path Architecture

In our scenario, Alice's Taproot address contains **two spending paths**:

**1. Key Path (Collaborative Path):**
- Alice signs directly with her tweaked private key
- 64-byte Schnorr signature, maximum efficiency
- Complete privacy, no script information leakage

**2. Script Path (Conditional Path):**
- Uses Hash Lock script: `OP_SHA256 <hash> OP_EQUALVERIFY OP_TRUE`
- Anyone who knows the preimage "helloworld" can spend
- Requires revealing script content, but doesn't expose Key Path existence

### The Commit-Reveal Development Pattern

Before diving into code, understand Taproot's core pattern: **Commit-Reveal**. This becomes the fundamental development model for all Taproot applications.

#### Commit-Reveal Pattern Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     COMMIT PHASE                            │
├─────────────────────────────────────────────────────────────┤
│  • Build script tree containing multiple spending conditions│
│  • Commit script tree to a Taproot address                  │
│  • Generate "intermediate address" to lock funds            │
│  • External parties CANNOT know specific conditions         │
└─────────────────────────────────────────────────────────────┘
                            ⬇
┌─────────────────────────────────────────────────────────────┐
│                     REVEAL PHASE                            │
├─────────────────────────────────────────────────────────────┤
│  • Choose ONE spending condition for unlocking              │
│  • Key Path: Spend directly with key (maximum privacy)     │
│  • Script Path: Reveal and execute specific script branch  │
│  • Only expose actually used conditions                     │
│  • Other conditions remain PRIVATE FOREVER                  │
└─────────────────────────────────────────────────────────────┘
```

**The Power of This Pattern:**

- **During Commit**: All contracts of different complexity look identical
- **During Reveal**: Only the actually used branch needs to be exposed

### Single-Leaf Hash Lock Script

A **single-leaf script tree** is the simplest Taproot structure: one script branch committed to the tree.

**Hash Lock Script Structure:**

```python
Script([
    'OP_SHA256',       # Calculate SHA256 of input
    preimage_hash,     # Expected hash to match against
    'OP_EQUALVERIFY',  # Verify hash equality or fail
    'OP_TRUE'          # Success condition
])
```

**How it works:**

1. Script expects one input on the stack: the preimage
2. `OP_SHA256` hashes it
3. Compare with expected hash using `OP_EQUALVERIFY`
4. If match: `OP_TRUE` (script succeeds)
5. If no match: script fails

**Serialized form:**

```
a8 20 936a185caaa266bb9cbe981e9e05cb78cd732b0b3280eb944412bb6f8f8f07af 88 51

a8 = OP_SHA256
20 = PUSH 32 bytes
936a185c... = SHA256("helloworld")
88 = OP_EQUALVERIFY
51 = OP_TRUE
```

### Control Block Structure (Single-Leaf)

The **control block** is cryptographic proof that a script is part of the committed tree.

**For single-leaf trees, the control block is 33 bytes:**

```
Control Block Structure (33 bytes):
┌─────────┬──────────────────────────────────┐
│ Byte 1  │         Bytes 2-33               │
├─────────┼──────────────────────────────────┤
│   c1    │ 50be5fc4...126bb4d3             │
├─────────┼──────────────────────────────────┤
│Ver+Parity│    Internal Public Key          │
└─────────┴──────────────────────────────────┘

Byte 1 breakdown:
  c1 = c0 (leaf version) + 01 (parity flag)
  
  c0 = 0xc0 (binary: 1100 0000) - Tapscript version 0
  01 = y-coordinate is odd
```

**What the control block contains:**

1. **Leaf Version + Parity** (1 byte):
   - Upper bits: Tapscript version (0xc0)
   - Lowest bit: y-coordinate parity of output key

2. **Internal Public Key** (32 bytes):
   - The original public key before tweak adjustment
   - Used to recalculate output key during verification

**For multi-leaf trees**: Control block would include sibling hashes (32 bytes each) to form Merkle proof. We'll cover this in Lesson 7.

### Tagged Hash (BIP 340 Concept)

Before implementing, understand **Tagged Hash**—a hashing method that prevents hash collisions between different protocols.

**Standard Tagged Hash Function:**

```python
def tagged_hash(tag, data):
    tag_hash = hashlib.sha256(tag.encode()).digest()
    return hashlib.sha256(tag_hash + tag_hash + data).digest()
```

**Common tags in Taproot:**

- `"TapLeaf"` - for calculating script leaf hash
- `"TapBranch"` - for calculating branch hash in Merkle tree
- `"TapTweak"` - for calculating tweak value (pubkey + merkle_root)

**Why double the tag hash?**

This design prevents hash collisions between different protocols. Each tag has its specific purpose, ensuring hash values from different contexts never accidentally duplicate.

### Witness Data Ordering (Critical!)

For **script-path spending**, the witness data order is **strictly defined**:

```
Witness Stack (bottom to top):
┌─────────────────────────────────────────────┐
│ [n-1] Control Block (last element)         │
│ [n-2] Script Code (second to last)         │
│ [...] Script Execution Inputs               │
│ [1]   Additional inputs (if needed)        │
│ [0]   First input for script               │
└─────────────────────────────────────────────┘

For our hash lock example:
┌─────────────────────────────────────────────┐
│ [2] Control Block: c150be5fc4...126bb4d3   │
│ [1] Script: a820936a185c...8851            │
│ [0] Preimage: 68656c6c6f776f726c64         │
└─────────────────────────────────────────────┘
```

**Why this order matters:**

Bitcoin Core parses witness data in fixed order to identify:
1. Control block (proves script legitimacy)
2. Script code (what to execute)
3. Stack inputs (data for script)

**Common error:** Reversing the order causes transaction validation to fail!

---

## 💻 Example Code

### Example 1: Phase 1 - Commit (Building the Taproot Address)

```python
from bitcoinutils.setup import setup
from bitcoinutils.keys import PrivateKey
from bitcoinutils.script import Script
import hashlib

def build_hash_lock_script(preimage):
    """
    Build a Hash Lock Script – anyone who knows the preimage can spend
    """
    # Hash the preimage
    preimage_hash = hashlib.sha256(preimage.encode('utf-8')).hexdigest()
    
    return Script([
        'OP_SHA256',        # Calculate SHA256 of input
        preimage_hash,      # Expected hash to match
        'OP_EQUALVERIFY',   # Verify equality or fail
        'OP_TRUE'           # Success condition
    ])

def create_taproot_commitment():
    """
    Phase 1: COMMIT - Build Taproot address with script commitment
    """
    setup('testnet')
    
    print("="*70)
    print("PHASE 1: COMMIT - Building Taproot Address")
    print("="*70)
    print()
    
    # Step 1: Alice's internal key - foundation for dual-path control
    internal_private = PrivateKey('cRxebG1hY6vVgS9CSLNaEbEJaXkpZvc6nFeqqGT7v6gcW7MbzKNT')
    internal_public = internal_private.get_public_key()
    
    print("Step 1: Internal Key Setup")
    print("-" * 70)
    print(f"Internal Private Key: {internal_private.to_wif()}")
    print(f"Internal Public Key:  {internal_public.to_hex()}")
    print(f"X-only Pubkey:        {internal_public.to_hex()[2:]}")
    print()
    
    # Step 2: Build Hash Lock script for "helloworld" secret
    preimage = "helloworld"
    hash_lock_script = build_hash_lock_script(preimage)
    
    print("Step 2: Hash Lock Script")
    print("-" * 70)
    print(f"Preimage (secret):    {preimage}")
    print(f"SHA256 Hash:          {hashlib.sha256(preimage.encode()).hexdigest()}")
    print(f"Script:               {hash_lock_script}")
    print(f"Serialized Script:    {hash_lock_script.to_hex()}")
    print()
    
    # Decode the serialized script
    script_hex = hash_lock_script.to_hex()
    print("Script Breakdown:")
    print(f"  a8     = OP_SHA256")
    print(f"  20     = PUSH 32 bytes")
    print(f"  {script_hex[4:68]} = SHA256('helloworld')")
    print(f"  88     = OP_EQUALVERIFY")
    print(f"  51     = OP_TRUE")
    print()
    
    # Step 3: Generate Taproot address (commit script tree to blockchain)
    # This creates our "intermediate address" where funds will be locked
    # The double brackets [[script]] indicate a single-leaf tree
    taproot_address = internal_public.get_taproot_address([[hash_lock_script]])
    
    print("Step 3: Taproot Address Generation")
    print("-" * 70)
    print(f"Taproot Address:      {taproot_address.to_string()}")
    print(f"Address Type:         Bech32m (tb1p...)")
    print()
    
    # Step 4: Understanding what was committed
    print("="*70)
    print("COMMITMENT ANALYSIS")
    print("="*70)
    print()
    print("What is hidden in this address:")
    print("  ✓ Internal public key (only control block reveals it)")
    print("  ✓ Hash lock script existence")
    print("  ✓ The secret hash value")
    print("  ✓ Number of script paths")
    print()
    print("What observers see:")
    print("  • Just a normal Taproot address (tb1p...)")
    print("  • Identical to any other Taproot address")
    print("  • Cannot distinguish from simple payment")
    print()
    
    print("🎯 This is the 'intermediate address' or 'custody address'")
    print("   Send funds here to lock them under the conditions.")
    print()
    
    return taproot_address, hash_lock_script, internal_private

# Execute the commit phase
address, script, private_key = create_taproot_commitment()
```

### Example 2: Phase 2 - Key Path Spending (Alice's Direct Control)

```python
from bitcoinutils.transactions import Transaction, TxInput, TxOutput, TxWitnessInput
from bitcoinutils.utils import to_satoshis

def alice_key_path_spending():
    """
    Phase 2: REVEAL - Key Path Spending
    Alice reclaims funds using her internal key (cooperative path)
    """
    setup('testnet')
    
    print("="*70)
    print("PHASE 2: REVEAL - Key Path Spending")
    print("="*70)
    print()
    
    # Step 1: Recreate the same setup from commit phase
    alice_private = PrivateKey('cRxebG1hY6vVgS9CSLNaEbEJaXkpZvc6nFeqqGT7v6gcW7MbzKNT')
    alice_public = alice_private.get_public_key()
    
    # Step 2: Rebuild same script and Taproot address
    preimage = "helloworld"
    tr_script = build_hash_lock_script(preimage)
    taproot_address = alice_public.get_taproot_address([[tr_script]])
    
    print("Step 1: Recreate Taproot Setup")
    print("-" * 70)
    print(f"Taproot Address: {taproot_address.to_string()}")
    print("(Must match commit phase exactly!)")
    print()
    
    # Step 3: Define UTXO to spend
    # This would come from the actual funding transaction
    commit_txid = "4fd83128fb2df7cd25d96fdb6ed9bea26de755f212e37c3aa017641d3d2d2c6d"
    input_amount = 0.00003900  # 3900 satoshis
    output_amount = 0.00003700  # 3700 satoshis (200 sats fee)
    
    print("Step 2: Input Details")
    print("-" * 70)
    print(f"Previous TXID: {commit_txid}")
    print(f"Input Amount:  {input_amount} BTC ({int(input_amount * 100_000_000)} sats)")
    print(f"Output Amount: {output_amount} BTC ({int(output_amount * 100_000_000)} sats)")
    print(f"Fee:           {input_amount - output_amount} BTC ({int((input_amount - output_amount) * 100_000_000)} sats)")
    print()
    
    # Step 4: Build transaction
    txin = TxInput(commit_txid, 0)
    txout = TxOutput(
        to_satoshis(output_amount),
        alice_public.get_taproot_address().to_script_pub_key()
    )
    tx = Transaction([txin], [txout], has_segwit=True)
    
    # Step 5: Sign with Schnorr (key path)
    # KEY POINT: Even for key path, we need script info to calculate tweak!
    sig = alice_private.sign_taproot_input(
        tx,
        0,  # Input index
        [taproot_address.to_script_pub_key()],  # Input ScriptPubKey
        [to_satoshis(input_amount)],             # Input amount
        script_path=False,                       # Explicitly specify Key Path
        tapleaf_scripts=[tr_script]              # Still need script to calculate tweak
    )
    
    # Step 6: Build witness (only signature!)
    tx.witnesses.append(TxWitnessInput([sig]))
    
    print("Step 3: Schnorr Signature (Key Path)")
    print("-" * 70)
    print(f"Signature:     {sig.hex()}")
    print(f"Length:        {len(bytes.fromhex(sig.hex()))} bytes (fixed)")
    print()
    
    print("Step 4: Witness Structure")
    print("-" * 70)
    print("Witness Stack:")
    print("  [0] Schnorr Signature (64 bytes)")
    print()
    print("⚡ Note: NO public key, NO script, NO control block!")
    print("   This is maximum privacy and efficiency.")
    print()
    
    # Step 7: Transaction details
    print("="*70)
    print("FINAL TRANSACTION (KEY PATH)")
    print("="*70)
    print(f"Transaction ID:  {tx.get_txid()}")
    print(f"Raw Transaction: {tx.serialize()}")
    print(f"Size:            {tx.get_size()} bytes")
    print(f"Virtual Size:    {tx.get_vsize()} vbytes")
    print()
    
    print("🎯 Key Path Characteristics:")
    print("   • Witness: Only 64-byte signature")
    print("   • Privacy: Complete (no script info revealed)")
    print("   • Efficiency: Highest (single verification)")
    print("   • Appearance: Identical to any simple Taproot payment")
    print()
    
    print("Important Technical Detail:")
    print("   Even for key path, signing needs tapleaf_scripts parameter")
    print("   to calculate the correct tweak value!")
    print("   This is because output key = internal key + tweak(script_tree)")
    
    return tx

# Execute key path spending
tx = alice_key_path_spending()
```

### Example 3: Phase 3 - Script Path Spending (Conditional Unlock)

```python
from bitcoinutils.script import ControlBlock

def script_path_spending():
    """
    Phase 3: REVEAL - Script Path Spending
    Anyone with the preimage "helloworld" can spend (no private key needed!)
    """
    setup('testnet')
    
    print("="*70)
    print("PHASE 3: REVEAL - Script Path Spending")
    print("="*70)
    print()
    
    # Step 1: Rebuild previous Taproot setup (must match commitment exactly!)
    alice_private = PrivateKey('cRxebG1hY6vVgS9CSLNaEbEJaXkpZvc6nFeqqGT7v6gcW7MbzKNT')
    alice_public = alice_private.get_public_key()
    
    # Step 2: Recreate same Hash Lock script
    preimage = "helloworld"
    tr_script = build_hash_lock_script(preimage)
    taproot_address = alice_public.get_taproot_address([[tr_script]])
    
    print("Step 1: Recreate Setup")
    print("-" * 70)
    print(f"Taproot Address: {taproot_address.to_string()}")
    print(f"Script:          {tr_script.to_hex()}")
    print()
    
    # Step 3: Build spending transaction structure
    commit_txid = "68f7c8f0ab6b3c6f7eb037e36051ea3893b668c26ea6e52094ba01a7722e604f"
    input_amount = 0.00005000  # 5000 satoshis
    output_amount = 0.00004000  # 4000 satoshis (1000 sats fee)
    
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
    
    # Step 4: CRITICAL - Build control block to prove script legitimacy
    control_block = ControlBlock(
        alice_public,           # Internal public key for verification
        [[tr_script]],          # Script tree structure (single leaf)
        0,                      # Script index in tree (0 for single leaf)
        is_odd=taproot_address.is_odd()  # Output key parity
    )
    
    print("Step 3: Control Block Construction")
    print("-" * 70)
    print(f"Control Block:   {control_block.to_hex()}")
    print(f"Length:          {len(bytes.fromhex(control_block.to_hex()))} bytes")
    print()
    
    # Decode control block
    cb_hex = control_block.to_hex()
    print("Control Block Breakdown:")
    print(f"  {cb_hex[:2]}     = Leaf version (c0) + Parity (1)")
    print(f"  {cb_hex[2:]}    = Internal public key (32 bytes)")
    print()
    
    # Step 5: Prepare script execution input - the secret "helloworld"
    preimage_hex = preimage.encode('utf-8').hex()
    
    print("Step 4: Preimage Preparation")
    print("-" * 70)
    print(f"Preimage (text): {preimage}")
    print(f"Preimage (hex):  {preimage_hex}")
    print(f"Decoded:         {bytes.fromhex(preimage_hex).decode('utf-8')}")
    print()
    
    # Step 6: Build Script Path witness (ORDER MATTERS!)
    script_path_witness = TxWitnessInput([
        preimage_hex,           # [0] Script execution input: the secret
        tr_script.to_hex(),     # [1] Revealed script content
        control_block.to_hex()  # [2] Control block: cryptographic proof
    ])
    
    tx.witnesses.append(script_path_witness)
    
    print("Step 5: Witness Stack Construction")
    print("-" * 70)
    print("Witness Stack (bottom to top):")
    print(f"  [0] Preimage:      {preimage_hex}")
    print(f"  [1] Script:        {tr_script.to_hex()}")
    print(f"  [2] Control Block: {control_block.to_hex()}")
    print()
    print("⚠️  ORDER IS CRITICAL!")
    print("    Bitcoin Core expects: [inputs...] [script] [control_block]")
    print()
    
    # Step 7: Final transaction
    print("="*70)
    print("FINAL TRANSACTION (SCRIPT PATH)")
    print("="*70)
    print(f"Transaction ID:  {tx.get_txid()}")
    print(f"Size:            {tx.get_size()} bytes")
    print(f"Virtual Size:    {tx.get_vsize()} vbytes")
    print()
    
    print("🎯 Script Path Characteristics:")
    print("   • Witness: Preimage + Script + Control Block")
    print("   • Privacy: Partial (reveals one script, not others)")
    print("   • Efficiency: Good (but larger than key path)")
    print("   • Anyone with preimage can spend (no key needed!)")
    print()
    
    print("What Gets Revealed:")
    print("   ✓ The hash lock script")
    print("   ✓ The preimage 'helloworld'")
    print("   ✓ Internal public key (via control block)")
    print()
    print("What Stays Hidden:")
    print("   • Whether other script paths exist")
    print("   • What those other paths might be")
    print("   • Key path spending capability")
    
    return tx

# Execute script path spending
tx_script = script_path_spending()
```

### Example 4: Stack Execution Trace (Debugging Script Path)

```python
def trace_script_path_execution():
    """
    Visualize the complete stack execution for script path spending
    This helps debug script failures
    """
    print("="*70)
    print("SCRIPT PATH STACK EXECUTION TRACE")
    print("="*70)
    print()
    
    preimage = "helloworld"
    preimage_hash = hashlib.sha256(preimage.encode()).hexdigest()
    
    print("Initial State: Empty Stack")
    print("-" * 70)
    print("│ (empty)                                     │")
    print("└─────────────────────────────────────────────┘")
    print()
    
    print("Step 1: Load Preimage from Witness")
    print("-" * 70)
    print(f"│ {preimage.encode().hex()} (preimage)          │")
    print("└─────────────────────────────────────────────┘")
    print()
    
    print("Step 2: Execute OP_SHA256")
    print("-" * 70)
    print(f"│ {preimage_hash} (computed_hash) │")
    print("└─────────────────────────────────────────────┘")
    print()
    
    print("Step 3: Script Pushes Expected Hash")
    print("-" * 70)
    print(f"│ {preimage_hash} (expected_hash)  │")
    print(f"│ {preimage_hash} (computed_hash)  │")
    print("└─────────────────────────────────────────────┘")
    print()
    
    print("Step 4: Execute OP_EQUALVERIFY")
    print("-" * 70)
    if preimage_hash == preimage_hash:
        print("│ (empty - hashes matched! ✓)                 │")
    else:
        print("│ FAIL - hashes don't match ✗                 │")
    print("└─────────────────────────────────────────────┘")
    print()
    
    print("Step 5: Execute OP_TRUE")
    print("-" * 70)
    print("│ 01 (TRUE)                                    │")
    print("└─────────────────────────────────────────────┘")
    print()
    
    print("Execution Result: SUCCESS ✓")
    print("The script-path spending is authorized!")
    print()
    
    print("Common Failure Points:")
    print("-" * 70)
    print("1. Wrong preimage → Hash mismatch at OP_EQUALVERIFY")
    print("2. Wrong witness order → Bitcoin Core can't parse structure")
    print("3. Wrong control block → Merkle proof fails")
    print("4. Wrong script → TapLeaf hash doesn't match commitment")

trace_script_path_execution()
```

---

## 🧪 Practice Tasks

### Task 1: Build Your Own Hash Lock Contract

**Objective**: Create a complete single-leaf Taproot hash lock contract on testnet.

**Steps:**

1. Generate a new internal private key
2. Choose your own secret word (not "helloworld"!)
3. Build the hash lock script
4. Generate the Taproot address (commit phase)
5. Fund it with testnet coins
6. Spend via script path using your secret
7. Verify the transaction on mempool.space

**Code Template:**

```python
# Your secret word
my_secret = "bitcoin2025"  # Choose your own!

# Follow the examples above to:
# 1. Build script
# 2. Create address
# 3. Fund it
# 4. Spend with script path
```

**Deliverables:**
- Screenshot of funded address
- Screenshot of successful spend
- Transaction IDs for both transactions
- Analysis: How much larger is script-path vs key-path?

### Task 2: Debug a Failing Script Path Transaction

**Challenge**: Identify and fix errors in script-path spending.

**Scenario**: A student submitted this code but the transaction fails:

```python
# BROKEN CODE - Find the errors!
def broken_script_path():
    setup('testnet')
    
    private = PrivateKey()
    public = private.get_public_key()
    
    script = Script(['OP_SHA256', 'wrong_hash', 'OP_EQUAL'])  # Error 1?
    address = public.get_taproot_address([[script]])
    
    # ... assume funded ...
    
    tx = Transaction([TxInput(txid, 0)], [TxOutput(amount, dest)])
    
    # Wrong witness order?
    control = ControlBlock(public, [[script]], 0, is_odd=False)  # Error 2?
    witness = TxWitnessInput([
        control.to_hex(),        # Error 3?
        script.to_hex(),
        "mypreimage".encode().hex()
    ])
    
    tx.witnesses.append(witness)
    return tx
```

**Your Task:**
1. Identify ALL errors (there are at least 3)
2. Fix each error with explanation
3. Create working version
4. Test on testnet

### Task 3: Compare Key Path vs Script Path

**Objective**: Quantify the privacy and efficiency trade-offs.

**Create two transactions from the same Taproot address:**
1. One using key-path spending
2. One using script-path spending

**Measure and compare:**
- Transaction sizes (bytes)
- Virtual sizes (vBytes)
- Witness sizes
- Fee costs (at 10 sat/vB)
- What information is revealed in each case

**Questions to answer:**
- How much more expensive is script-path?
- When would you choose script-path over key-path?
- What privacy is lost in script-path?

---

## 📘 Reading

### Required Reading
- **Mastering Taproot** - Chapter 6: Building Real Taproot Contracts (pages 72-90)
- Focus on:
  - Commit-Reveal pattern (pages 73-75)
  - Single-leaf hash lock implementation (pages 75-82)
  - Control block structure (pages 80-81)
  - Witness data ordering (pages 81-82)

### BIP References
- **BIP 341**: Taproot specification - Script validation rules
  - Section: Control block and Merkle proof
- **BIP 342**: Tapscript specification
  - Section: Tagged hash usage
- **BIP 340**: Schnorr signatures
  - Review: Tagged hash definition

### Supplementary Resources
- **Bitcoin Optech**: Taproot script validation
- **Debugging Tools**: Use StackFlow for script execution visualization
- **Real Examples**: Analyze actual Taproot script-path spends on mainnet

---

## 💬 Discussion Prompts

### Question 1: Commit-Reveal Security

**"In the Commit-Reveal pattern, the commitment phase locks funds to a script tree without revealing the scripts. Explain why this is secure. What prevents an attacker from modifying the script conditions after funds are committed? How does the control block prove script legitimacy during the reveal phase?"**

*Hint: Think about cryptographic binding via Merkle root and output key calculation.*

### Question 2: Privacy Analysis

**"Alice's hash lock contract has two paths: key path (private) and script path (reveals script). If Alice uses key path, does anyone learn that a hash lock script existed? If she uses script path, what remains hidden? Discuss the partial privacy of script path spending."**

*Consider: What does the blockchain permanently record in each case?*

### Question 3: Practical Use Cases

**"The lesson showed a simple hash lock (knowledge of secret). Design a more complex single-leaf contract for a real-world scenario. What script would you use? Why is a single leaf sufficient (or not) for your use case? When would you need multiple leaves?"**

*Examples to consider: Time locks, multi-sig, atomic swaps, etc.*

---

## 🎙️ Instructor's Note

Welcome to Script Path spending—where Taproot gets interesting!

In Lesson 5, we learned the cryptographic foundation: Schnorr signatures and key tweaking. Now we're adding the "script tree" part of Taproot. This is where conditions beyond "just sign with the key" come into play.

**Key insights for this lesson:**

1. **Commit-Reveal is fundamental**: You'll use this pattern for EVERY Taproot contract. Commit first (lock funds), reveal later (spend via chosen path).

2. **Control blocks are proof**: The control block cryptographically proves that a script was part of the original commitment. Without it, script-path spending fails.

3. **Witness ordering matters**: Get the order wrong (`[preimage, script, control]`) and your transaction is invalid. Bitcoin Core is strict about this.

4. **Single-leaf is the simplest case**: We're starting simple (one script) before moving to dual-leaf (Lesson 7) and four-leaf (Lesson 8) trees.

**Common pitfalls:**

- **Forgetting tapleaf_scripts in key-path signing**: Even key-path needs the script tree to calculate tweak!
- **Wrong control block parity**: The `is_odd` flag must match the output key's y-coordinate
- **Preimage encoding**: Must convert string → UTF-8 bytes → hex

Take time with the practice tasks. Build a real contract on testnet, break it, fix it. Understanding single-leaf trees is essential before we tackle multi-leaf structures.

Next lesson, we'll learn how two scripts can hide behind one address—and how Merkle proofs work.

See you in Discord!

—Aaron

---

## 📌 Key Takeaways

✅ Commit-Reveal pattern: Commit scripts to address, reveal only what's used  
✅ Single-leaf tree: Simplest Taproot structure (one script branch)  
✅ Hash lock script: `OP_SHA256 <hash> OP_EQUALVERIFY OP_TRUE`  
✅ Control block (33 bytes): Proves script legitimacy for single-leaf  
✅ Key path = maximum privacy; Script path = partial privacy  
✅ Witness ordering: `[inputs...] [script] [control_block]` (ORDER MATTERS!)  
✅ Tagged hash prevents cross-protocol collisions  
✅ Script path reveals one condition; others stay hidden forever

---

**Next Lesson**: [Lesson 7: Dual-Leaf Taproot Script Tree →](./lesson-07-dual-leaf-taptree.md)

---

*This lesson is part of the Taproot Script Engineering Cohort at Chaincode Labs (2026)*