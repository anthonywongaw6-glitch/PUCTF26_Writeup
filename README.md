# Under The Shell — Writeup

## Challenge Info

| Field | Value |
|---|---|
| **Challenge Name** | Under The Shell |
| **Author** | liyanqwq |
| **Category** | Reverse Engineering |
| **Description** | Just cat the flag. But where is it? |

---

## TL;DR

This challenge presents a **fake web terminal** that looks like a restricted Linux shell, but the important logic runs client-side in **WebAssembly (WASM)**.

Trying to solve it purely from inside the shell leads to:

- decoys  
- fake permissions  
- misleading outputs  

The key hint appears when trying to read the flag:

```
cat: flag.txt: Permission denied. Maybe try a deeper inspection?
```

That **"deeper inspection"** refers to **browser developer tools**.

By locating the **WASM validator**, setting a **breakpoint**, and dumping **WASM memory**, the real flag can be extracted.

---

## Final Flag

```
PUCTF26{C4t_c4T_7h3_w45M_6c138b4309f10ce058582896f8d2e581}
```

---

## Solution Overview

The solving process:

1. Enumerate the fake shell environment  
2. Identify decoys and misleading outputs  
3. Pivot to browser developer tools  
4. Locate the WASM-backed flag validator  
5. Pause execution with a breakpoint  
6. Dump and search WASM memory  
7. Extract the real flag  

---

# 1. Initial Enumeration

The challenge drops us into a terminal-like environment called **NuttyShell**.

Running `help` reveals a limited set of commands:

- `cat`
- `ls`
- `whoami`
- `sudo`
- `vim`
- `nvim`
- `b4ckd00r`

A quick directory listing shows some interesting files:

```
ls -la
```

Relevant entries included:

- `flag.txt`
- `.flag.txt`
- `notes.md`
- `payloads/`
- `scans/`
- `secret_stash.zip`

The obvious first attempt:

```
cat flag.txt
```

which returned:

```
cat: flag.txt: Permission denied. Maybe try a deeper inspection?
```

This was the first strong sign that the shell itself was not the whole challenge.

---

# 2. Decoys and Misdirection

Trying the hidden file:

```
cat .flag.txt
```

returned:

```
flag{try_h4rder_h4rd_t0_s0lve}
```

At first glance this looked useful, but it did **not match the provided flag format**:

```
PUCTF26{[a-zA-Z0-9_]+_[a-fA-F0-9]{32}}
```

So this was clearly a **decoy flag**.

---

### Suspicious Command

The custom command `b4ckd00r` looked suspicious. Testing it suggested it was acting as a **validator** instead of a file utility:

```
b4ckd00r flag.txt
b4ckd00r .flag.txt
```

Both returned something like:

```
Incorrect flag. Keep trying.
```

---

### Checking Notes

```
cat notes.md
```

Output:

```
### Lab Notes
1. Audit the Buffer Overflow payload in payloads/
2. Remind the team to stop using 'sudo' as a prank.
3. Secure the credentials.
```

This added flavor and misdirection, but not a direct path to the solution.

---

### Another Observation

Listing `payloads/` and `scans/` appeared to loop back to the same content.

This strongly suggested the shell was a **simulated interface**, not a real filesystem.

---

# 3. Pivoting to Browser Analysis

The phrase:

```
Maybe try a deeper inspection?
```

turned out to be the real hint.

Instead of continuing to poke at the shell, I opened **Developer Tools (F12)** and started checking the page itself.

I looked through:

- **Elements**
- **Console**
- **Application**
  - Local Storage
  - Session Storage
  - Cookies

None of those directly contained the flag.

---

### Breakthrough

In the **Sources tab**, I found several files:

```
wasm_worker.js
wasm_validator.js
wasm_validator_bg.wasm
```

Finding a `.wasm` file made the challenge’s structure much clearer.

The shell behavior was backed by **WebAssembly**, meaning the real flag-checking logic was likely inside the client.

---

# 4. Understanding the Validator

Inside `wasm_validator.js`, the key exported function looked like this:

```javascript
export function verify_flag(flag) {
    const ptr0 = passStringToWasm0(
        flag,
        wasm.__wbindgen_export,
        wasm.__wbindgen_export2
    );
    const len0 = WASM_VECTOR_LEN;
    const ret = wasm.verify_flag(ptr0, len0);
    return ret !== 0;
}
```

This told us that:

- the frontend sends a candidate string into the WASM module
- the WASM module checks it
- the result is returned as `true` or `false`

So `b4ckd00r` was effectively backed by a **client-side flag verifier**.

That is always a big reversing clue:  
if validation happens on the client, the secret must exist somewhere in the client code.

---

# 5. Accessing the WASM Scope

My first instinct was to call things directly in the console:

```javascript
verify_flag("PUCTF26{test_hash}")
```

or inspect memory directly:

```javascript
new TextDecoder().decode(new Uint8Array(wasm.memory.buffer))
```

But both failed because `verify_flag` and `wasm` were **not globally accessible**.

They were scoped inside the module.

---

### Using Breakpoints

To work around that, I used a breakpoint.

Steps:

1. Open `wasm_validator.js` in the **Sources** tab
2. Set a breakpoint inside `verify_flag`
3. Trigger validation from the shell:

```
b4ckd00r test
```

4. Execution pauses inside the function

Now the console has access to the current scope, including the `wasm` object.

This is a useful browser reversing technique:

> If a variable is not global, pause execution where it exists and inspect it there.

---

# 6. Dumping WASM Memory

With execution paused and `wasm` accessible, I dumped the module’s memory:

```javascript
new TextDecoder().decode(new Uint8Array(wasm.memory.buffer))
```

This returned a huge blob of mostly unreadable data.

Instead of inspecting it manually, I stored it and searched with a regex:

```javascript
const memory = new TextDecoder().decode(
    new Uint8Array(wasm.memory.buffer)
);

console.log(memory.match(/PUCTF26\{.*?\}/g));
```

Output:

```
["PUCTF26{C4t_c4T_7h3_w45M_6c138b4309f10ce058582896f8d2e581}"]
```

And that was the real flag.

---

# Why This Worked

The challenge relied on **client-side validation**.

Even though the shell UI tried to hide the answer behind:

- fake permissions
- decoy files
- looping directories

the browser still needed the **real verification logic** somewhere in memory.

Since the WASM module had to know what a valid flag looked like, the correct flag string was recoverable from **WASM linear memory** during execution.

In short:

- the shell was fake
- validation was client-side
- the browser had everything needed to recover the flag
- developer tools were the intended path

---

# Key Takeaways

- Not every shell challenge is really a shell challenge
- Client-side validation is reversible
- WebAssembly should be treated like any other binary
- Browser developer tools are essential for modern web CTFs
- Breakpoints allow access to variables not exposed globally
- Memory dumping + regex searching often reveals secrets

---

# Commands / Snippets Used

### Shell Interaction

```
ls -la
cat flag.txt
cat .flag.txt
cat notes.md
b4ckd00r flag.txt
b4ckd00r .flag.txt
```

### WASM Memory Extraction

```javascript
const memory = new TextDecoder().decode(
    new Uint8Array(wasm.memory.buffer)
);

console.log(memory.match(/PUCTF26\{.*?\}/g));
```

---

# Closing Thoughts

The fake terminal, permission errors, decoy flag, and looping directories were all designed to keep the player thinking like a shell user.

The real solution path only becomes obvious once you stop trusting the terminal and start inspecting the application behind it.

The hint **“deeper inspection”** was accurate in the best possible way:  
the answer was not under the shell, but **behind it**, somehow.
