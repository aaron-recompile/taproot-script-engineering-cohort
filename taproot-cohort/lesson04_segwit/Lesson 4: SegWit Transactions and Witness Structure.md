# Lesson 4: SegWit Transactions and Witness Structure

**Week**: 4  
**Duration**: Self-paced (estimated 5-6 hours)  
**Prerequisites**: Lessons 1-3 (Keys, P2PKH, P2SH)

---

## 🎯 Learning Objectives

By the end of this lesson, you will be able to:

1. Explain transaction malleability and how SegWit solves it
2. Understand the witness structure and witness segregation concept
3. Construct complete P2WPKH (native SegWit) transactions
4. Trace P2WPKH stack execution with witness data
5. Compare Legacy, P2SH-wrapped SegWit, and native SegWit
6. Calculate transaction weight units and fee optimization
7. Recognize SegWit as the foundation for Taproot

---

## 🧩 Key Concepts

### Transaction Malleability: The Problem SegWit Solves

**What is Transaction Malleability?**

Transaction malleability is the ability to modify a transaction's signature **without invalidating it**, thereby changing the transaction ID (TXID).

**The Core Problem:**

```
Legacy Transaction TXID Calculation:
TXID = SHA256(SHA256(entire_transaction_including_signatures))
                                        ^^^^^^^^^^^^^^^^^^^^
                                        Problem: Mutable!
```

**Why Signatures Are Malleable (ECDSA):**

DER signature encoding allows multiple valid representations:
```
Original:  304402201234567890abcdef...   (71 bytes)
Malleable: 3045022100001234567890abcdef... (72 bytes with zero padding)
```

**Both signatures:**
- ✅ Cryptographically valid
- ✅ Pass ECDSA verification
- ❌ Produce **different TXIDs**

**Why This Breaks Layer 2 Protocols:**

```
Lightning Channel Setup:

Funding TX (TXID_A) ──→ Commitment TX ──→ Timeout TX
     ↓                      ↓                 ↓
  Creates              References         References
  Channel             TXID_A             TXID_B

If TXID_A is malleable:
  → Commitment TX becomes invalid
  → Timeout TX becomes invalid
  → Entire channel unusable
  → Funds potentially locked!
```

**SegWit's Solution:**

```
SegWit Transaction TXID Calculation:
TXID = SHA256(SHA256(transaction_without_witness_data))
                                      ^^^^^^^^^^^^^^^^
                                      Excluded from TXID!
```

**Result:**
- 🔒 **Fixed TXIDs**: Signatures outside TXID calculation
- ⚡ **Layer 2 protocols**: Lightning Network becomes possible
- 🔗 **Transaction chains**: Pre-signed transactions work reliably

---

### Witness Structure: Segregated Signatures

**Legacy vs SegWit Structure:**

**Legacy Transaction:**
```
┌─────────────────────────────────────────┐
│ Version │ Inputs │ Outputs │ Locktime │  } All in TXID
│         │ ┌─────┐│         │          │
│         │ │ScSig││         │          │
│         │ └─────┘│         │          │
└─────────────────────────────────────────┘
     ↓
TXID = SHA256(SHA256(everything))
```

**SegWit Transaction:**
```
┌─────────────────────────────────────────┐
│ Version │ Inputs │ Outputs │ Locktime │  } Base Transaction
│         │ ┌─────┐│         │          │    (in TXID)
│         │ │Empty││         │          │
│         │ └─────┘│         │          │
└─────────────────────────────────────────┘
        } TXID = SHA256(SHA256(base only))

┌─────────────────────────────────────────┐
│        Witness Data (Separated)         │  } NOT in TXID
│  ┌─────────────────────────────────┐   │    (committed via
│  │ Signature │ Public Key          │   │     wtxid merkle)
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

**Key Insight:**
> Witness data is **committed** to blocks via witness merkle tree, but **excluded** from TXID calculation. This separation is called "witness segregation."

---

### SegWit Address Types

**P2WPKH (Pay-to-Witness-Public-Key-Hash):**

```
Bech32 Address: bc1qw508d6qejxtdg4y5r3zarvary0c5xw7kygt080
                ^^^^
                bc1q = Native SegWit (version 0)

Locking Script: 00 14 <20-byte-pubkey-hash>
                ^^    ^^^^^^^^^^^^^^^^^^^^^
                │     Public key hash
                Witness version 0
```

**Benefits:**
- 🪶 **Lower fees**: ~40% cheaper than Legacy
- ✅ **Better error detection**: Bech32 encoding
- 🔤 **Case insensitive**: Reduces copy-paste errors
- 🚀 **Native SegWit**: No wrapper needed

**P2SH-P2WPKH (Wrapped SegWit):**

```
Base58 Address: 3J98t1WpEZ73CNmQviecrnyiWrnqRhWNLy
                ^
                3 = P2SH wrapper

Locking Script: OP_HASH160 <20-byte-script-hash> OP_EQUAL
Redeem Script:  00 14 <20-byte-pubkey-hash>

Purpose: Backward compatibility with old wallets
```

**When to use:**
- **P2WPKH**: Modern wallets, lowest fees
- **P2SH-P2WPKH**: Legacy wallet compatibility

---

### Transaction Weight Units: Fee Optimization

**Weight Unit Formula (BIP 141):**
```
Weight = (Base Size × 4) + Witness Size

Base Size:    Everything except witness data
Witness Size: Signature + public key data
```

**Why × 4 multiplier?**

Creates **75% discount** for witness data:
```
Legacy P2PKH:
  Size: 250 bytes
  Weight: 250 × 4 = 1000 WU
  
SegWit P2WPKH:
  Base: 110 bytes → 110 × 4 = 440 WU
  Witness: 107 bytes → 107 × 1 = 107 WU
  Total Weight: 547 WU
  
Savings: 45% less weight = 45% lower fees!
```

**Virtual Size (vsize):**
```
vsize = ceil(weight / 4)

This is what fee estimators use!
```

---

### P2WPKH Stack Execution

**Locking Script (ScriptPubKey):**
```
00 14 1d7cd6c75c2e86c08d0bad1b4c982c3e4f33e7f5
```

**Witness Stack:**
```
Item 0: <signature>   (71 bytes)
Item 1: <public_key>  (33 bytes)
```

**Pattern Recognition:**

Bitcoin Core sees `OP_0 <20-bytes>` and interprets as **P2WPKH**, which is equivalent to:

```
OP_DUP OP_HASH160 <pubkey_hash> OP_EQUALVERIFY OP_CHECKSIG
```

**Execution Steps:**

**Initial State** (witness items loaded):
```
┌─────────────────────────────────┐
│ 021f8f6e70fc...f5 (public_key)  │ ← Top
│ 304402201234...78 (signature)   │
└─────────────────────────────────┘
```

**Step 1**: OP_DUP (duplicate public key)
```
┌─────────────────────────────────┐
│ 021f8f6e70fc...f5 (public_key)  │
│ 021f8f6e70fc...f5 (public_key)  │
│ 304402201234...78 (signature)   │
└─────────────────────────────────┘
```

**Step 2**: OP_HASH160 (hash public key)
```
┌─────────────────────────────────┐
│ 1d7cd6c75c2e...f5 (computed)    │
│ 021f8f6e70fc...f5 (public_key)  │
│ 304402201234...78 (signature)   │
└─────────────────────────────────┘
```

**Step 3**: PUSH expected_hash (from locking script)
```
┌─────────────────────────────────┐
│ 1d7cd6c75c2e...f5 (expected)    │
│ 1d7cd6c75c2e...f5 (computed)    │
│ 021f8f6e70fc...f5 (public_key)  │
│ 304402201234...78 (signature)   │
└─────────────────────────────────┘
```

**Step 4**: OP_EQUALVERIFY (compare hashes, remove if equal)
```
┌─────────────────────────────────┐
│ 021f8f6e70fc...f5 (public_key)  │
│ 304402201234...78 (signature)   │
└─────────────────────────────────┘
```
✅ Hashes match! Continue.

**Step 5**: OP_CHECKSIG (verify signature)
```
┌─────────────────────────────────┐
│ 1 (TRUE)                        │
└─────────────────────────────────┘
```

**✅ Transaction Valid!**

**Key Difference from Legacy:**
- No ScriptSig (empty)
- Authorization data in witness
- Execution model identical to P2PKH

---

### SegWit to Taproot Evolution

**SegWit establishes three critical foundations for Taproot:**

1. **Witness Structure**
   ```
   SegWit:  Witness = [signature, public_key]
   Taproot: Witness = [signature] or [signature, script, control_block]
   ```

2. **Version Byte System**
   ```
   OP_0  = Witness v0 (SegWit)
   OP_1  = Witness v1 (Taproot)
   OP_2+ = Future upgrades
   ```

3. **Weight Unit Economics**
   ```
   SegWit: 75% discount for witness data
   Taproot: Builds on this for even greater efficiency
   ```

**Taproot = SegWit + Schnorr + MAST**

---

## 💻 Example Code

### Example 1: Create Native SegWit (P2WPKH) Transaction

```python
from bitcoinutils.setup import setup
from bitcoinutils.utils import to_satoshis
from bitcoinutils.transactions import Transaction, TxInput, TxOutput
from bitcoinutils.keys import PrivateKey, P2wpkhAddress

def create_segwit_transaction():
    """
    Build a complete native SegWit transaction
    P2WPKH to P2WPKH on testnet
    """
    setup('testnet')
    
    # Create keys and addresses
    private_key = PrivateKey('cTALNpTpRbbxTCJ2A5Vq88UxT44w1PE2cYqiB3n4hRvzyCev1Wwo')
    public_key = private_key.get_public_key()
    
    # From address (our SegWit address)
    from_address = P2wpkhAddress(public_key.get_address().to_hash160())
    
    # To address (recipient SegWit address)
    to_address = P2wpkhAddress('tb1qw508d6qejxtdg4y5r3zarvary0c5xw7kygt080')
    
    print("="*60)
    print("SEGWIT TRANSACTION CREATION")
    print("="*60)
    print(f"From: {from_address.to_string()}")
    print(f"To:   {to_address.to_string()}")
    print()
    
    # UTXO to spend (replace with your actual UTXO!)
    utxo_txid = 'a1b2c3d4e5f6789012345678901234567890123456789012345678901234567890'
    utxo_vout = 0
    utxo_amount = 0.00029000  # 29,000 sats
    
    # Create transaction components
    txin = TxInput(utxo_txid, utxo_vout)
    
    # Send 28,500 sats, rest is fee (500 sats)
    txout = TxOutput(to_satoshis(0.00028500), to_address.to_script_pub_key())
    
    # Build unsigned transaction
    tx = Transaction([txin], [txout])
    
    print("="*60)
    print("UNSIGNED TRANSACTION")
    print("="*60)
    print(f"Hex: {tx.serialize()}")
    print()
    
    # SegWit signing: witness data, NOT scriptSig!
    print("="*60)
    print("SIGNING WITH WITNESS")
    print("="*60)
    
    signature = private_key.sign_input(tx, 0, from_address.to_script_pub_key())
    print(f"Signature: {signature[:40]}...")
    print(f"Public Key: {public_key.to_hex()}")
    print()
    
    # CRITICAL: Set witness, NOT script_sig
    txin.witness = [signature, public_key.to_hex()]
    
    # Verify scriptSig is empty
    print("="*60)
    print("SEGWIT STRUCTURE VERIFICATION")
    print("="*60)
    print(f"ScriptSig: '{txin.script_sig.to_hex() if txin.script_sig else '(empty)'}'")
    print(f"Witness Items: {len(txin.witness)}")
    print(f"  [0] Signature ({len(signature)//2} bytes)")
    print(f"  [1] Public Key ({len(public_key.to_hex())//2} bytes)")
    print()
    
    # Complete signed transaction
    signed_tx = tx.serialize()
    
    print("="*60)
    print("SIGNED TRANSACTION")
    print("="*60)
    print(signed_tx)
    print()
    
    # Size and weight analysis
    base_size = 110  # Approximate base size
    witness_size = len(signature)//2 + len(public_key.to_hex())//2 + 2  # +2 for length bytes
    weight = (base_size * 4) + witness_size
    vsize = (weight + 3) // 4  # Round up
    
    print("="*60)
    print("TRANSACTION METRICS")
    print("="*60)
    print(f"Base Size:    ~{base_size} bytes")
    print(f"Witness Size: ~{witness_size} bytes")
    print(f"Weight:       ~{weight} WU")
    print(f"Virtual Size: ~{vsize} vbytes")
    print()
    print(f"Fee: {utxo_amount - 0.00028500} BTC ({(utxo_amount - 0.00028500)*100000000:.0f} sats)")
    print(f"Fee Rate: ~{((utxo_amount - 0.00028500)*100000000 / vsize):.1f} sats/vbyte")
    print()
    
    return tx

if __name__ == "__main__":
    tx = create_segwit_transaction()
```

### Example 2: Decode and Analyze SegWit Transaction Structure

```python
def analyze_segwit_structure():
    """
    Decode a raw SegWit transaction and identify components
    """
    # Example signed SegWit transaction
    raw_tx = (
        "02000000"  # Version
        "00"        # Marker (SegWit indicator)
        "01"        # Flag (SegWit version)
        "01"        # Input count
        "a1b2c3d4e5f6789012345678901234567890123456789012345678901234567890"  # TXID
        "00000000"  # VOUT
        "00"        # ScriptSig length (empty)
        "ffffffff"  # Sequence
        "01"        # Output count
        "146f000000000000"  # Value (28,500 sats)
        "16"        # ScriptPubKey length (22 bytes)
        "00141d7cd6c75c2e86c08d0bad1b4c982c3e4f33e7f5"  # ScriptPubKey
        # Witness data
        "02"        # Witness stack items
        "47"        # Signature length (71 bytes)
        "304402201234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef02201234567801"
        "21"        # Public key length (33 bytes)
        "021f8f6e70fcb96c29e6c87b47ac21e5e9c2e9e5f5f5f5f5f5f5f5f5"
        "00000000"  # Locktime
    )
    
    print("="*60)
    print("SEGWIT TRANSACTION STRUCTURE ANALYSIS")
    print("="*60)
    print()
    
    print("[VERSION]        02000000")
    print("[MARKER]         00          ← SegWit indicator")
    print("[FLAG]           01          ← SegWit version")
    print("[INPUT_COUNT]    01")
    print("[TXID]           a1b2c3d4... (32 bytes)")
    print("[VOUT]           00000000")
    print("[SCRIPTSIG_LEN]  00          ← Empty! (SegWit feature)")
    print("[SEQUENCE]       ffffffff")
    print("[OUTPUT_COUNT]   01")
    print("[VALUE]          146f000000000000 (28,500 sats)")
    print("[SCRIPT_LEN]     16          (22 bytes)")
    print("[SCRIPTPUBKEY]   00141d7cd6c7... (witness v0 pubkey hash)")
    print()
    print("--- WITNESS DATA (Segregated) ---")
    print()
    print("[WITNESS_ITEMS]  02          ← 2 items")
    print("[SIG_LEN]        47          (71 bytes)")
    print("[SIGNATURE]      30440220... (ECDSA signature)")
    print("[PK_LEN]         21          (33 bytes)")
    print("[PUBLIC_KEY]     021f8f6e... (compressed pubkey)")
    print()
    print("[LOCKTIME]       00000000")
    print()
    
    print("="*60)
    print("KEY OBSERVATIONS")
    print("="*60)
    print("✓ Marker (00) + Flag (01) identify SegWit transaction")
    print("✓ ScriptSig is empty (00)")
    print("✓ Witness data comes AFTER outputs")
    print("✓ TXID calculated WITHOUT witness data")
    print()

if __name__ == "__main__":
    analyze_segwit_structure()
```

### Example 3: Compare Transaction Sizes (Legacy vs SegWit)

```python
def compare_transaction_types():
    """
    Compare size and cost of Legacy vs SegWit transactions
    """
    print("="*60)
    print("TRANSACTION TYPE COMPARISON")
    print("="*60)
    print()
    
    # Typical sizes for 1-in, 1-out transactions
    legacy_p2pkh = {
        'type': 'Legacy P2PKH',
        'total_size': 250,
        'base_size': 250,
        'witness_size': 0,
        'weight': 250 * 4,
        'vsize': 250
    }
    
    native_segwit = {
        'type': 'Native SegWit (P2WPKH)',
        'total_size': 193,
        'base_size': 110,
        'witness_size': 107,
        'weight': (110 * 4) + 107,
        'vsize': ((110 * 4) + 107 + 3) // 4
    }
    
    wrapped_segwit = {
        'type': 'P2SH-wrapped SegWit',
        'total_size': 217,
        'base_size': 133,
        'witness_size': 107,
        'weight': (133 * 4) + 107,
        'vsize': ((133 * 4) + 107 + 3) // 4
    }
    
    # Calculate savings
    fee_rate = 10  # sats/vbyte
    
    for tx_type in [legacy_p2pkh, native_segwit, wrapped_segwit]:
        fee = tx_type['vsize'] * fee_rate
        
        print(f"{tx_type['type']}")
        print(f"  Total Size:   {tx_type['total_size']} bytes")
        print(f"  Base Size:    {tx_type['base_size']} bytes")
        print(f"  Witness Size: {tx_type['witness_size']} bytes")
        print(f"  Weight:       {tx_type['weight']} WU")
        print(f"  Virtual Size: {tx_type['vsize']} vbytes")
        print(f"  Fee (@10 sat/vb): {fee} sats")
        
        if tx_type['type'] != 'Legacy P2PKH':
            savings = (legacy_p2pkh['vsize'] - tx_type['vsize']) / legacy_p2pkh['vsize'] * 100
            fee_savings = (legacy_p2pkh['vsize'] - tx_type['vsize']) * fee_rate
            print(f"  Savings: {savings:.1f}% ({fee_savings} sats)")
        
        print()
    
    print("="*60)
    print("RECOMMENDATION")
    print("="*60)
    print("✓ Use Native SegWit (P2WPKH) for lowest fees")
    print("✓ Use P2SH-wrapped only for backward compatibility")
    print("✓ Native SegWit ~40% cheaper than Legacy")
    print()

if __name__ == "__main__":
    compare_transaction_types()
```

### Example 4: Verify SegWit Address Encoding

```python
from bitcoinutils.keys import P2wpkhAddress, PublicKey
import hashlib

def verify_segwit_address():
    """
    Manually verify P2WPKH address encoding
    """
    # Public key
    pubkey_hex = '021f8f6e70fcb96c29e6c87b47ac21e5e9c2e9e5f5f5f5f5f5f5f5f5'
    pubkey = PublicKey(pubkey_hex)
    
    # Calculate Hash160 (SHA256 → RIPEMD160)
    pubkey_bytes = bytes.fromhex(pubkey_hex)
    sha256_hash = hashlib.sha256(pubkey_bytes).digest()
    hash160 = hashlib.new('ripemd160', sha256_hash).digest()
    
    print("="*60)
    print("P2WPKH ADDRESS ENCODING VERIFICATION")
    print("="*60)
    print()
    print(f"Public Key:    {pubkey_hex}")
    print(f"SHA256:        {sha256_hash.hex()}")
    print(f"Hash160:       {hash160.hex()}")
    print()
    
    # Create P2WPKH address
    p2wpkh_addr = P2wpkhAddress.from_hash(hash160.hex())
    
    print("="*60)
    print("P2WPKH ADDRESS")
    print("="*60)
    print(f"Address: {p2wpkh_addr.to_string()}")
    print()
    
    # Locking script
    locking_script = p2wpkh_addr.to_script_pub_key()
    
    print("="*60)
    print("LOCKING SCRIPT (ScriptPubKey)")
    print("="*60)
    print(f"Hex: {locking_script.to_hex()}")
    print()
    print("Breakdown:")
    print(f"  00  = OP_0 (witness version)")
    print(f"  14  = PUSH 20 bytes")
    print(f"  {hash160.hex()} = Hash160(pubkey)")
    print()
    
    return p2wpkh_addr

if __name__ == "__main__":
    addr = verify_segwit_address()
```

---

## 🧪 Practice Task

### Task 1: Build Your First Native SegWit Transaction

**Objective**: Create and broadcast a complete P2WPKH transaction on testnet.

**Steps:**

1. Generate a SegWit private key
2. Derive the P2WPKH address (bc1q...)
3. Get testnet coins to your address
4. Build a transaction spending to another SegWit address
5. Sign with witness data (NOT scriptSig)
6. Broadcast and verify on explorer

**Verification Checklist:**
- [ ] Address starts with `tb1q` (testnet SegWit)
- [ ] Transaction ScriptSig is empty
- [ ] Witness contains [signature, public_key]
- [ ] Transaction confirms successfully

### Task 2: Transaction Malleability Demonstration

**Challenge**: Understand why Legacy is malleable but SegWit isn't.

**Part A**: Given this Legacy transaction:
```
TXID = hash(version + inputs + outputs + locktime)
                      ^^^^^^
                      includes signatures!
```

**Question**: If you modify the DER encoding of a signature (e.g., add zero padding), does:
1. The signature remain cryptographically valid?
2. The TXID change?
3. Why is this a problem for Lightning?

**Part B**: Given a SegWit transaction:
```
TXID = hash(version + inputs + outputs + locktime)
                              (witness excluded)
```

**Question**: Can signature modification change the TXID? Why or why not?

### Task 3: Weight Unit Calculation

**Given:**
- Legacy P2PKH transaction: 250 bytes
- SegWit P2WPKH transaction: 110 base bytes + 107 witness bytes

**Calculate:**
1. Legacy weight units
2. SegWit weight units
3. SegWit virtual size (vsize)
4. Percentage savings
5. Fee savings at 20 sats/vbyte

**Show your work!**

---

## 📘 Reading

### Required Reading
- **Mastering Taproot** - Chapter 4: Building SegWit Transactions (pages 42-55)

### BIP References
- **BIP 141**: Segregated Witness (Consensus layer) - https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki
- **BIP 143**: Transaction Signature Verification for Version 0 Witness Program
- **BIP 144**: Segregated Witness (Peer Services)
- **BIP 173**: Bech32 address format for native v0-16 witness outputs

### Supplementary Resources
- **Bitcoin Optech**: SegWit Benefits - https://bitcoinops.org/en/topics/segregated-witness/
- **Jimmy Song**: "Understanding SegWit" article series
- **Pieter Wuille**: SegWit presentation at Scaling Bitcoin

---

## 💬 Discussion Prompts

### Question 1: Malleability Impact
**"Before SegWit, exchanges had to wait for 6 confirmations before crediting deposits. With SegWit, some exchanges credit after 1-2 confirmations. Why did malleability require extra confirmations, and how does SegWit make earlier crediting safe?"**

*Hint: Think about TXID changes and child transactions.*

### Question 2: Witness Discount Rationale
**"Why does BIP 141 give witness data a 75% discount (4× multiplier for base) instead of treating all bytes equally? What blockchain properties does this incentive structure encourage?"**

*Hint: Consider UTXO set growth vs. signature data storage.*

### Question 3: SegWit Adoption
**"SegWit activated in August 2017. Why did it take years for adoption to reach >80%? What were the incentives and barriers for wallet developers and exchanges?"**

---

## 🎙️ Instructor's Note

Welcome to the witness revolution!

SegWit is often misunderstood as "just" a malleability fix, but it's so much more. It's a **complete redesign** of how Bitcoin handles authorization data, and it enables the entire Layer 2 ecosystem we see today.

The key insight: **separate the authorization data from the transaction data**. This simple architectural change cascades into massive improvements:

1. **Fixed TXIDs** → Lightning becomes possible
2. **Weight units** → Economic incentives for efficient scripts
3. **Version byte system** → Clean upgrade path to Taproot
4. **Witness structure** → Foundation for script trees

When you're tracing through P2WPKH execution, notice how it's **identical** to P2PKH—just with data coming from a different place. Bitcoin Core recognizes the `OP_0 <20-bytes>` pattern and says "oh, this is a P2WPKH, execute it like P2PKH but pull data from witness."

This pattern recognition becomes even more important with Taproot. When the node sees `OP_1 <32-bytes>`, it knows "this is Taproot, execute the special rules for witness v1."

The weight unit calculation might seem arbitrary (why × 4?), but it's brilliant economics. By making witness data cheaper, SegWit incentivizes moving authorization data off the base transaction, which keeps the UTXO set smaller and the network more scalable. Taproot amplifies this benefit even further with signature aggregation.

Next week: **Taproot introduction**! We'll see how Schnorr signatures enable key aggregation and how key tweaking creates the foundation for Taproot's privacy magic.

This is where everything comes together!

—Aaron

---

## 📌 Key Takeaways

✅ SegWit solves transaction malleability by excluding witness from TXID  
✅ Witness segregation = authorization separate from transaction structure  
✅ P2WPKH = SegWit v0, identical execution to P2PKH  
✅ Weight units: (base × 4) + witness = 75% discount for witness  
✅ Bech32 addresses (bc1q) are native SegWit, better than Base58  
✅ Fixed TXIDs enable Lightning Network and Layer 2 protocols  
✅ SegWit is the foundation that makes Taproot possible

---

**Previous**: [← Lesson 3: P2SH Multi-signature](./lesson-03-p2sh-multisig.md)  
**Next**: [Lesson 5: Taproot Introduction: Schnorr and Key Tweaking →](./lesson-05-taproot-schnorr.md)

---

*This lesson is part of the Taproot Script Engineering Cohort at Chaincode Labs (2026)*