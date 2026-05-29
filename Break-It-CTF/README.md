# 🛡️ TryHackMe Write-up: Break It

**Difficulty:** Medium/Hard (Cryptography focus)  
**Topics:** Advanced Cryptography, Nested Encodings (Matryoshka), Custom Scripting, Automation.

## 🎯 Objective
The "Break It" room is a heavy cryptography challenge designed to test a player's ability to recognize, decode, and decrypt heavily obfuscated strings. The ultimate goal is to uncover hidden flags buried beneath multiple layers of diverse encoding schemes and ciphers.

## 🧩 The Challenge: Matryoshka Cryptography
Unlike standard crypto rooms where a string is encoded once or twice, "Break It" utilizes a "Matryoshka" approach. A single string might undergo a sequence like:
`Array ASCII -> Hex -> Base64 -> ROT47 -> Vigenère (with a dynamically hidden key)`

Attempting to solve this manually using tools like CyberChef quickly becomes inefficient and highly prone to human error, especially when dealing with hidden inline artifacts (e.g., `(key: something)`) or edge cases like Base91 bytearrays and false-positive URL encodings.

## 🚀 The Approach: Automation Over Manual Labor
Instead of manually peeling back the layers of each challenge, I approached this room from a Developer/Security Engineer perspective. I decided to build a custom, heuristic-based Python tool capable of automatically identifying and unpacking the encodings.

### The Cybernetic Autonomous Decoder
I developed a dedicated tool to tackle this specific brand of nested cryptography. The script relies on two main phases:
1. **Deterministic Decoding:** Automatically peels off standard bases (Base16, 32, 58, 64, 85, 91, Ascii85) and standardizes the output.
2. **Cryptographic Brute-Force & Heuristics:** Generates candidates using Caesar, XOR, and Vigenère (using dynamically extracted keys and a 10,000-word dictionary fallback). 

The core logic relies on a **Language Scoring Engine** that mathematically assesses if a decrypted string makes sense in natural English, effectively stopping the loop when the true plaintext or a CTF flag is revealed, while actively mitigating "Reward Hacking" (false positives caused by mathematical coincidences).

🔗 **[View the complete Cybernetic Autonomous Decoder Repository here!](https://github.com/mattiapertusati/Cybernetic-CTF-Decoder)**

## 🏆 Conclusion
By developing a custom automation tool, I successfully bypassed the tedious manual labor usually required for deeply nested crypto challenges. This room was a fantastic exercise not only in cryptography but in **Python scripting, heuristic analysis, and AI-assisted tool development**. The resulting decoder is now a permanent addition to my cybersecurity arsenal.

<img width="1036" height="636" alt="Screenshot 2026-05-29 110553" src="https://github.com/user-attachments/assets/3dda84e0-2c78-4e9c-bc58-a522e7bcfca8" />
