# Recipe — Writeup

## Challenge Info

| Field | Value |
|---|---|
| **Challenge Name** | Recipe |
| **Author** | Idkcallwutname |
| **Category** | Misc |
| **Description** | Can you find out what Recipe to make this thing? |

---

## TL;DR

The provided file `dishes` contains a **gzip-compressed payload**.

After decompressing it, the file reveals a **custom substitution cipher alphabet embedded at the beginning of the file**. The rest of the file is encoded using that substitution mapping.

By reversing the substitution and decoding the resulting **ASCII85 encoded payload**, the hidden message can be recovered, revealing the flag.

---

## Final Flag

```
PUCTF26{y0u_4re_m4st3r_ln_r3v3r53_b453_me554g3_cb6a739ace061277c5ec70e7abfc7c36}
```

---

## Solution Overview

The solving process:

1. Inspect the provided file
2. Identify the gzip compression layer
3. Analyze the embedded character mapping
4. Reverse the substitution cipher
5. Detect ASCII85 encoding
6. Decode the final payload
7. Recover the flag

---

# 1. Initial Inspection

The challenge provides a single file:

```
dishes
```

First step is to determine what kind of file it is.

```
file dishes
```

Output:

```
dishes: gzip compressed data
```

This tells us that the file is **compressed using gzip**, meaning the first step is simply decompression.

---

# 2. Decompressing the File

We decompress the file using `gunzip`:

```
gunzip -c dishes > stage1
```

Now inspect the contents:

```
head stage1
```

The beginning of the file looks like this:

```
ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789!@#$%^&*()-_=+[]{}|;:',.<>?/`~cipherA:C;D>`Ja?&mMl/9sw.yP-~$ER1H*VI8d<=Fq^g@'}ZYtLv#G,]W+%{ob4Q_(3fT!N[ux)UBS|kK0z62OX75nj
?>cCJlewl0~_u]X...
```

The first line clearly does **not look like encoded data**.

Instead, it appears to be a **character mapping table**.

---

# 3. Understanding the Character Table

Looking closely at the first line, it contains **184 characters**.

This splits cleanly into:

```
92 characters
+
92 characters
```

This strongly suggests the presence of a **substitution cipher mapping**.

Structure:

```
[ original character set ][ encoded character set ]
```

Meaning each encoded character can be mapped back to its original value.

```
encoded_char → original_char
```

So the rest of the file is simply the payload encoded using this substitution.

---

# 4. Reversing the Substitution Cipher

To decode the payload, we construct a translation table using Python.

```
data = open("stage1","rb").read()

alphabet = data.split(b"\n",1)[0]
payload  = data.split(b"\n",1)[1]

orig = alphabet[:92]
enc  = alphabet[92:]

table = {enc[i]:orig[i] for i in range(92)}

decoded = bytes(table.get(b,b) for b in payload)

open("stage2","wb").write(decoded)
```

After applying the substitution reversal, inspecting the output reveals:

```
PLAINTEXT:c$_4>>9#DZ8kYIL?!tlCQHqLUA)...
```

This indicates that the substitution layer has been successfully removed.

---

# 5. Identifying the Next Encoding

The decoded content begins with:

```
PLAINTEXT:
```

The rest of the characters fall within the printable range typical of **ASCII85 encoding**.

ASCII85 is commonly used for compact text-based binary encoding.

So the next step is to remove the prefix and decode the ASCII85 payload.

---

# 6. Decoding ASCII85

Using Python:

```
import base64

encoded = decoded.split(b":",1)[1]

stage3 = base64.a85decode(encoded)

open("stage3","wb").write(stage3)
```

This produces the final decoded output.

Searching the output reveals the flag.

---

# Final Flag

```
PUCTF26{y0u_4re_m4st3r_ln_r3v3r53_b453_me554g3_cb6a739ace061277c5ec70e7abfc7c36}
```

---

# Why This Worked

The challenge relied on **layered encoding rather than encryption**.

The steps were:

- gzip compression
- substitution cipher
- ASCII85 encoding

The substitution alphabet was conveniently embedded in the file itself, making the decoding process straightforward once identified.

---

# Key Takeaways

- Always inspect files using `file`
- Compression layers are common in CTF challenges
- Character tables often indicate substitution ciphers
- ASCII85 is less common but recognizable by its printable character range
- Layered encoding challenges require peeling layers one by one

---

# Commands / Snippets Used

### File Inspection

```
file dishes
```

### Decompression

```
gunzip -c dishes > stage1
```

### Substitution Reversal

```
table = {enc[i]:orig[i] for i in range(92)}
decoded = bytes(table.get(b,b) for b in payload)
```

### ASCII85 Decoding

```
import base64
base64.a85decode(data)
```

---

# Closing Thoughts

This challenge demonstrates a classic CTF pattern: **layered encoding combined with a substitution cipher**.

Rather than hiding the flag behind complex cryptography, the challenge relies on the solver recognizing structural hints in the file and carefully peeling each layer.

Once the embedded substitution alphabet is recognized, the rest of the decoding process becomes straightforward.
