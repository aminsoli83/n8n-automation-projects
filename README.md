# AD Assistant — an AI agent for Active Directory, built on n8n

An AI help-desk assistant that lets IT staff ask questions about Active Directory in plain **English or Persian (Farsi)**: *"Which groups is Sara in?"*, *"What is the IP of this user's laptop?"*, *"Whose password expires in the next 10 days?"*, *"علی رضایی رو پیدا کن"*.

The agent runs on **n8n** with a **local LLM (Ollama, Qwen 3.5 9B)**, so no directory data leaves the network and there are no per-token API costs. Every answer comes from live AD lookups; the model only chooses the tool and writes the reply. Changes to AD are never made by the model itself: it proposes them, and a person approves each one in the web UI.

> Built and run in production for the IT team of a multi-company holding (~8,000 computers, several AD OUs).

---

## Features

**Questions (read-only)**
- Find users by name in Latin or Persian script, username, email or employee number. Persian names are matched by consonant skeletons, so spelling variants (ژ → zh / j / z, ق / غ → gh / q, ی / ي) still match.
- User details: title, department, manager, account status, lockout, password last set / expiry, last logon, direct and nested groups
- A user's computers (from the computer description or the employee number in the computer name, filtered by company code) and their IP addresses, taken from DNS with a reverse-DNS (PTR) fallback
- Computers, groups and OUs: details, owners, members
- Reports: locked users, passwords expiring in N days, stale computers, object counts, members of an OU or group
- Group Policy: which GPOs apply to a user / computer and why others don't (link order, security filtering, WMI filters, inheritance blocking)

**Changes (human-approved)**
- Add or remove a user to/from a group, enable / disable, move to another OU
- The model emits a structured `ad-change` block; the web UI shows **Yes / Cancel**, and a separate service account with delegated rights only (no admin groups) carries out the change after protection rules are checked (protected groups, allowed approvers)

**Offboarding workflow** (one button, step-by-step with live status)
1. Disable the AD account
2. Back up the mailbox to PST (Exchange remote PowerShell from Linux, with retries and corrupt-item handling)
3. Remove all group memberships (the list is kept for the audit trail)
4. Disable Skype for Business, 5. disable the mailbox (via ManageEngine ADManager Plus REST API)
6. Move the account to the leavers OU

**Web UI**
- Chat with answer history, "Find a user" card (name, email, OU, computer, IP, phone), offboarding page, admin audit log
- AD sign-in (LDAPS), role-based access by AD group, CSRF protection, every lookup and change written to the audit log

---

## Architecture

```
 Browser ──HTTPS──► Web UI (Python, stdlib only)
                      │  sign-in via LDAPS, audit log (SQLite), approval of changes
                      │
                      ├──► n8n webhook ──► AI Agent ──► Ollama (Qwen 3.5 9B, local GPU server)
                      │                       │
                      │                       ├── ad_lookup        (HTTP tool)  ─┐
                      │                       ├── report sub-workflows           │
                      │                       └── get_applied_gpos (HTTP tool)  ─┤
                      │                                                          ▼
                      ├──► AD helper (Python) ──LDAPS──► Domain controllers (read-only account)
                      │        └── DNS helper ──► DNS servers (A / PTR)
                      │
                      └──► Offboarding engine ──► AD (delegated change account)
                                                ├─► Exchange (pypsrp + Kerberos)
                                                └─► ADManager Plus REST API
```

**Design choices**
- **Small model, smart tools.** Each tool returns a short, ready-to-use JSON answer (candidates, notes, errors written for the model), so a 9B local model is enough. This cut the prompt size from 3,000–5,000 tokens per question (with a hosted model) to a fraction, and removed rate limits.
- **Focus OUs.** Most tickets come from a few OUs, so searches rank those first instead of scanning the whole directory.
- **Least privilege.** Lookups use a read-only account; changes use a separate account with delegated rights only; secrets live in root-only config files, base64-encoded.
- **No agents on PCs.** Mailbox export runs from the Linux server through Exchange remote PowerShell; nothing is installed on staff computers.

---

## Repository layout

```
n8n/
  ad-agent.json                 main workflow: webhook → AI Agent → Ollama + tools
  sub-workflows/
    AD Group Members.json       report sub-workflows called as tools
    AD Locked Users.json
    AD OU Computers.json
    AD OU Users.json
    AD Password Expiry.json
    AD Stale Computers.json
  prompts/
    system-prompt.txt           agent instructions (language rule, tools, change format)
docs/
  screenshots/
```

---

## Setup

**Requirements:** n8n (self-hosted), Ollama with `qwen3.5:9b` (or another tool-calling model), an AD read-only service account, and the AD helper service (LDAPS lookups exposed as `GET /ad?action=&q=` with an API key).

1. **Import the workflows:** in n8n → *Workflows → Import from file*, import `ad-agent.json` and the files in `sub-workflows/`.
2. **Model:** in the *Ollama Chat Model* node, set the base URL (e.g. `http://<ollama-host>:11434`), model `qwen3.5:9b`, context length `16384`.
3. **Tools:** in the `ad_lookup` HTTP Request Tool, set the AD helper URL and an `X-API-Key` header credential. Parameters are filled by the model through `$fromAI('action')` and `$fromAI('q')`.
4. **Prompt:** paste `prompts/system-prompt.txt` into the AI Agent's system message.
5. **Webhook:** activate the workflow and point the web UI (or any client) at the production webhook URL.
6. **Long reports:** for large result sets (e.g. password expiry over 90 days), keep the Summarize node's row limit low and raise the client timeout.

---

## Example

```
Q:  Which computer does Ali Rezaei use, and what is its IP?
A:  Ali Rezaei (rezaei.ali) has 1 computer:
    | Computer       | IP            |
    | ACM001N012345  | 10.10.25.40   |
```

```
Q:  Add ahmadi.sara to the VPN-Users group
A:  Sara Ahmadi (ahmadi.sara) will be added to VPN-Users.   [ Yes ]  [ Cancel ]
```

---

## Tech stack

n8n · Ollama (Qwen 3.5 9B) · Python (stdlib HTTP server, raw LDAP v3 over TLS) · LDAP / Active Directory · DNS · Exchange remote PowerShell (pypsrp, Kerberos) · ManageEngine ADManager Plus REST API · SQLite · systemd · Ubuntu

---

## Security notes

- Exported workflows in this repository contain **no credentials**; n8n stores them separately. Hostnames, IPs and domain names have been replaced with placeholders.
- The model can't change AD on its own: every change needs a person to approve it, is checked against protection rules, and is written to the audit log.

