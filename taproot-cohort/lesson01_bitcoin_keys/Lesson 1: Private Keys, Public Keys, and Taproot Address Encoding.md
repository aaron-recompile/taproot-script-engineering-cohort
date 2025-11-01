# Lesson 1: Private Keys, Public Keys, and Taproot Address Encoding

**Week**: 1  
**Duration**: Self-paced (estimated 3-4 hours)  
**Prerequisites**: Basic understanding of cryptography and Bitcoin transactions

---

## 🎯 Learning Objectives

By the end of this lesson, you will be able to:

1. Generate Bitcoin private keys and understand their cryptographic properties
2. Derive public keys from private keys using elliptic curve cryptography
3. Create addresses in multiple formats (Legacy, SegWit, Taproot)
4. Understand the relationship between WIF, private keys, and address encoding
5. Explain the significance of x-only public keys in Taproot
6. Construct and decode Bech32m-encoded Taproot addresses

---

## 🧩 Key Concepts

### The Cryptographic Hierarchy

Bitcoin's security model relies on a **one-way** mathematical relationship:

```
Private Key (256-bit) → Public Key (ECDSA point) → Address (encoded hash)
```

**Critical Security Properties:**
- **Forward derivation**: Easy (private key → public key → address)
- **Reverse derivation**: Computationally infeasible (cannot go backward)
- **Collision resistance**: Extremely unlikely for different keys to produce same address

### Private Keys: The Foundation of Ownership

A Bitcoin private key is a **256-bit random number**:
- **Key space**: 2^256 possible keys (larger than atoms in the universe)
- **Entropy requirement**: Must be truly random, not predictable
- **Representation formats**: 
  - Hexadecimal (64 chars, e.g., `e9873d...`)
  - WIF - Wallet Import Format (Base58Check encoded)

**WIF Encoding Process:**
```
1. Add version prefix (0x80 mainnet, 0xEF testnet)
2. Optional: Add 0x01 for compressed public keys
3. Calculate checksum: SHA256(SHA256(data)), take first 4 bytes
4. Apply Base58 encoding
```

**WIF Prefixes:**
- `L` or `K`: Mainnet private keys
- `c`: Testnet private keys

### Public Keys: Cryptographic Verification Points

Public keys are **points on the secp256k1 elliptic curve**:

**Curve equation**: `y² = x³ + 7`

**Format Options:**

| Format | Size | Structure | Use Case |
|--------|------|-----------|----------|
| **Uncompressed** | 65 bytes | `04 + x (32) + y (32)` | Legacy, rarely used |
| **Compressed** | 33 bytes | `02/03 + x (32)` | Modern standard |
| **X-only (Taproot)** | 32 bytes | `x coordinate only` | Taproot exclusive |

**Compression mechanism:**
- The curve's mathematical properties allow reconstructing y from x
- Prefix `02`: y is even
- Prefix `03`: y is odd

**Taproot's X-only Innovation:**
- Uses only the x-coordinate (32 bytes)
- Reduces transaction size
- Enables key aggregation and Schnorr signatures
- Simplifies signature verification

### Address Generation: User-Friendly Payment Destinations

Addresses are **NOT** public keys—they are **encoded hashes** of public keys.

**Why use addresses?**
- **Privacy**: Don't directly reveal public keys until spent
- **Quantum resistance**: Hash functions provide post-quantum security layer
- **Error detection**: Built-in checksums prevent typos

**Address Generation Pipeline:**
```
1. Hash public key: SHA256(pubkey) → RIPEMD160 = Hash160
2. Add metadata: Version bytes + script type info
3. Add checksum: Error detection
4. Encode: Base58Check or Bech32(m)
```

### Address Types Comparison

| Type | Encoding | Prefix | Example | Primary Use |
|------|----------|--------|---------|-------------|
| **P2PKH** | Base58Check | `1...` | `1A1zP1eP5Q...` | Legacy payments |
| **P2SH** | Base58Check | `3...` | `3J98t1WpEZ...` | Multisig, wrapped SegWit |
| **P2WPKH** | Bech32 | `bc1q...` | `bc1qw508d6q...` | Native SegWit |
| **P2TR** | Bech32m | `bc1p...` | `bc1plz0h3rl...` | Taproot |

**Important Insight:**
> From the node's perspective, Bitcoin never stores addresses—only scripts (scriptPubKey). Addresses are a human-friendly representation of locking scripts. Once you recognize the prefix, you know the script type behind it.

---

## 💻 Example Code

### Setup: Install Required Libraries

```bash
# Install the required Python packages
pip install python-bitcoinlib bitcoinutils ecdsa
```

### Example 1: Generate Keys and Addresses

```python
from bitcoinlib.keys import Key

# Generate a new Bitcoin key pair
key = Key()

# Extract private key in different formats
private_key_hex = key.private_hex      # 32 bytes in hex
private_key_wif = key.wif()            # Wallet Import Format

print("="*60)
print("PRIVATE KEY FORMATS")
print("="*60)
print(f"Hex: {private_key_hex}")
print(f"WIF: {private_key_wif}")
print()

# Generate public keys in different formats
public_key_compressed = key.public_hex                    # 33 bytes
public_key_uncompressed = key.public_uncompressed_hex     # 65 bytes

print("="*60)
print("PUBLIC KEY FORMATS")
print("="*60)
print(f"Compressed (33 bytes):   {public_key_compressed}")
print(f"Uncompressed (65 bytes): {public_key_uncompressed[:70]}...")
print()

# Taproot x-only public key (32 bytes)
taproot_pubkey = key.public_hex[2:]  # Remove 02/03 prefix

print("="*60)
print("TAPROOT X-ONLY PUBLIC KEY")
print("="*60)
print(f"X-only (32 bytes): {taproot_pubkey}")
print()

# Generate different address types from the same key
legacy_address = key.address()                                     # P2PKH
segwit_native = key.address(encoding='bech32')                     # P2WPKH
segwit_p2sh = key.address(encoding='base58', script_type='p2sh')   # P2SH-P2WPKH
taproot_address = key.address(script_type='p2tr')                  # P2TR

print("="*60)
print("ADDRESS FORMATS (ALL FROM SAME PRIVATE KEY)")
print("="*60)
print(f"Legacy (P2PKH):    {legacy_address}")
print(f"SegWit Native:     {segwit_native}")
print(f"SegWit P2SH:       {segwit_p2sh}")
print(f"Taproot:           {taproot_address}")
print()
```

**Expected Output Structure:**
```
============================================================
PRIVATE KEY FORMATS
============================================================
Hex: e9873d79c6d87dc0fb6a5778633389dfa5c32fa27f99b5199abf2f9848ee0289
WIF: L1aW4aubDFB7yfras2S1mN3bqg9w3KmCPSM3Qh4rQG9E1e84n5Bd

============================================================
PUBLIC KEY FORMATS
============================================================
Compressed (33 bytes):   0250be5fc44ec580c387bf45df275aaa8b27e2d7716af31f10eeed357d126bb4d3
Uncompressed (65 bytes): 0450be5fc44ec580c387bf45df275aaa8b27e2d7716af31f10eeed357d...

============================================================
TAPROOT X-ONLY PUBLIC KEY
============================================================
X-only (32 bytes): 50be5fc44ec580c387bf45df275aaa8b27e2d7716af31f10eeed357d126bb4d3

============================================================
ADDRESS FORMATS (ALL FROM SAME PRIVATE KEY)
============================================================
Legacy (P2PKH):    1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa
SegWit Native:     bc1qw508d6qejxtdg4y5r3zarvary0c5xw7kygt080
SegWit P2SH:       3J98t1WpEZ73CNmQviecrnyiWrnqRhWNLy
Taproot:           bc1plz0h3rlj2zvn88pgywqtr9k3df3p75p3ltuxh0
```

### Example 2: Import Existing Key and Verify Derivation

```python
from bitcoinlib.keys import Key

# Import a testnet private key (this is for educational purposes only)
testnet_wif = 'cPeon9fBsW2BxwJTALj3hGzh9vm8C52Uqsce7MzXGS1iFJkPF4AT'

# Create key object from WIF
key = Key(import_key=testnet_wif, network='testnet')

print("="*60)
print("IMPORTED TESTNET KEY VERIFICATION")
print("="*60)
print(f"Network: {key.network.name}")
print(f"Private Key (Hex): {key.private_hex}")
print(f"Private Key (WIF): {key.wif()}")
print(f"Public Key: {key.public_hex}")
print(f"Legacy Address: {key.address()}")
print(f"Taproot Address: {key.address(script_type='p2tr')}")
```

### Example 3: Understanding Address Prefix Recognition

```python
def identify_address_type(address):
    """
    Identify Bitcoin address type from its prefix
    """
    if address.startswith('1'):
        return "P2PKH (Legacy Pay-to-Public-Key-Hash)"
    elif address.startswith('3'):
        return "P2SH (Pay-to-Script-Hash)"
    elif address.startswith('bc1q'):
        return "P2WPKH (Native SegWit)"
    elif address.startswith('bc1p'):
        return "P2TR (Taproot)"
    elif address.startswith('tb1q'):
        return "P2WPKH (Testnet Native SegWit)"
    elif address.startswith('tb1p'):
        return "P2TR (Testnet Taproot)"
    else:
        return "Unknown or non-standard format"

# Test the function
addresses = [
    "1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa",
    "3J98t1WpEZ73CNmQviecrnyiWrnqRhWNLy",
    "bc1qw508d6qejxtdg4y5r3zarvary0c5xw7kygt080",
    "bc1plz0h3rlj2zvn88pgywqtr9k3df3p75p3ltuxh0",
]

print("="*60)
print("ADDRESS TYPE IDENTIFICATION")
print("="*60)
for addr in addresses:
    addr_type = identify_address_type(addr)
    print(f"{addr[:20]}... → {addr_type}")
```

---

## 🧪 Practice Task

### Task 1: Generate Your First Testnet Taproot Address

**Objective**: Create a complete key derivation pipeline and test on Bitcoin testnet.

**Steps:**

1. **Generate a new key pair** using the code above
2. **Extract all formats**:
   - Private key (hex and WIF)
   - Public key (compressed and x-only)
   - All address types
3. **Save your Taproot testnet address** for future lessons
4. **Get testnet coins**: Visit https://coinfaucet.eu/en/btc-testnet/ and send some coins to your address
5. **Verify reception**: Check your address on a testnet explorer (e.g., https://mempool.space/testnet)

**Deliverable**: 
- Take a screenshot of your address receiving testnet coins
- Save your private key WIF in a secure note (you'll need it for future lessons)

### Task 2: Address Derivation Verification

**Challenge**: Given this private key (testnet only, educational purposes):
```
cPeon9fBsW2BxwJTALj3hGzh9vm8C52Uqsce7MzXGS1iFJkPF4AT
```

Write code to:
1. Import the private key
2. Derive the public key
3. Generate a Taproot address
4. Verify the address starts with `tb1p` (testnet Taproot prefix)

**Expected Result**:
Your code should produce a testnet Taproot address. Verify it matches the one you derive manually.

### Task 3: Encoding Detective Work

**Question**: Why does Taproot use Bech32m instead of Bech32?

Research BIP 350 and write a 2-3 sentence explanation of the specific flaw in Bech32 that Bech32m fixes.

---

## 📘 Reading

### Required Reading
- **Mastering Taproot** - Chapter 1: Private Keys, Public Keys, and Address Encoding (pages 1-11)

### Supplementary Resources
- **BIP 340**: Schnorr Signatures for secp256k1 - https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki
- **BIP 341**: Taproot: SegWit version 1 spending rules - https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki
- **BIP 350**: Bech32m format for v1+ witness addresses - https://github.com/bitcoin/bips/blob/master/bip-0350.mediawiki
- **Bitcoin Optech**: Taproot Workshop - https://bitcoinops.org/en/schorr-taproot-workshop/

### Recommended Deep Dives
- Elliptic Curve Cryptography explained visually: https://www.youtube.com/watch?v=NF1pwjL9-DE
- Base58Check encoding specification
- Testnet faucet and explorer tools

---

## 💬 Discussion Prompts

Post your thoughts in the Discord #week-1-discussion channel:

### Question 1: Privacy Implications
**"How does the x-only public key format in Taproot improve privacy compared to compressed public keys in SegWit? What information is eliminated, and why does this matter for Taproot's key aggregation features?"**

*Hint: Think about what information the 02/03 prefix reveals and how Schnorr signatures handle this differently.*

### Question 2: Quantum Resistance
**"Bitcoin addresses use Hash160 (SHA256 + RIPEMD160) to hash public keys before encoding them as addresses. How does this provide a layer of post-quantum protection? What happens to this protection once you spend from an address?"**

*Hint: Consider what information is revealed on-chain before vs. after the first spend.*

### Question 3: Encoding Trade-offs
**"Taproot addresses are longer than SegWit addresses (58-62 chars vs 42-46 chars). Given that Taproot is supposed to be more efficient, why did we accept this size increase? What benefits does Bech32m encoding provide that justify the extra length?"**

---

## 🎙️ Instructor's Note

Welcome to the Taproot Script Engineering Cohort! This first lesson establishes the cryptographic foundation we'll build upon throughout the course.

You might feel overwhelmed by the different formats and encodings—that's normal. The key insight to take away is this: **addresses are for humans, scripts are for nodes**. Everything we're learning about key formats and address encoding is preparation for understanding how Taproot enables multiple spending conditions to hide behind a single, uniform address format.

By the end of this cohort, you'll be comfortable constructing multi-leaf Taproot trees where each leaf represents a different spending path—all while maintaining the privacy of unused paths. But first, we need to master the basics.

Take your time with the practice tasks, especially getting testnet coins. You'll need these for hands-on transaction building in upcoming lessons.

See you in Discord for Q&A!

—Aaron

---

## 📌 Key Takeaways

✅ Private keys (256-bit) are the ultimate source of ownership  
✅ Public keys are derived via elliptic curve multiplication (one-way function)  
✅ X-only public keys (32 bytes) are Taproot's innovation for efficiency  
✅ Addresses are encoded hashes of public keys, not the keys themselves  
✅ Bech32m encoding (bc1p...) is the standard for Taproot addresses  
✅ Understanding the derivation hierarchy is critical for Taproot development

---

**Next Lesson**: [Lesson 2: Stack Operations and P2PKH Execution →](./lesson-02-p2pkh-stack-execution.md)

---

*This lesson is part of the Taproot Script Engineering Cohort at Chaincode Labs (2026)*