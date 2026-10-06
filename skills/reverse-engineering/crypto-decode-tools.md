# Crypto / codec tool cheat sheet

> RE and CTF often hit encrypted/encoded/hashed data. Practical tools by scene.

---

## Auto-ID + decrypt (unknown cipher)

| Tool | Stars | Use | Link |
|------|-------|------|------|
| **Ciphey** | 18k+ | AI auto-ID and decrypt (50+ encodings/ciphers/hashes) | https://github.com/Ciphey/Ciphey |
| **CyberChef** | 29k+ | Online/offline codec Swiss army knife (drag-drop) | https://github.com/gchq/CyberChef |
| **dcode.fr** | — | Online 900+ cipher/encoding/math tools | https://www.dcode.fr/ |

### Ciphey usage

```bash
pip install ciphey
# auto-detect and decrypt
ciphey -t "ciphertext"
# from file
ciphey -f encrypted.txt
```

Ciphey supports: Base64/32/16, Caesar, Vigenere, XOR, AES (weak keys), Morse, Binary, Hex, URL encoding, HTML entities, hash ID, etc.

### CyberChef usage

```text
Online: https://gchq.github.io/CyberChef/
Offline: download GitHub Release HTML and open it

Common recipes:
- From Base64 → decode Base64
- XOR → XOR decrypt (can brute key)
- AES Decrypt → AES decrypt
- Magic → auto-detect encoding
```

---

## Hash ID and crack

| Tool | Use | Link |
|------|------|------|
| **hashID** | ID hash type (MD5/SHA/bcrypt etc.) | https://github.com/psypanda/hashID |
| **hash-identifier** | same, Python | https://github.com/blackploit/hash-identifier |
| **haiti** | modern hash ID (more accurate) | `gem install haiti` |
| **Hashcat** | GPU hash crack | https://hashcat.net/ |
| **John the Ripper** | CPU hash crack | https://www.openwall.com/john/ |
| **hashes.com** | online hash lookup (rainbow) | https://hashes.com/ |

```bash
# ID hash type
hashid '5f4dcc3b5aa765d61d8327deb882cf99'
# output: [+] MD5

# haiti (more accurate)
haiti '5f4dcc3b5aa765d61d8327deb882cf99'

# Hashcat crack
hashcat -m 0 hash.txt rockyou.txt  # MD5
hashcat -m 1000 hash.txt rockyou.txt  # NTLM
```

---

## RSA attacks

| Tool | Use | Link |
|------|------|------|
| **RsaCtfTool** | auto RSA attacks (20+ methods) | https://github.com/Ganapati/RsaCtfTool |
| **SageMath** | math (factoring / elliptic curves) | https://www.sagemath.org/ |
| **factordb.com** | online factor lookup | http://factordb.com/ |
| **yafu** | local factoring | https://github.com/bbuhrow/yafu |

```bash
# RsaCtfTool auto-attack
python RsaCtfTool.py --publickey pub.pem --private
python RsaCtfTool.py --publickey pub.pem --uncipherfile cipher.txt

# supported attacks:
# Wiener, Boneh-Durfee, Fermat, Pollard p-1, Williams p+1
# Common modulus, Small q, Hastads, Noveltyprimes, etc.
```

---

## XOR analysis

| Tool | Use | Link |
|------|------|------|
| **xortool** | XOR key-length guess + known-plaintext | https://github.com/hellman/xortool |
| **CyberChef XOR** | visual XOR | built into CyberChef |

```bash
# guess XOR key length
xortool encrypted_file
# decrypt with guessed key length
xortool -l 4 -c 00 encrypted_file

# known-plaintext (partial plaintext known)
xortool-xor -f encrypted -s "known_plaintext"
```

---

## Classical ciphers

| Cipher | Tool | Notes |
|---------|------|------|
| Caesar | CyberChef / dcode.fr | brute 25 shifts |
| Vigenere | dcode.fr / Ciphey | need guessed key length |
| Substitution | quipqiup.com | frequency analysis auto-solve |
| Enigma | dcode.fr | online simulator |
| Rail Fence | dcode.fr / CyberChef | rail-fence cipher |
| Playfair | dcode.fr | needs key |
| Morse | CyberChef | dots/dashes to text |
| Bacon | dcode.fr | binary stego |
| ROT13/47 | CyberChef / `tr` | simple substitution |

---

## Encoding ID and convert

| Encoding | ID features | Decode |
|------|---------|---------|
| Base64 | trailing `=` or `==`, charset A-Za-z0-9+/ | `base64 -d` / CyberChef |
| Base32 | uppercase + 2-7, trailing `=` | CyberChef |
| Base58 | no 0/O/I/l; common in short IDs | CyberChef |
| Hex | only 0-9a-f, even length | `xxd -r -p` / CyberChef |
| URL encoding | `%XX` | `urldecode` / CyberChef |
| HTML entities | `&#XX;` or `&amp;` | CyberChef |
| Unicode escape | `\uXXXX` | Python `decode('unicode_escape')` |
| JWT | `xxxxx.yyyyy.zzzzz` (three Base64URL parts) | jwt.io / CyberChef |
| Brainfuck | only `><+-.,[]` | online interpreter |
| Ook! | only `Ook.` `Ook!` `Ook?` | online interpreter |

---

## Crypto ID during RE

### ID algorithm by constants

| Constant/feature | Algorithm |
|-----------|------|
| `0x67452301, 0xEFCDAB89, 0x98BADCFE, 0x10325476` | MD5 |
| `0x6A09E667, 0xBB67AE85, 0x3C6EF372` | SHA-256 |
| `0x63, 0x7C, 0x77, 0x7B` (S-Box start) | AES |
| `0x243F6A88` (hex of π) | Blowfish |
| `0xB7E15163, 0x9E3779B9` | RC5/RC6/TEA |
| `0x61707865` ("expa") | ChaCha20/Salsa20 |
| `0xC6EF3720` | XTEA |

### ID by behavior

| Behavior | Likely algorithm |
|---------|-----------|
| 256-byte LUT + swap | RC4 |
| 16-byte blocks + multi-round permute | AES |
| Feistel (left/right swap) | DES/Blowfish/TEA |
| big-int mul / modexp | RSA |
| elliptic-curve point ops | ECDSA/ECDH |
| fixed 64-round loop | TEA/XTEA |
| 32 rounds + delta constant | XTEA |

---

## Automated cryptanalysis

| Tool | Use | Link |
|------|------|------|
| **FeatherDuster** | automated cryptanalysis framework | https://github.com/nccgroup/featherduster |
| **PkCrack** | ZIP known-plaintext | https://www.unix-ag.uni-kl.de/~conrad/krypto/pkcrack.html |
| **bkcrack** | ZIP known-plaintext (modern) | https://github.com/kimci86/bkcrack |
| **z3** | SMT solver (constraints) | https://github.com/Z3Prover/z3 |
| **angr** | symbolic execution (auto-solve input) | https://angr.io/ |

---

## Fast decision tree

```text
Got unknown data:

1. Look at length and charset
   - only hex chars → maybe hex encoding or hash
   - trailing = → Base64
   - three dot-separated parts → JWT
   - 32/40/64 hex chars → hash (MD5/SHA1/SHA256)

2. Try Ciphey auto
   ciphey -t "data"

3. If Ciphey fails → CyberChef Magic mode

4. If hash → hashID type → Hashcat/John crack

5. If RSA → RsaCtfTool auto-attack

6. If XOR → xortool analyze key

7. If classic ZIP crypto → prefer `bkcrack` known-plaintext; do not start evidence-free password brute

8. If custom crypto → IDA/Ghidra reverse the algorithm → hand-write decrypt script
```

---

## Online resources

| Resource | Link | Use |
|------|------|------|
| CyberChef | https://gchq.github.io/CyberChef/ | universal codec |
| dcode.fr | https://www.dcode.fr/ | 900+ cipher tools |
| quipqiup | https://quipqiup.com/ | substitution auto-solve |
| factordb | http://factordb.com/ | RSA factoring |
| jwt.io | https://jwt.io/ | JWT decode/verify |
| hashes.com | https://hashes.com/ | hash lookup |
| crackstation | https://crackstation.net/ | online hash crack |
