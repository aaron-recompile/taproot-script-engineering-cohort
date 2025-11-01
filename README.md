# Taproot Script Engineering Cohort - Course Materials

**Instructor**: Aaron Zhang  
**Program**: Chaincode Labs Global Residency Program 2026  
**Course**: Taproot Script Engineering Cohort

---

## 📦 What's Included

Welcome to the **Taproot Script Engineering Cohort**! This course contains the following structured learning materials:

### 1. **Lesson 1**: Private Keys, Public Keys, and Taproot Address Encoding
- Private keys, public keys, and address encoding
- X-only public keys for Taproot
- Bech32m encoding format
- Complete Python code examples
- Testnet practice tasks
- Available in English and Chinese

### 2. **Lesson 2**: Stack Operations and P2PKH Execution
- UTXO model deep dive
- Stack-based execution visualization
- P2PKH transaction construction
- Step-by-step script execution tracing
- Real testnet transaction examples
- Available in English and Chinese

### 3. **Lesson 3**: P2SH Multi-signature and Time Locks
- P2SH architecture and two-phase execution model
- Building 2-of-3 multisig escrow contracts
- CSV (CheckSequenceVerify) timelocks
- Stack execution tracing with stack reset mechanism
- Available in English and Chinese

### 4. **Lesson 4**: SegWit Transactions and Witness Structure
- Transaction malleability and SegWit solution
- Witness segregation concept
- P2WPKH (native SegWit) transaction construction
- Transaction weight units and fee optimization
- Foundation for Taproot
- Available in English and Chinese

### 5. **Lesson 5**: Taproot Introduction - Schnorr Signatures and Key Tweaking
- Schnorr signature linearity and key aggregation
- Key tweaking formula: P' = P + t × G
- X-only public keys (32 bytes)
- Simple key-path Taproot transactions
- Payment unifiability and privacy
- Available in English and Chinese

### 6. **Lesson 6**: Single-Leaf Taproot Script Tree
- Commit-Reveal pattern for Taproot contracts
- Single-leaf script trees with hash locks
- Control blocks (33 bytes for single-leaf)
- Dual-path spending: key-path vs script-path
- Privacy and efficiency comparison

### 7. **Lesson 7**: Dual-Leaf Taproot Script Tree
- Dual-leaf Merkle tree structure
- TapLeaf and TapBranch hash calculation
- Control blocks with Merkle proofs (65 bytes)
- Lexicographic sorting for Merkle root
- Multiple spending paths from same address
- Available in English and Chinese

### 8. **Lesson 8**: Four-Leaf Taproot Script Tree
- Balanced four-leaf Merkle trees (two-level structure)
- OP_CHECKSIGADD for efficient multisig (Tapscript innovation)
- Extended control blocks (97 bytes) with multi-level Merkle proofs
- Combining CSV timelocks with script paths
- Five different spending paths comparison
- Available in English and Chinese

---

## 🎯 Course Content Structure

Each lesson follows a consistent, student-friendly format:

```
📋 Learning Objectives (clear, measurable goals)
🧩 Key Concepts (core technical knowledge)
💻 Example Code (working Python examples)
🧪 Practice Tasks (hands-on testnet exercises)
📘 Reading Materials (Mastering Taproot book + BIPs + other resources)
💬 Discussion Prompts (for Discord engagement)
🎙️ Instructor's Notes (Aaron's teaching insights)
📌 Key Takeaways (summary checklist)
```

---

## 🚀 How to Use These Materials

### Getting Started:
- **All 8 lessons are available**: Complete course materials from fundamentals to advanced Taproot patterns
- **Self-paced learning**: Study at your own pace with estimated durations per lesson (4-7 hours)
- **Prerequisites**: Each lesson builds on previous ones - start with Lesson 1 and progress sequentially
- **Bilingual support**: Most lessons available in both English and Chinese (indicated in lesson listings)

### Course Structure:
- **Lessons 1-2**: Bitcoin fundamentals (Keys, UTXO, P2PKH)
- **Lessons 3-4**: Advanced scripting (P2SH, SegWit)
- **Lessons 5-8**: Taproot mastery (Schnorr, Script Trees, Enterprise patterns)
- Each lesson includes: objectives, concepts, code examples, practice tasks, and reading materials

### Learning Tips:
- **Complete lessons in order**: Each lesson builds on previous concepts
- **Practice on testnet**: All code examples use Bitcoin testnet - safe to experiment
- **Use both versions**: Read English version for technical terms, Chinese version for deeper understanding
- **Follow along with code**: Copy and run the Python examples to see concepts in action
- **Reference the book**: Use *Mastering Taproot* book chapters listed for each lesson for deeper dives
- **Join discussions**: Use Discord discussion prompts to engage with the community
- **Ask for help**: Office hours and Discord are available for questions and clarifications

---

## 📖 Integration with Mastering Taproot Book

These course materials are designed to **complement** Aaron Zhang's *Mastering Taproot* book:

| Lesson | Book Chapter | Focus |
|--------|--------------|-------|
| Lesson 1 | Chapter 1 | Keys, addresses, foundations |
| Lesson 2 | Chapter 2 | Bitcoin Script, P2PKH |
| Lesson 3 | Chapter 3 | P2SH, multisig, time locks |
| Lesson 4 | Chapter 4 | SegWit structure |
| Lesson 5 | Chapter 5 | Taproot introduction |
| Lesson 6 | Chapter 6 | Single-leaf Taproot |
| Lesson 7 | Chapter 7 | Dual-leaf Taproot |
| Lesson 8 | Chapter 8 | Four-leaf enterprise patterns |

**Book = Deep Technical Reference**  
**Lessons = Practical Implementation Guide**

---

## 🛠️ Tools Integration

The following tools will be naturally introduced throughout the course:

- **btcaaron**: Python toolkit used in examples
- **StackFlow**: For script visualization
- **RootScope**: Introduced in Taproot lessons
- **Testnet**: All examples use live testnet

Through hands-on practice, you will naturally master this tool ecosystem.

---

## ✅ Course Content Quality Guarantee

Each lesson includes:

- ✅ Clear learning objectives (3-6 per lesson)
- ✅ Visual stack execution diagrams (ASCII art)
- ✅ Working code examples (copy-paste ready)
- ✅ Testnet practice tasks (progressive difficulty)
- ✅ Discussion prompts (open-ended technical discussions)
- ✅ Reading lists (book chapters + BIPs + articles)
- ✅ Key takeaways (summary points)
- ✅ Instructor notes (Aaron's personal teaching insights)

---

## 📝 Course Schedule

### Current Phase:
1. ✅ Lessons 1-8 content has been released
2. ✅ All code examples have been verified on testnet
3. ✅ Both English and Chinese versions available for most lessons
4. ✅ Ready to start learning

### During the Course:
1. **Every Monday Morning**: New lesson content is released
2. **Discord**: Ask questions and discuss anytime
3. **Office Hours**: Regular review of practice tasks
4. **Practice Task Submissions**: Submit your assignments as required

### Lesson Status:
- ✅ **Lessons 1-5**: Available in both English and Chinese
- ✅ **Lessons 6-8**: Available in English (Chinese versions for 7-8)
- 📚 Each lesson corresponds to the relevant chapter of the *Mastering Taproot* book
- 📈 Difficulty and depth increase progressively from basics to advanced patterns

---

## 🎓 Learning Outcomes

After completing all lessons, you will be able to:

✅ Generate and manage Bitcoin keys (all formats)  
✅ Construct transactions (Legacy, SegWit, Taproot)  
✅ Trace script execution step-by-step  
✅ Build multi-signature contracts  
✅ Implement time-locked spending conditions  
✅ Construct multi-leaf Taproot trees  
✅ Debug Taproot transactions using course tools  
✅ Deploy production-ready Taproot applications

---

## 📞 Getting Help

If you have questions:
- **Course Content Questions**: Ask in Discord
- **Code Example Questions**: Check code comments or ask the community
- **Practice Task Difficulties**: Attend Office Hours for help
- **Concept Understanding Questions**: Refer to relevant chapters of the *Mastering Taproot* book

We're here to support you!

---

## 🙏 Acknowledgments

This course material is based on:
- Aaron Zhang's **Mastering Taproot** book
- **BIPs**: 340 (Schnorr), 341 (Taproot), 342 (Tapscript)
- **Chaincode Labs** residency program structure
- **Bitcoin Optech** technical resources

---

## 📄 License

This course material was prepared by Aaron Zhang for the Chaincode Labs Global Residency Program 2026.

After the course completes, these materials may be open-sourced to benefit the broader Bitcoin developer community.

---

**Ready to become the next generation of Bitcoin developers!** 🚀

If you have any questions, feel free to ask in Discord or refer to the *Mastering Taproot* book for deeper learning.

Happy learning!

