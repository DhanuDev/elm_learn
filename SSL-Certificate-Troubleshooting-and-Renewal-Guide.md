# SSL/TLS Certificate Troubleshooting, Installation & Renewal Guide

**Scope:** Diagnosing "Not secure" browser warnings, checking certificate expiry, and running a repeatable install/renewal process for production applications.

---

## 1. Understanding the "Not secure" Warning

When Chrome/Edge/Firefox show **"Not secure"** next to an `https://` URL (as in the example screenshot of a `.gov.sa` login page), it almost always means one of these:

| Cause | What the browser detected |
|---|---|
| **Expired certificate** | Current date is past the cert's `Not After` date |
| **Self-signed / untrusted CA** | Cert isn't signed by a CA in the browser's trust store |
| **Hostname mismatch** | Cert's CN/SAN doesn't match the URL (e.g., cert for `swa.gov.sa` but site is `abr.swa.gov.sa`) |
| **Incomplete chain** | Server sent the leaf cert but not the intermediate CA cert |
| **Mixed content** | Page is HTTPS but loads some resources (scripts/images/iframes) over HTTP |
| **Revoked certificate** | Cert appears on the CA's CRL/OCSP revocation list |
| **Deprecated protocol/cipher** | Server still using TLS 1.0/1.1 or weak ciphers, which modern browsers flag |

The red strikethrough on "https" specifically (rather than a generic "Not secure" on an `http://` site) points to **certificate validity**, not a plaintext connection — so start with cert diagnosis, not firewall/network checks.

---

## 2. How to Check SSL Certificate Expiry

### A. Browser (quickest, manual, one-off)
1. Click the padlock / "Not secure" label in the address bar.
2. Select **Connection is secure → Certificate is valid (or invalid)**.
3. View **Details** tab → check **Valid from / Valid to** dates and the **Certification path** (chain).

### B. Command line — `openssl` (best for servers/scripts)
Check a live endpoint:
```bash
echo | openssl s_client -connect abr.swa.gov.sa:443 -servername abr.swa.gov.sa 2>/dev/null | openssl x509 -noout -dates -subject -issuer
```
Output gives:
```
notBefore=Jan  1 00:00:00 2026 GMT
notAfter=Apr  1 23:59:59 2026 GMT
subject=CN=abr.swa.gov.sa
issuer=CN=Some CA, O=Some Org
```

Check a local certificate file:
```bash
openssl x509 -in /etc/ssl/certs/mycert.crt -noout -dates -subject -issuer
```

Check days remaining programmatically:
```bash
DOMAIN=abr.swa.gov.sa
EXPIRY=$(echo | openssl s_client -connect $DOMAIN:443 -servername $DOMAIN 2>/dev/null | openssl x509 -noout -enddate | cut -d= -f2)
EXPIRY_EPOCH=$(date -d "$EXPIRY" +%s)
NOW_EPOCH=$(date +%s)
echo "Days remaining: $(( (EXPIRY_EPOCH - NOW_EPOCH) / 86400 ))"
```

### C. Verify the full chain (catches "incomplete chain" issues)
```bash
openssl s_client -connect abr.swa.gov.sa:443 -servername abr.swa.gov.sa -showcerts
```
Look for `Verify return code: 0 (ok)` at the bottom. Anything else (e.g., `unable to get local issuer certificate`) means the intermediate cert isn't being served.

### D. Online tools (no shell access needed)
- **SSL Labs (Qualys)** — `https://www.ssllabs.com/ssltest/` — full grade, protocol/cipher audit, chain check.
- **whynopadlock.com** / **sslshopper.com** — quick expiry + mixed-content checks.
- **crt.sh** — search Certificate Transparency logs for all certs ever issued for a domain (useful for spotting rogue/expired certs still in DNS).

### E. Continuous monitoring (recommended for production)
Don't rely on someone remembering to check manually. Use one of:
- **Uptime/monitoring tools** (UptimeRobot, Pingdom, Datadog Synthetics, Zabbix, Nagios `check_ssl_cert` plugin) — alert at 30/14/7/1 days before expiry.
- **Cloud-native**: AWS ACM auto-renews and emits CloudWatch events; Azure/GCP have equivalent managed-cert expiry alerts.
- **Cron + script** (shown above) piped to Slack/email/PagerDuty.

---

## 3. Live Troubleshooting Process (Production Incident)

When a production site suddenly shows "Not secure," work through this in order:

```
1. Confirm scope
   └─ Is it one hostname, all subdomains, or the whole domain?
   └─ Is it every user, or only some regions (CDN edge issue)?

2. Check expiry immediately
   └─ openssl s_client ... -noout -dates  (Section 2B)
   └─ If expired → jump to Section 5 (Renewal)

3. Check hostname match
   └─ openssl x509 -noout -text | grep -A1 "Subject Alternative Name"
   └─ Compare against the exact URL in the browser

4. Check chain completeness
   └─ openssl s_client -showcerts, look for "Verify return code: 0 (ok)"
   └─ If broken, the intermediate .crt / bundle isn't configured on the server/LB

5. Check for mixed content
   └─ Open DevTools → Console tab → look for "blocked mixed content" warnings

6. Check load balancer / CDN / reverse proxy
   └─ If behind Nginx/HAProxy/Cloudflare/ALB, confirm the cert is installed
      at the edge that actually terminates TLS, not just the origin server

7. Check clock skew
   └─ Server with wrong system time can cause premature "not yet valid"
     or false "expired" errors → `timedatectl status` / `date`
```

### Immediate mitigation (while root-causing)
- If expired and you have a **backup/previous valid cert** with time left, temporarily roll back.
- If using a **CDN (Cloudflare, CloudFront, Akamai)**, some offer "Flexible/Full" SSL modes that can mask an origin issue temporarily — but treat this as a stopgap, not a fix.
- Communicate status to stakeholders; for government/regulated sites, downtime or "not secure" exposure may need incident logging per your org's compliance policy.

---

## 4. SSL Certificate Installation Process

### Step 1 — Generate a Private Key + CSR (Certificate Signing Request)
```bash
openssl req -new -newkey rsa:2048 -nodes \
  -keyout abr.swa.gov.sa.key \
  -out abr.swa.gov.sa.csr \
  -subj "/C=SA/ST=Riyadh/L=Riyadh/O=SWA/CN=abr.swa.gov.sa"
```
- Keep the `.key` file secret — never commit it to source control, and set permissions `chmod 600`.
- For multi-domain/subdomain coverage, use a config file with `subjectAltName` (SAN) entries instead of a single CN.

### Step 2 — Submit CSR to a Certificate Authority (CA)
Options:
- **Commercial CA** (DigiCert, GlobalSign, Sectigo) — for EV/OV certs often required by government/enterprise policy.
- **Let's Encrypt** (free, automated, 90-day certs) — via **Certbot** or **ACME** clients, ideal when automation is allowed.
- **Internal/private CA** — for internal-only services (intranet apps), not for public-facing sites (browsers won't trust it — this is the "self-signed" warning case).

The CA validates domain ownership (DNS TXT record, HTTP file, or email) and issues:
- The **leaf/server certificate**
- One or more **intermediate certificates** (the "chain")

### Step 3 — Install on the Server

**Nginx**
```nginx
server {
    listen 443 ssl;
    server_name abr.swa.gov.sa;

    ssl_certificate     /etc/ssl/certs/abr.swa.gov.sa.fullchain.crt;  # leaf + intermediates
    ssl_certificate_key  /etc/ssl/private/abr.swa.gov.sa.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
}
```
Build the fullchain with:
```bash
cat abr.swa.gov.sa.crt intermediate.crt > abr.swa.gov.sa.fullchain.crt
```

**Apache**
```apache
<VirtualHost *:443>
    ServerName abr.swa.gov.sa
    SSLEngine on
    SSLCertificateFile      /etc/ssl/certs/abr.swa.gov.sa.crt
    SSLCertificateKeyFile   /etc/ssl/private/abr.swa.gov.sa.key
    SSLCertificateChainFile /etc/ssl/certs/intermediate.crt
</VirtualHost>
```

**IIS (Windows)**
1. Import the `.pfx` (cert + key + chain bundled) via **IIS Manager → Server Certificates → Import**.
2. Bind it to the site: **Site → Bindings → Add → Type: https → select the certificate**.

**Cloud Load Balancers / CDNs**
- **AWS ALB/CloudFront**: upload/import via **ACM (AWS Certificate Manager)** — supports free auto-renewing public certs.
- **Cloudflare**: upload under **SSL/TLS → Edge Certificates**, or use Cloudflare's free "Universal SSL."
- **Azure App Service / Front Door**: upload `.pfx` under **TLS/SSL settings**.

### Step 4 — Verify Installation
```bash
openssl s_client -connect abr.swa.gov.sa:443 -servername abr.swa.gov.sa -showcerts
```
Confirm:
- `Verify return code: 0 (ok)`
- Correct `notAfter` date
- Correct SAN entries
- Then re-test with SSL Labs for a full grade/report.

---

## 5. Renewal Process

### Manual renewal (commercial CA)
1. Generate a **new CSR** (can reuse the existing key or generate a new one — new key is better practice).
2. Submit to the CA ~30 days before expiry (most CAs allow early renewal that extends from the current expiry date).
3. Complete domain validation again.
4. Download the new cert + chain.
5. Replace files on server/LB and reload the service (not just restart — reload avoids downtime):
   ```bash
   sudo nginx -t && sudo systemctl reload nginx
   # or
   sudo apachectl configtest && sudo systemctl reload apache2
   ```
6. Re-verify with `openssl s_client` and SSL Labs.

### Automated renewal (recommended — removes human error)
**Let's Encrypt + Certbot** (auto-renews 90-day certs):
```bash
sudo certbot --nginx -d abr.swa.gov.sa    # first-time issue + auto-configures nginx
sudo certbot renew --dry-run              # test the renewal hook
```
Certbot installs a systemd timer/cron job that renews automatically when <30 days remain, and reloads the web server via a deploy hook.

**Cloud-managed certs** (AWS ACM, Cloudflare Universal SSL, GCP-managed certs): renewal is fully automatic as long as DNS validation records remain in place — no manual action needed, but still worth monitoring for renewal failures (e.g., DNS record accidentally removed).

### Renewal checklist (put this in your runbook)
- [ ] Alert configured for 30/14/7/1 days before expiry
- [ ] CSR/key generation process documented and access-controlled
- [ ] Chain/intermediate bundled correctly (test with `-showcerts`)
- [ ] Staging/test environment validated before production rollout
- [ ] Reload (not hard restart) used to avoid downtime
- [ ] Post-renewal verification via `openssl` + SSL Labs
- [ ] Old cert/key securely archived or destroyed per policy
- [ ] Change logged in your CMDB/change-management system

---

## 6. Best Practices to Prevent Recurrence

1. **Automate everything possible** — ACME (Let's Encrypt) or cloud-managed certs eliminate the #1 cause of outages: someone forgetting to renew.
2. **Alert well before expiry** — 30 days minimum, escalating at 14/7/1 days.
3. **Use SAN certs or wildcards** for environments with many subdomains, reducing the number of certs to track.
4. **Centralize cert inventory** — a simple spreadsheet or a tool like HashiCorp Vault/cert-manager (for Kubernetes) tracking every cert, its expiry, and owner.
5. **Terminate TLS at a single, well-documented layer** (LB/CDN) rather than scattered across multiple origin servers, so there's one place to renew and verify.
6. **Test in staging first** — a renewal that changes the key or chain should be validated before touching production.
7. **Monitor Certificate Transparency logs** (crt.sh) for unexpected certs issued for your domains — an early signal of misconfiguration or compromise.

---

### Quick Reference — One-Liners

| Task | Command |
|---|---|
| Check live site expiry | `echo \| openssl s_client -connect HOST:443 -servername HOST 2>/dev/null \| openssl x509 -noout -dates` |
| Check local cert file expiry | `openssl x509 -in cert.crt -noout -enddate` |
| Verify chain | `openssl s_client -connect HOST:443 -showcerts` |
| Check SAN entries | `openssl x509 -in cert.crt -noout -text \| grep -A1 "Subject Alternative Name"` |
| Test cert/key match | `openssl x509 -noout -modulus -in cert.crt \| openssl md5` vs `openssl rsa -noout -modulus -in key.key \| openssl md5` (must match) |
| Certbot dry-run renewal | `certbot renew --dry-run` |
