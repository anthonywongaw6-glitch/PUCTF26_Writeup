# QtLicense — Writeup

## Challenge Info

| Field | Value |
|---|---|
| **Challenge Name** | QtLicense |
| **Author** | PolyU CTF |
| **Category** | Reverse Engineering |
| **Description** | Activate the software with a valid license key. |

---

## TL;DR

This challenge provides a **Qt-based Windows application** that verifies license keys using a remote API.

By reversing the binary, we discover that the client sends the following JSON request to the server:

```json
{
  "license_key": "...",
  "server_time": "...",
  "is_4dm1n_m0de": 0
}
```

The hidden parameter `is_4dm1n_m0de` is always set to `0`.

By manually sending the request with:

```
"is_4dm1n_m0de": 1
```

we can bypass the normal verification logic and retrieve the flag.

---

# Solution Overview

The solving process:

1. Extract the Qt application
2. Inspect strings to discover API endpoints
3. Reverse the activation workflow in IDA
4. Identify the hidden `is_4dm1n_m0de` parameter
5. Recreate the server request manually
6. Enable admin mode to bypass verification
7. Retrieve the flag

---

# 1. Extracting the Application

The challenge provides a ZIP archive containing a Qt application and its dependencies.

```bash
unzip License_v2.zip
```

Important files include:

```
QtLicense.exe
Qt5Core.dll
Qt5Gui.dll
Qt5Network.dll
Qt5Widgets.dll
```

The main executable is:

```
QtLicense.exe
```

---

# 2. Inspecting the Binary

First identify the file type:

```bash
file QtLicense.exe
```

Output:

```
PE32+ executable for MS Windows (x86-64)
```

Since this is a Windows GUI application, we analyze it using **IDA Pro**.

---

# 3. Discovering Important Strings

Opening the **Strings window** in IDA (`Shift + F12`) reveals several useful strings:

```
/license/verify
/time
server_time
license_key
License key is incorrect.
License key is valid.
Server time mismatch. Please try again.
is_4dm1n_m0de
https://chal.polyuctf.com:11337
```

These indicate that the application communicates with a remote API.

---

# 4. Understanding the Activation Workflow

Tracing the functions in IDA reveals the activation process.

### Activate Button Handler

Function: `sub_140001E70`

Key behavior:

1. Read license key from the input field
2. Send a request to `/time`
3. Parse `"server_time"` from the response
4. Send another request to `/license/verify`

---

# 5. Inspecting License Verification

Function: `sub_140002330`

This function builds the JSON request sent to the server.

Relevant code:

```c
QString("license_key")
QString("server_time")
QString("is_4dm1n_m0de")

QJsonValue::QJsonValue((QJsonValue *)v14, 0);
```

Which results in the following payload:

```json
{
  "license_key": "<user input>",
  "server_time": "<server time>",
  "is_4dm1n_m0de": 0
}
```

The important observation is the hidden field:

```
"is_4dm1n_m0de"
```

The client **always sets it to 0**.

---

# 6. Fetching Server Time

The server requires a valid timestamp.

```bash
curl -k https://chal.polyuctf.com:11337/time
```

Example response:

```json
{"server_time":"2026-03-07T20:00:35.694306+00:00"}
```

---

# 7. Reproducing the Request Manually

Instead of using the client, we send the request ourselves.

The key trick is to change:

```
"is_4dm1n_m0de": 0
```

to:

```
"is_4dm1n_m0de": 1
```

Example request:

```bash
curl -k https://chal.polyuctf.com:11337/license/verify \
  -H "Content-Type: application/json" \
  -d '{
    "license_key":"AAAAA-AAAAA-AAAAA-AAAAA-AAAAA",
    "server_time":"<SERVER_TIME>",
    "is_4dm1n_m0de":1
}'
```

If the timestamp is stale, the server returns:

```json
{"status":"time_mismatch"}
```

So the request must be sent immediately after retrieving `/time`.

---

# 8. Automating the Attack

To avoid timing issues, we can automate the process.

```python
import requests
import urllib3

urllib3.disable_warnings()

BASE = "https://chal.polyuctf.com:11337"
LICENSE = "AAAAA-AAAAA-AAAAA-AAAAA-AAAAA"

s = requests.Session()
s.verify = False

t = s.get(f"{BASE}/time").json()["server_time"]

payload = {
    "license_key": LICENSE,
    "server_time": t,
    "is_4dm1n_m0de": 1
}

r = s.post(f"{BASE}/license/verify", json=payload)
print(r.text)
```

Running this script sends the modified request with admin mode enabled.

---

# Root Cause

The vulnerability exists because the server trusts a **client-controlled field**:

```
"is_4dm1n_m0de"
```

The application sets this to `0`, but the server apparently grants elevated privileges when it is set to `1`.

Since the client cannot be trusted, this results in a **privilege escalation vulnerability**.

---

# Final Flag

```
PUCTF26{...} again, i forgor to save the flag before submitting 
```

---

# Key Takeaways

- Client-side binaries often expose hidden API parameters
- Never trust security decisions made by the client
- Reverse engineering can reveal undocumented server features
- API request replication is often easier than patching binaries
