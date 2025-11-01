# Lesson 3: P2SH Multi-signature and Time Locks

**Week**: 3  
**Duration**: Self-paced (estimated 5-6 hours)  
**Prerequisites**: Lessons 1-2 (Keys, Addresses, P2PKH)

---

## 🎯 Learning Objectives

By the end of this lesson, you will be able to:

1. Understand P2SH's two-phase execution model (hash verification → script execution)
2. Construct 2-of-3 multisignature treasury contracts
3. Implement CSV (CheckSequenceVerify) time-locked inheritance scripts
4. Trace through P2SH stack execution with stack reset mechanism
5. Recognize P2SH's limitations and how Taproot addresses them
6. Build and broadcast real P2SH transactions on testnet

---

## 🧩 Key Concepts

### P2SH Architecture: Scripts Behind the Hash

P2SH (Pay-to-Script-Hash) revolutionized Bitcoin by enabling **complex scripts hidden behind simple addresses**.

**The Core Innovation:**
```
Instead of:  OP_2 <pk1> <pk2> <pk3> OP_3 OP_CHECKMULTISIG
Use:         OP_HASH160 <script_hash> OP_EQUAL
```

**P2SH Advantages:**
- ✅ **Compact addresses**: Complex scripts → 20-byte hash
- ✅ **Cost shift**: Recipient pays for script reveal, not sender
- ✅ **Privacy**: Script not revealed until spent
- ✅ **Backward compatibility**: Works with all Bitcoin software

**P2SH Two-Phase Execution Model:**

```
Phase 1: Hash Verification
  ScriptSig + ScriptPubKey
  ↓
  Verify: HASH160(redeem_script) == expected_hash
  
Phase 2: Script Execution (if Phase 1 succeeds)
  Stack Reset → Execute redeem_script
  ↓
  Verify: Redeem script conditions satisfied
```

**Critical Mechanism: Stack Reset**

Bitcoin Core recognizes the P2SH pattern `OP_HASH160 <hash> OP_EQUAL` and triggers special handling:

1. **Phase 1 completes** with TRUE on stack
2. **Stack resets** to post-ScriptSig state (discards TRUE)
3. **Redeem script extracted** from original ScriptSig
4. **Phase 2 begins** with clean stack executing redeem script

This two-phase model is the direct ancestor of Taproot's commit-reveal pattern!

---

### Multi-signature Treasury: 2-of-3 Corporate Security

**Business Scenario: Startup Treasury**

Consider a blockchain startup with three stakeholders:
- **Alice (CEO)**: Operational authority
- **Bob (CTO)**: Technical oversight  
- **Carol (CFO)**: Financial controls

**Policy**: Any **2-of-3** signatures required for fund movement.

**Why 2-of-3?**
- ❌ **Not 3-of-3**: Prevents operational paralysis if one key lost
- ❌ **Not 1-of-3**: Prevents single person risk
- ✅ **2-of-3**: Balances security and operational flexibility

### Redeem Script Construction

```python
redeem_script = Script([
    'OP_2',              # Require 2 signatures
    alice_pk,            # Alice's public key (33 bytes)
    bob_pk,              # Bob's public key (33 bytes)
    carol_pk,            # Carol's public key (33 bytes)
    'OP_3',              # Total of 3 keys
    'OP_CHECKMULTISIG'   # Multisig verification opcode
])
```

**Serialized Format:**
```
52 21 02898711... 21 0284b595... 21 0317aa89... 53 ae

52 = OP_2
21 = PUSH 33 bytes (compressed public key)
53 = OP_3
ae = OP_CHECKMULTISIG
```

**P2SH Address Generation:**
```
1. Serialize redeem script → bytes
2. Hash160(redeem_script) → 20-byte hash
3. Add version (0xc4 testnet, 0x05 mainnet)
4. Base58Check encode → Address starts with '2' (testnet)
```

---

### OP_CHECKMULTISIG: The Multisig Workhorse

**Stack Consumption Pattern:**
```
Input stack:
  OP_0          // Bug workaround (extra item)
  <sig1>        // First signature
  <sig2>        // Second signature
  OP_2          // Number of signatures
  <pk1>         // First public key
  <pk2>         // Second public key
  <pk3>         // Third public key
  OP_3          // Number of public keys

OP_CHECKMULTISIG consumes:
  - Key count (3)
  - All 3 public keys
  - Signature count (2)
  - All 2 signatures
  - Extra item (OP_0, due to off-by-one bug)

Output:
  TRUE or FALSE
```

**The Famous OP_CHECKMULTISIG Bug:**

Bitcoin's original implementation has an off-by-one error that pops an extra item from the stack. This is **consensus code** and cannot be changed without a hard fork.

**Solution:** Always push `OP_0` as the first item in ScriptSig for multisig scripts.

---

### CSV Time Locks: Relative Time Constraints

**CSV (CheckSequenceVerify) - BIP 112**

Enables **relative time locks**: spending delayed relative to when UTXO was created.

**Use Cases:**
- 🏦 **Digital inheritance**: Heirs access after inactivity period
- ⚡ **Lightning Network**: Settlement delay for dispute resolution
- 🔒 **Escrow**: Automatic release after time expires

**CSV Script Pattern:**
```
<delay_blocks>
OP_CHECKSEQUENCEVERIFY
OP_DROP
<standard_script>  // e.g., P2PKH
```

**BIP 68 Sequence Number Encoding:**
```
nSequence field in transaction input:
  Bit 31:    0 = enable, 1 = disable
  Bit 22:    0 = blocks, 1 = 512-second intervals
  Bits 0-15: Relative lock time value

Example: 3 blocks delay
  nSequence = 0x00000003
```

---

### P2SH Stack Execution: Complete Trace

**Transaction Example:**
```
TXID: e68bef534c7536300c3ae5ccd0f79e031cab29d262380a37269151e8ba
```

**Phase 1: Hash Verification**

**Initial Stack** (empty):
```
┌────────────────────────────────┐
│ (empty)                        │
└────────────────────────────────┘
```

**Step 1**: PUSH OP_0 (multisig bug workaround)
```
┌────────────────────────────────┐
│ 00                             │
└────────────────────────────────┘
```

**Step 2**: PUSH Alice's signature
```
┌────────────────────────────────┐
│ 30440220694f...7a6501 (alice)  │
│ 00                             │
└────────────────────────────────┘
```

**Step 3**: PUSH Bob's signature
```
┌────────────────────────────────┐
│ 3044022065f8...fd9e01 (bob)    │
│ 30440220694f...7a6501 (alice)  │
│ 00                             │
└────────────────────────────────┘
```

**Step 4**: PUSH redeem_script
```
┌────────────────────────────────┐
│ 522102898711...ae (redeem)     │
│ 3044022065f8...fd9e01 (bob)    │
│ 30440220694f...7a6501 (alice)  │
│ 00                             │
└────────────────────────────────┘
```

**Step 5**: OP_HASH160 (hash the redeem script)
```
┌────────────────────────────────┐
│ dd81b5beb3d8...ca (computed)   │
│ 3044022065f8...fd9e01 (bob)    │
│ 30440220694f...7a6501 (alice)  │
│ 00                             │
└────────────────────────────────┘
```

**Step 6**: PUSH expected_hash (from locking script)
```
┌────────────────────────────────┐
│ dd81b5beb3d8...ca (expected)   │
│ dd81b5beb3d8...ca (computed)   │
│ 3044022065f8...fd9e01 (bob)    │
│ 30440220694f...7a6501 (alice)  │
│ 00                             │
└────────────────────────────────┘
```

**Step 7**: OP_EQUAL
```
┌────────────────────────────────┐
│ 1 (TRUE)                       │
│ 3044022065f8...fd9e01 (bob)    │
│ 30440220694f...7a6501 (alice)  │
│ 00                             │
└────────────────────────────────┘
```

**✅ Phase 1 Complete: Hash matches!**

---

**CRITICAL: Stack Reset Mechanism**

Bitcoin Core detects P2SH pattern and:
1. Discards the TRUE from Phase 1
2. Resets stack to **post-ScriptSig** state
3. Extracts redeem script for Phase 2

**Stack After Reset:**
```
┌────────────────────────────────┐
│ 3044022065f8...fd9e01 (bob)    │
│ 30440220694f...7a6501 (alice)  │
│ 00                             │
└────────────────────────────────┘
```

---

**Phase 2: Redeem Script Execution**

**Redeem Script:**
```
OP_2 <alice_pk> <bob_pk> <carol_pk> OP_3 OP_CHECKMULTISIG
```

**Step 8**: OP_2 (push required signature count)
```
┌────────────────────────────────┐
│ 2                              │
│ 3044022065f8...fd9e01 (bob)    │
│ 30440220694f...7a6501 (alice)  │
│ 00                             │
└────────────────────────────────┘
```

**Steps 9-11**: PUSH alice_pk, bob_pk, carol_pk
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

**Step 12**: OP_3 (push total key count)
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

**Step 13**: OP_CHECKMULTISIG

Verification process:
1. ✅ Alice's signature verified against Alice's public key
2. ✅ Bob's signature verified against Bob's public key
3. ✅ Required threshold (2-of-3) satisfied

```
┌────────────────────────────────┐
│ 1 (TRUE)                       │
└────────────────────────────────┘
```

**✅✅ Transaction Valid: Both phases succeeded!**

---

## 💻 Example Code

### Example 1: Create 2-of-3 Multisig P2SH Address

```python
from bitcoinutils.setup import setup
from bitcoinutils.keys import PrivateKey
from bitcoinutils.script import Script
from bitcoinutils.keys import P2shAddress

def create_multisig_p2sh():
    """
    Create a 2-of-3 multisignature P2SH address
    Corporate treasury example
    """
    setup('testnet')
    
    # Stakeholder public keys
    alice_pk = '02898711e6bf63f5cbe1b38c05e89d6c391c59e9f8f695da44bf3d20ca674c8519'
    bob_pk = '0284b5951609b76619a1ce7f48977b4312ebe226987166ef044bfb374ceef63af5'
    carol_pk = '0317aa89b43f46a0c0cdbd9a302f2508337ba6a06d123854481b52de9c20996011'
    
    print("="*60)
    print("2-OF-3 MULTISIG P2SH ADDRESS CREATION")
    print("="*60)
    print(f"Alice's Public Key:  {alice_pk}")
    print(f"Bob's Public Key:    {bob_pk}")
    print(f"Carol's Public Key:  {carol_pk}")
    print()
    
    # 2-of-3 multisig redeem script
    redeem_script = Script([
        'OP_2',              # Require 2 signatures
        alice_pk,            # Alice's public key
        bob_pk,              # Bob's public key
        carol_pk,            # Carol's public key
        'OP_3',              # Total of 3 keys
        'OP_CHECKMULTISIG'   # Multisig verification
    ])
    
    print("="*60)
    print("REDEEM SCRIPT")
    print("="*60)
    print(f"Hex: {redeem_script.to_hex()}")
    print(f"Asm: {redeem_script.to_string()}")
    print(f"Size: {len(redeem_script.to_bytes())} bytes")
    print()
    
    # Generate P2SH address
    p2sh_addr = P2shAddress.from_script(redeem_script)
    
    print("="*60)
    print("P2SH ADDRESS")
    print("="*60)
    print(f"Address: {p2sh_addr.to_string()}")
    print(f"Script Hash: {p2sh_addr.to_hash160()}")
    print()
    
    # Locking script (what appears on blockchain)
    locking_script = p2sh_addr.to_script_pub_key()
    print("="*60)
    print("LOCKING SCRIPT (ScriptPubKey)")
    print("="*60)
    print(f"Hex: {locking_script.to_hex()}")
    print(f"Asm: {locking_script.to_string()}")
    print()
    
    print("="*60)
    print("SEND TESTNET COINS TO THIS ADDRESS!")
    print("="*60)
    
    return p2sh_addr, redeem_script

if __name__ == "__main__":
    addr, redeem_script = create_multisig_p2sh()
```

### Example 2: Spend from 2-of-3 Multisig (Alice + Bob Sign)

```python
from bitcoinutils.setup import setup
from bitcoinutils.utils import to_satoshis
from bitcoinutils.transactions import Transaction, TxInput, TxOutput
from bitcoinutils.keys import PrivateKey, P2wpkhAddress
from bitcoinutils.script import Script

def spend_multisig_p2sh():
    """
    Spend from 2-of-3 multisig P2SH
    Alice and Bob authorize the payment
    """
    setup('testnet')
    
    # Private keys (keep secure in production!)
    alice_sk = PrivateKey('cMahea7zqjxrtgAbB7LSGbcQUr1uX1ojuat9jZodMN87JcbXMTcA')
    bob_sk = PrivateKey('cMahea7zqjxrtgAbB7LSGbcQUr1uX1ojuat9jZodMN87K8CaLbxz')
    
    # Redeem script (same as used in creation)
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
    
    # Previous UTXO details (replace with your UTXO!)
    utxo_txid = '4b869865bc4a156d7e0ba14590b5c8971e57b8198af64d88872558ca88a8ba5f'
    utxo_vout = 0
    utxo_amount = 0.00001600  # 1,600 satoshis
    
    # Recipient address
    recipient_address = P2wpkhAddress('tb1qckeg66a6jx3xjw5mrpmte5ujjv3cjrajtvm9r4')
    
    # Create transaction
    txin = TxInput(utxo_txid, utxo_vout)
    
    # Send 888 sats, rest is fee
    txout = TxOutput(to_satoshis(0.00000888), recipient_address.to_script_pub_key())
    
    tx = Transaction([txin], [txout])
    
    print("="*60)
    print("SPENDING FROM MULTISIG P2SH")
    print("="*60)
    print(f"Input UTXO: {utxo_txid}:{utxo_vout}")
    print(f"Amount: {utxo_amount} BTC")
    print(f"Recipient: {recipient_address.to_string()}")
    print(f"Sending: 0.00000888 BTC")
    print(f"Fee: {utxo_amount - 0.00000888} BTC")
    print()
    
    # Sign with Alice and Bob's keys
    print("="*60)
    print("SIGNING PHASE")
    print("="*60)
    print("Alice signing...")
    alice_sig = alice_sk.sign_input(tx, 0, redeem_script)
    print(f"Alice signature: {alice_sig[:40]}...")
    
    print("Bob signing...")
    bob_sig = bob_sk.sign_input(tx, 0, redeem_script)
    print(f"Bob signature: {bob_sig[:40]}...")
    print()
    
    # Construct ScriptSig
    # ORDER MATTERS: OP_0, signatures in order matching public keys
    txin.script_sig = Script([
        'OP_0',              # OP_CHECKMULTISIG bug workaround
        alice_sig,           # First signature
        bob_sig,             # Second signature
        redeem_script.to_hex()  # Reveal the redeem script
    ])
    
    print("="*60)
    print("UNLOCKING SCRIPT (ScriptSig)")
    print("="*60)
    print(f"Components:")
    print(f"  1. OP_0 (bug workaround)")
    print(f"  2. Alice's signature (71 bytes)")
    print(f"  3. Bob's signature (71 bytes)")
    print(f"  4. Redeem script ({len(redeem_script.to_bytes())} bytes)")
    print()
    
    # Get signed transaction
    signed_tx = tx.serialize()
    
    print("="*60)
    print("SIGNED TRANSACTION")
    print("="*60)
    print(signed_tx)
    print()
    print(f"Transaction size: {tx.get_size()} bytes")
    print()
    print("Ready to broadcast!")
    print("Use: https://mempool.space/testnet/tx/push")
    
    return tx

if __name__ == "__main__":
    tx = spend_multisig_p2sh()
```

### Example 3: CSV Time-Locked Inheritance

```python
from bitcoinutils.setup import setup
from bitcoinutils.transactions import Sequence
from bitcoinutils.constants import TYPE_RELATIVE_TIMELOCK
from bitcoinutils.script import Script
from bitcoinutils.keys import P2shAddress, PrivateKey

def create_csv_inheritance():
    """
    Create a 3-block time-locked inheritance script
    Beneficiary can only spend after 3 blocks
    """
    setup('testnet')
    
    # Beneficiary's information
    beneficiary_sk = PrivateKey('cTALNpTpRbbxTCJ2A5Vq88UxT44w1PE2cYqiB3n4hRvzyCev1Wwo')
    beneficiary_pk = beneficiary_sk.get_public_key()
    beneficiary_pkh = beneficiary_pk.to_hash160()
    
    # 3-block relative time lock
    relative_blocks = 3
    seq = Sequence(TYPE_RELATIVE_TIMELOCK, relative_blocks)
    
    print("="*60)
    print("CSV TIME-LOCKED INHERITANCE SCRIPT")
    print("="*60)
    print(f"Delay: {relative_blocks} blocks")
    print(f"Sequence value: 0x{seq.for_script():08x}")
    print(f"Beneficiary: {beneficiary_pk.to_address().to_string()}")
    print()
    
    # Combined CSV + P2PKH script
    redeem_script = Script([
        seq.for_script(),          # PUSH 3
        'OP_CHECKSEQUENCEVERIFY',  # Verify time lock
        'OP_DROP',                 # Remove delay value from stack
        'OP_DUP',                  # Standard P2PKH starts here
        'OP_HASH160',
        beneficiary_pkh,
        'OP_EQUALVERIFY',
        'OP_CHECKSIG'
    ])
    
    print("="*60)
    print("REDEEM SCRIPT BREAKDOWN")
    print("="*60)
    print(f"Hex: {redeem_script.to_hex()}")
    print()
    print("Components:")
    print(f"  1. PUSH 3 (delay value)")
    print(f"  2. OP_CHECKSEQUENCEVERIFY")
    print(f"  3. OP_DROP")
    print(f"  4-8. Standard P2PKH")
    print()
    
    # Generate P2SH address
    p2sh_addr = P2shAddress.from_script(redeem_script)
    
    print("="*60)
    print("P2SH ADDRESS")
    print("="*60)
    print(f"Address: {p2sh_addr.to_string()}")
    print()
    print("⏰ Funds sent to this address can only be")
    print("   spent by beneficiary after 3 blocks!")
    print()
    
    return p2sh_addr, redeem_script, seq

if __name__ == "__main__":
    addr, redeem, seq = create_csv_inheritance()
```

### Example 4: Spend CSV Time-Locked UTXO

```python
def spend_csv_inheritance():
    """
    Spend from CSV-locked P2SH after time lock expires
    """
    setup('testnet')
    
    # Get the CSV script components
    beneficiary_sk = PrivateKey('cTALNpTpRbbxTCJ2A5Vq88UxT44w1PE2cYqiB3n4hRvzyCev1Wwo')
    beneficiary_pk = beneficiary_sk.get_public_key()
    
    # Must wait 3 blocks after UTXO creation
    seq = Sequence(TYPE_RELATIVE_TIMELOCK, 3)
    
    # Recreate redeem script
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
    
    # UTXO details
    utxo_txid = '34f5bf0cf328d77059b5674e71442ded8cdcfc723d0136733e0dbf1808619'
    utxo_vout = 0
    
    # CRITICAL: Set sequence in transaction input
    txin = TxInput(utxo_txid, utxo_vout, sequence=seq.for_input_sequence())
    
    recipient = P2wpkhAddress('tb1qckeg66a6jx3xjw5mrpmte5ujjv3cjrajtvm9r4')
    txout = TxOutput(to_satoshis(0.00000500), recipient.to_script_pub_key())
    
    tx = Transaction([txin], [txout])
    
    print("="*60)
    print("SPENDING TIME-LOCKED UTXO")
    print("="*60)
    print(f"Sequence value: 0x{seq.for_input_sequence():08x}")
    print(f"Time lock: {3} blocks (must have passed)")
    print()
    
    # Sign the transaction
    sig = beneficiary_sk.sign_input(tx, 0, redeem_script)
    
    # Construct ScriptSig
    txin.script_sig = Script([
        sig,                    # Signature for P2PKH
        beneficiary_pk.to_hex(),  # Public key for P2PKH
        redeem_script.to_hex()    # Reveal the script
    ])
    
    signed_tx = tx.serialize()
    
    print("="*60)
    print("SIGNED TRANSACTION")
    print("="*60)
    print(signed_tx)
    print()
    print("⚠️  This will fail if fewer than 3 blocks have passed!")
    print("    Error: 'non-BIP68-final'")
    print()
    
    return tx

if __name__ == "__main__":
    tx = spend_csv_inheritance()
```

---

## 🧪 Practice Task

### Task 1: Build Your Own 2-of-3 Multisig Treasury

**Objective**: Create a complete 2-of-3 multisig setup on testnet.

**Steps:**

1. Generate three key pairs (Alice, Bob, Carol)
2. Create the 2-of-3 redeem script
3. Generate P2SH address
4. Send testnet coins to the address
5. Create transaction signed by Alice + Bob
6. Broadcast and verify on explorer

**Deliverables:**
- P2SH address
- Redeem script (hex)
- Transaction ID after successful spend
- Screenshot of transaction on explorer

**Bonus Challenge**: Try spending with Bob + Carol instead!

### Task 2: Implement 3-Block Time Lock

**Scenario**: Create a digital inheritance script that unlocks after 3 blocks.

**Requirements:**
1. Use CSV with 3-block delay
2. Combine with P2PKH for beneficiary
3. Test transaction creation
4. Calculate minimum fee for size

**Question**: What happens if you try to broadcast before 3 blocks? Test it!

### Task 3: P2SH Stack Execution Trace

**Challenge**: Given this P2SH locking script:
```
a914dd81b5beb3d8ed72ea5f19f93d2e0b12145cb0ca87
```

And this unlocking script:
```
00 <sig1> <sig2> <redeem_script>
```

**Tasks:**
1. Identify the two execution phases
2. Draw the stack at each step of Phase 1
3. Explain the stack reset mechanism
4. Draw the stack at each step of Phase 2
5. At which step is signature verification performed?

---

## 📘 Reading

### Required Reading
- **Mastering Taproot** - Chapter 3: P2SH Script Engineering (pages 27-41)

### BIP References
- **BIP 16**: Pay to Script Hash - https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki
- **BIP 112**: CHECKSEQUENCEVERIFY - https://github.com/bitcoin/bips/blob/master/bip-0112.mediawiki
- **BIP 68**: Relative lock-time using consensus-enforced sequence numbers - https://github.com/bitcoin/bips/blob/master/bip-0068.mediawiki

### Supplementary Resources
- **Bitcoin Wiki**: Multisignature - https://en.bitcoin.it/wiki/Multisignature
- **Bitcoin Optech**: CheckSequenceVerify - https://bitcoinops.org/en/topics/op_checksequenceverify/
- **Andreas Antonopoulos**: "Mastering Bitcoin" Chapter 7 (Advanced Transactions)

---

## 💬 Discussion Prompts

### Question 1: P2SH Privacy Trade-offs
**"P2SH hides script complexity behind a hash, but the full script is revealed when spent. How does this compare to Taproot's approach? What are the privacy implications when only using one branch of a complex multi-conditional script?"**

*Hint: Consider what information leaks on-chain at spending time.*

### Question 2: The Stack Reset Mechanism
**"Why does Bitcoin Core need to reset the stack between Phase 1 and Phase 2 of P2SH execution? What would happen if the TRUE from Phase 1 remained on the stack during redeem script execution?"**

*Hint: Think about how OP_CHECKMULTISIG or other opcodes might be affected.*

### Question 3: CSV vs CLTV
**"CSV (CheckSequenceVerify) uses relative time, while CLTV (CheckLockTimeVerify) uses absolute time. For digital inheritance scenarios, why might CSV be preferred? When would CLTV be better?"**

*Hint: Consider what happens if the funding transaction is delayed.*

---

## 🎙️ Instructor's Note

Congratulations on reaching the P2SH milestone! This is where Bitcoin Script becomes truly powerful.

The two-phase execution model—hash verification then script execution—is elegant but subtle. The stack reset mechanism trips up many developers, so make sure you really understand why Bitcoin Core discards that TRUE and resets to the post-ScriptSig state. This isn't arbitrary; it's necessary to prevent the Phase 1 result from interfering with Phase 2 logic.

The multisig example shows P2SH's strength: complex authorization requirements hidden behind a simple 34-character address. But notice the limitation: **every detail of the redeem script must be revealed when spending**, even unused branches. If you have 5 possible spending conditions and only use one, all 5 are exposed on-chain.

This is why Taproot is revolutionary. It uses a similar commit-reveal pattern (like P2SH's hash commitment), but with a critical improvement: **only the executed path is revealed**. Unused branches remain completely hidden. We'll see this in action starting in Lesson 5.

CSV time locks are fascinating because they're **relative**, not absolute. The clock starts ticking when the UTXO is confirmed, not at some predetermined date. This makes them perfect for inheritance (3 months of inactivity), payment channels (dispute resolution period), or any scenario where timing is relative to the transaction itself.

Pay close attention to the sequence number encoding in BIP 68—it's complex but essential for Lightning Network and other Layer 2 protocols.

Next week: SegWit! We'll explore how witness segregation solves transaction malleability and creates the foundation for Taproot's efficiency.

Keep building!

—Aaron

---

## 📌 Key Takeaways

✅ P2SH enables complex scripts hidden behind simple addresses  
✅ Two-phase execution: hash check → stack reset → script execution  
✅ OP_CHECKMULTISIG requires OP_0 workaround for off-by-one bug  
✅ 2-of-3 multisig balances security with operational flexibility  
✅ CSV provides relative time locks (blocks since UTXO creation)  
✅ P2SH limitations: full script reveal, no selective disclosure  
✅ Taproot builds on P2SH but reveals only executed paths

---

**Previous**: [← Lesson 2: Bitcoin Script and P2PKH](./lesson-02-bitcoin-script-p2pkh.md)  
**Next**: [Lesson 4: SegWit Transactions and Witness Structure →](./lesson-04-segwit-witness.md)

---

*This lesson is part of the Taproot Script Engineering Cohort at Chaincode Labs (2026)*