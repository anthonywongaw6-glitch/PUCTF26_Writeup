
	# Under The Shell
	
	> Reverse engineering a fake shell backed by **WebAssembly (WASM)**
	
	## Challenge Info
	
	| Field | Value |
	| --- | --- |
	| **Challenge Name** | Under The Shell |
	| **Author** | liyanqwq |
	| **Category** | Reverse Engineering |
	| **Description** | Just cat the flag. But where is it? |
	
	---
	
	## TL;DR
	
	This challenge presents a fake web terminal that looks like a restricted Linux shell, but the important logic runs client-side in **WebAssembly (WASM)**.
	
	Trying to solve it purely from inside the shell leads to decoys, fake permissions, and misleading outputs. The key clue was:
	
	> `cat: flag.txt: Permission denied. Maybe try a deeper inspection?`
	
	That “deeper inspection” meant using **browser developer tools**. By locating the **WASM validator**, setting a **breakpoint** in the JavaScript wrapper, and dumping **WASM linear memory**, I was able to recover the real flag.
	
	---
	
	## Final Flag
	
	```text
	PUCTF26{C4t_c4T_7h3_w45M_6c138b4309f10ce058582896f8d2e581}


---

Solution Overview

- Enumerate the fake shell environment

- Identify decoys and misleading outputs

- Pivot to browser developer tools

- Locate the WASM-backed flag validator

- Pause execution with a breakpoint

- Dump and search WASM memory

- Extract the real flag


---

1. Initial Enumeration


The challenge drops us into a terminal-like environment called NuttyShell.

Running help showed a limited set of available commands, including:


- cat

- ls

- whoami

- sudo

- vim

- nvim

- b4ckd00r

A quick directory listing showed some interesting files:


	ls -la

Relevant entries included:


- flag.txt

- .flag.txt

- notes.md

- payloads/

- scans/

- secret_stash.zip

The obvious first attempt was:


	cat flag.txt

which returned:


	cat: flag.txt: Permission denied. Maybe try a deeper inspection?

This was the first strong sign that the shell itself was not the whole challenge.


---

2. Decoys and Misdirection


Trying the hidden file:


	cat .flag.txt

returned:


	flag{try_h4rder_h4rd_t0_s0lve}

At first glance this looked useful, but it did not match the provided flag format:


	PUCTF26{[a-zA-Z0-9_]+_[a-fA-F0-9]{32}}

So this was clearly a decoy.

The custom command b4ckd00r also looked suspicious. Testing it suggested it was acting as a validator instead of a file utility:


	b4ckd00r flag.txt
	b4ckd00r .flag.txt

Both returned some form of:


	Incorrect flag. Keep trying.

I also checked the notes:


	cat notes.md

which gave:


	### Lab Notes
	1. Audit the Buffer Overflow payload in payloads/
	2. Remind the team to stop using 'sudo' as a prank.
	3. Secure the credentials.

This added flavor and misdirection, but not a direct path to the solution.

Another useful observation was that listing payloads/ and scans/ seemed to loop back to the same content. That strongly suggested the shell was a simulated interface, not a real filesystem.


---

3. Pivoting to Browser Analysis


The phrase:


Maybe try a deeper inspection?


turned out to be the real hint.

Instead of continuing to poke at the shell, I opened Developer Tools with F12 and started checking the page itself.

I looked through:


- Elements

- Console

- Application
	- Local Storage

	- Session Storage

	- Cookies


None of those directly contained the flag.

The breakthrough came from the Sources tab, where I found files like:


- wasm_worker.js

- wasm_validator.js

- wasm_validator_bg.wasm

Finding a .wasm file made the challenge’s structure much clearer: the shell behavior was backed by WebAssembly, meaning the real flag-checking logic was likely inside the client.


---

4. Understanding the Validator


Inside wasm_validator.js, the key exported function looked like this:


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

This told us that:


- the frontend sends a candidate string into the WASM module

- the WASM module checks it

- the result is returned as true or false

So b4ckd00r was effectively backed by a client-side flag verifier.

That is always a big reversing clue: if the validation happens on the client, then the secret or the verification logic is somewhere we can inspect.


---

5. Accessing the WASM Scope


My first instinct was to call things directly in the console, for example:


	verify_flag("PUCTF26{test_hash}")

or to inspect memory directly:


	new TextDecoder().decode(new Uint8Array(wasm.memory.buffer))

But both failed, because verify_flag and wasm were not globally accessible. They were scoped inside the module/bundled frontend code.

To work around that, I used a breakpoint.

What I did

1. Opened wasm_validator.js in the Sources tab

2. Set a breakpoint inside verify_flag

3. Triggered the validator from the terminal using b4ckd00r

4. Let execution pause inside the function

Once execution was paused, the console had access to the current scope, including the wasm object.

This is a very useful browser reversing technique:

if a variable is not global, pause execution where it exists and inspect it there.


---

6. Dumping WASM Memory


With execution paused and wasm accessible, I dumped the module’s memory:


	new TextDecoder().decode(new Uint8Array(wasm.memory.buffer))

This returned a huge blob of data, most of it unreadable. Instead of trying to inspect it manually, I stored it and searched it with a regex:


	const memory = new TextDecoder().decode(
	    new Uint8Array(wasm.memory.buffer)
	);
	
	console.log(memory.match(/PUCTF26\{.*?\}/g));

That returned:


	["PUCTF26{C4t_c4T_7h3_w45M_6c138b4309f10ce058582896f8d2e581}"]

And that was the real flag.


---

Why This Worked


The challenge relied on client-side validation. Even though the shell UI tried to hide the answer behind fake permissions and decoy files, the browser still needed the real validation logic somewhere in memory.

Since the WASM module had to know what a valid flag looked like, the correct flag string was recoverable from WASM linear memory during execution.

In short:


- the shell was fake

- the validation was client-side

- the browser had everything needed to recover the flag

- developer tools were the intended path


---

Key Takeaways

- Not every shell challenge is really a shell challenge

- Client-side validation is reversible

- WebAssembly should be treated like any other binary

- Browser developer tools are essential for modern web-based CTFs

- Breakpoints are extremely useful when variables are not globally accessible

- Memory dumping + regex searching is often enough to recover secrets


---

Commands / Snippets Used

Shell interaction

	ls -la
	cat flag.txt
	cat .flag.txt
	cat notes.md
	b4ckd00r flag.txt
	b4ckd00r .flag.txt

WASM memory extraction

	const memory = new TextDecoder().decode(
	    new Uint8Array(wasm.memory.buffer)
	);
	
	console.log(memory.match(/PUCTF26\{.*?\}/g));


---

Closing Thoughts


This was a nice challenge because it disguised a reverse engineering problem as a simple shell task.

The fake terminal, permission errors, decoy flag, and looping directories were all there to keep the player thinking like a shell user. The real solve path only becomes obvious once you stop trusting the terminal and start inspecting the application behind it.

The hint “deeper inspection” was accurate in the best possible way: the answer was not under the shell, but behind it.
