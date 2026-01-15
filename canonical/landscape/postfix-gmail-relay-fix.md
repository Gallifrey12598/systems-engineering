# Postfix → Gmail Relay Troubleshooting (Clean Summary)

This document consolidates the troubleshooting steps and findings from the Postfix mail relay errors encountered when sending mail through **smtp.gmail.com**. Repeated information has been removed and the remaining content is organized into a clear, actionable flow.

---

## 1. Problem Overview

Postfix is configured to relay outbound mail through Gmail (`smtp.gmail.com:587`). Multiple failures occurred during setup, progressing through several distinct error states:

1. TLS handshake failures
2. IPv6 connectivity failures
3. Gmail authentication required (530 5.7.0)
4. SASL authentication failure: `No worthy mechs found`

Each stage indicated progress, but also revealed the next missing dependency or misconfiguration.

---

## 2. Current Status (What Is Working)

The following items are **confirmed working**:

* Postfix successfully starts and queues mail
* STARTTLS negotiation with Gmail over IPv4 works
* Relay host is correctly defined as `[smtp.gmail.com]:587`
* TLS configuration is correct for port 587 (explicit TLS)
* SASL authentication is enabled in Postfix
* Required SASL packages are installed:

  * `libsasl2-modules`
  * `libsasl2-modules-db`
  * `sasl2-bin`

Postfix is now **attempting** SASL authentication, which confirms earlier TLS issues are resolved.

---

## 3. Remaining Primary Issue

### SASL Error

```
SASL authentication failure: No worthy mechs found
SASL authentication failed; no mechanism available
```

### Meaning

Postfix is correctly configured to authenticate, but **no usable SASL client mechanisms** (such as `PLAIN` or `LOGIN`) are available to the SMTP client.

Gmail requires `AUTH PLAIN` or `AUTH LOGIN`. Without these mechanisms, authentication cannot proceed.

---

## 4. Root Cause

Although `libsasl2-modules` is installed, the **PLAIN (and optionally LOGIN) plugins are missing** from the SASL plugin directory.

Current plugin directory contents show only:

* `ANONYMOUS`
* `CRAM-MD5`
* `DIGEST-MD5`

But **not**:

* `libplain.so`
* `liblogin.so`

Without `libplain.so`, Postfix cannot authenticate to Gmail.

---

## 5. Required Fix

### Step 1: Verify Missing Plugins

Run:

```bash
ls -l /usr/lib/x86_64-linux-gnu/sasl2/libplain* \
      /usr/lib/x86_64-linux-gnu/sasl2/liblogin* 2>/dev/null
```

If nothing is returned, the plugins are missing.

---

### Step 2: Reinstall SASL Modules

Force reinstall to ensure all plugins are present:

```bash
sudo apt-get install --reinstall -y libsasl2-modules
```

Verify the files exist:

```bash
dpkg -L libsasl2-modules | egrep 'libplain|liblogin'
```

You should see at least `libplain.so`.

---

### Step 3: Confirm Available Mechanisms

Run:

```bash
sasl2listmech
```

Expected output should include:

```
PLAIN
LOGIN
```

If these appear, SASL is correctly installed.

---

## 6. Postfix Configuration (Final Expected State)

Ensure the following settings exist in `/etc/postfix/main.cf`:

```ini
relayhost = [smtp.gmail.com]:587

smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_sasl_tls_security_options = noanonymous

smtp_tls_security_level = encrypt
smtp_tls_CApath = /etc/ssl/certs
```

Ensure `/etc/postfix/sasl_passwd` uses **exactly** this key:

```
[smtp.gmail.com]:587 youraccount@gmail.com:YOUR_GOOGLE_APP_PASSWORD
```

Then rebuild and restart:

```bash
sudo postmap /etc/postfix/sasl_passwd
sudo systemctl restart postfix
```

---

## 7. IPv6 Warning (Secondary Issue)

Repeated log entries:

```
connect to smtp.gmail.com[2607:...]:587: Network is unreachable
```

### Cause

The host does not have functional IPv6 routing.

### Recommended Fix

Force Postfix to use IPv4 only:

```bash
sudo postconf -e 'inet_protocols = ipv4'
sudo systemctl restart postfix
```

This does **not** affect authentication but removes noise and retry delays.

---

## 8. Final Validation Steps

After fixing SASL plugins:

1. Run:

   ```bash
   sasl2listmech
   ```

   Confirm `PLAIN` is listed

2. Flush the mail queue:

   ```bash
   sudo postqueue -f
   ```

3. Watch logs:

   ```bash
   tail -n 100 /var/log/mail.log
   ```

Expected result:

* Successful `AUTH PLAIN`
* Message accepted by Gmail
* No more `No worthy mechs found` errors

---

## 9. Summary

* TLS is fixed
* Gmail relay is reachable
* Authentication is required and correctly configured
* **Final blocker is missing `libplain.so` SASL plugin**
* Reinstalling `libsasl2-modules` should fully resolve the issue

Once `PLAIN` is available, Postfix → Gmail relay should function normally.
