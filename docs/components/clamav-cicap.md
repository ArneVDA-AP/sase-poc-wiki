---
title: "ClamAV + c-icap — Malware Scanning and DLP Layer 1"
tags: [clamav, c-icap, icap, antivirus, yara, dlp, respmod, opnsense, squid]
---

# ClamAV + c-icap — Malware Scanning and DLP Layer 1

**Role:** ICAP RESPMOD service on pop01 — scans all HTTP responses (downloads) for malware signatures, DLP-sensitive content (credit cards via StructuredDataDetection), and custom patterns (YARA rules for IBAN, BSN, CONFIDENTIAL labels, AWS keys).  
**Version:** ClamAV 1.x (bundled with OPNsense `os-clamav`), c-icap (bundled with `os-c-icap`)  
**Config location:** `/usr/local/etc/clamd.conf` (ClamAV), `/var/log/cicap/cicap_YYYYMMDD.log` (c-icap logs)

---

## How it works in this stack

ClamAV runs as a daemon (`clamd`) on pop01 and receives files from c-icap for scanning. c-icap is the ICAP server process that Squid communicates with. When Squid receives a response from the internet (a downloaded file, a webpage body), it forwards a copy to c-icap via ICAP RESPMOD. c-icap extracts the body, passes it to ClamAV for scanning, and returns either `ICAP/1.0 204 No Content` (clean — Squid passes it through) or a redirect to a block page (infected/DLP match — Squid returns 403).

**Why RESPMOD and not REQMOD for ClamAV:** ClamAV's `virus_scan` service correctly handles HTTP response bodies. POST request bodies (form data, file uploads) use `multipart/form-data` encoding, which c-icap's `virus_scan` does not correctly parse — uploaded content is passed as raw bytes to ClamAV, making DLP-pattern matching unreliable. For upload scanning, see [Python DLP](python-dlp.md). See [Decision: Two-layer DLP](../decisions/two-layer-dlp.md).

**DLP Layer 1 — what ClamAV detects:**
- **Malware signatures** — 3.6M+ from ClamAV's official databases (daily, main, bytecode)
- **StructuredDataDetection (SDD)** — Luhn-validated credit card detection (threshold: 3+ valid card numbers per file)
- **YARA rules** — four custom rules loaded from `/var/db/clamav/dlp_custom.yar`:
  - `DLP_Confidential_Label` — "CONFIDENTIAL", "VERTROUWELIJK", "GEHEIM", "DO NOT DISTRIBUTE" (case-insensitive)
  - `DLP_IBAN_Pattern` — NL, DE, BE IBAN format patterns
  - `DLP_BSN_Candidate` — 9-digit sequences in clusters (threshold >2)
  - `DLP_AWS_AccessKey` — `AKIA[A-Z0-9]{16}` pattern

YARA rules run *after* ClamAV's file decomposition engine — a rule matching "CONFIDENTIAL" will fire on a `.docx` file because ClamAV unpacks the ZIP container and scans the inner `word/document.xml`. This is a significant advantage over scanning raw bytes.

---

## Configuration

### Enabling ClamAV + c-icap

Install via OPNsense → System → Firmware → Plugins: `os-clamav` and `os-c-icap`.

Configure via Services → ClamAV Antivirus:
- Enable ClamAV: ✔
- Scan for filetypes: Text files, Binary files, Executables, Archives, GIF, JPEG, Office
- Allow 204 responses: ✔ (performance — no body copy when clean)

The GUI does **not** have a "Scan mode" or "Port" field (contrary to older handbook descriptions). Mode is configured on the Squid side; port 1344 is the fixed default.

### Squid ICAP integration

Via Services → Squid Web Proxy → Antivirus (ICAP):
- ICAP host: `127.0.0.1`
- ICAP port: `1344`
- ICAP service: `virus_scan`

**Critical:** After any ICAP configuration change, run `configctl proxy restart` — a GUI "Apply" alone does not activate ICAP in the running Squid daemon.

### StructuredDataDetection

In `/usr/local/etc/clamd.conf`:

```ini
StructuredDataDetection yes
StructuredMinCreditCardCount 3
StructuredMinSSNCount 3
StructuredSSNFormatNormal yes
StructuredSSNFormatStripped no
StructuredCCOnly no
```

Reload without full restart:
```bash
configctl clamav stop
configctl clamav start
```

### YARA rules

```bash
cat > /var/db/clamav/dlp_custom.yar << 'EOF'
rule DLP_Confidential_Label {
    strings:
        $s1 = "CONFIDENTIAL" nocase
        $s2 = "VERTROUWELIJK" nocase
        $s3 = "GEHEIM" nocase
        $s4 = "DO NOT DISTRIBUTE" nocase
    condition:
        any of them
}
rule DLP_IBAN_Pattern {
    strings:
        $iban_nl = /NL\d{2}[A-Z]{4}\d{10}/
        $iban_de = /DE\d{2}\d{18}/
        $iban_be = /BE\d{2}\d{12}/
    condition:
        any of them
}
rule DLP_BSN_Candidate {
    strings:
        $bsn = /\b\d{9}\b/
    condition:
        #bsn > 2
}
rule DLP_AWS_AccessKey {
    strings:
        $ak = /AKIA[0-9A-Z]{16}/
    condition:
        $ak
}
EOF
clamdscan --reload
```

YARA files in `/var/db/clamav/` are auto-loaded by ClamAV. Matches appear with `.UNOFFICIAL` suffix in logs — this is normal for custom rules.

---

## Integration points

| Component | Direction | What |
|-----------|-----------|------|
| [Squid](squid.md) | ICAP RESPMOD ← | Squid sends all HTTP responses for scanning |
| [Python DLP](python-dlp.md) | complementary | Python DLP handles upload (REQMOD); ClamAV handles downloads (RESPMOD) |
| [Suricata](suricata.md) | parallel | Both run on pop01; no interaction — different inspection domains |

**ICAP is Squid-global:** The ICAP service configuration applies to all Squid listeners automatically — both the LAN listener and the pre-auth NetBird overlay listener. No per-listener ICAP configuration is needed. This differs from SSL Bump, which must be specified per `http_port` directive.

---

## Known issues / gotchas

**GUI timeout on first database download** — the ClamAV signature download (~300 MB) takes longer than the GUI timeout. The download continues in the background. Verify with `freshclam --verbose`.

**`database.clamav.net` returns 403 for direct HTTP requests** — the CDN requires the freshclam user-agent. A 403 from the server confirms TCP/TLS works correctly; it is not a firewall problem.

**c-icap startup warnings are cosmetic** — messages like `WARNING: Can not check the used c-icap release to build service virus_scan.so` relate only to build-version verification, not runtime functionality.

**c-icap log path is non-obvious** — logs are at `/var/log/cicap/cicap_YYYYMMDD.log`, not `/var/log/c-icap/server.log`.

**YARA rules do not algorithmically validate** — `DLP_IBAN_Pattern` matches format but does not run mod-97 checksum validation. `DLP_BSN_Candidate` matches 9-digit clusters but does not run the 11-proof. False positives are possible. The [Python DLP server](python-dlp.md) on the upload path does algorithmic validation.

**`--ssl-no-revoke` required for curl.exe testing** — curl.exe on Windows uses the schannel TLS stack which attempts CRL/OCSP verification on the SSL Bump certificate. Self-signed CAs have no revocation endpoint, so this fails. Add `--ssl-no-revoke` to all CLI test commands.

**Handbook uses `/squid_clamav` as ICAP service path** — older handbook versions specify `icap://localhost:1344/squid_clamav`. The correct service path on OPNsense's c-icap installation is `/virus_scan`. Using the handbook path returns ICAP 404.

**YARA persistence after OPNsense firmware updates** — YARA files in `/var/db/clamav/` survive OPNsense updates as long as they are not in update-managed directories. After a firmware update, verify that `dlp_custom.yar` is still present and `clamdscan --reload` completes without error.

**ClamAV/ICAP scope limitations** — ClamAV can only inspect traffic that flows through Squid. The following traffic is NOT inspected:
- Sites in the SSL Bump no-bump list (e.g. `.microsoft.com`) — encrypted bytes pass through without decryption
- Clients that do not use WPAD/PAC or have disabled the proxy setting — their traffic bypasses Squid entirely
- Non-HTTP protocols (SSH, SMB, DNS to external servers) — handled by Suricata, not ClamAV

---

## Related

- [Architecture overview](../overview/architecture.md)
- [Concept: ICAP](../concepts/icap.md)
- [Concept: DLP](../concepts/dlp.md)
- [Concept: SSL Bump](../concepts/ssl-bump.md)
- [Component: Squid](squid.md)
- [Component: Python DLP](python-dlp.md)
- [Decision: Two-layer DLP](../decisions/two-layer-dlp.md)
- [Runbook: Malware & DLP Pipeline](../runbooks/04-malware-dlp.md)
