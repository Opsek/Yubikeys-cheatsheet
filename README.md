# YubiKeys-cheatsheet

This repo contains tips on how to properly use your YubiKeys.

- 1️⃣ Step by step YubiKey setup
- 2️⃣ Tutorial on how to sign GitHub commits with YubiKey
- More to come 😉

## YubiKey Interfaces / Applications

| Interface / App | What it is | Phishing-resistant | How you use it | Where / When to use it | Concrete examples |
|-----------------|------------|--------------------|---------------|------------------------|-------------------|
| **FIDO2** | Modern passwordless and 2FA authentication standard (WebAuthn + CTAP2) | **Yes** | Touch the YubiKey when prompted by the browser or OS | Best choice for web logins and OS authentication | Register passkeys on Google, GitHub, Microsoft, AWS; passwordless login or strong 2FA |
| **FIDO U2F** | Legacy FIDO standard (CTAP1), requires password first | **Yes** | Enter username/password, then touch the YubiKey | Older services that don’t support full FIDO2 | GitHub legacy security key login; older VPNs, NAS devices |
| **OATH (TOTP / HOTP)** | One-time password generator stored on the YubiKey | **No** | Use **Yubico Authenticator** to read 6- or 8-digit codes | When a service only supports OTP-based 2FA | Generate TOTP for GitHub, Google, servers; fallback if passkeys/security keys aren’t supported |
| **OpenPGP** | Smart card for PGP keys (signing, encryption, authentication) | **Yes** (for auth & signing) | Use `gpg`, email clients, SSH via GPG agent | Developer workflows, cryptographic identity | Sign Git commits; encrypt/decrypt emails; SSH login using GPG |
| **PIV (Smart Card)** | PKI smart card using X.509 certificates | **Yes** | Used automatically by OS, browsers, VPN clients | Enterprise, government, system authentication | Windows/macOS smart-card login; VPN authentication; client TLS certificates |
| **Yubico OTP** | Yubico proprietary one-time password | **No** | Touch key to type a long OTP string | Legacy systems and simple integrations | PAM authentication on servers; legacy VPNs; Yubico validation service |
| **HSM (HMAC / Secure Key Storage)** | Secure cryptographic operations inside the key | **Yes** (challenge-response) | Used by applications, not manually | Protect secrets and keys | LUKS disk unlock; challenge-response authentication |
| **NFC (Transport)** | Wireless communication channel | Depends on app | Tap key on phone or reader | Mobile and portable authentication | FIDO2 login on Android/iOS; OATH via NFC |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## 1️⃣ YubiKey set up

### Download required tools

- **Yubico Authenticator** (To manage interfaces and set a PIN)  
  🔗 [Download here](https://www.yubico.com/products/yubico-authenticator/#h-download-yubico-authenticator)

- **YubiKey Manager** (for `ykman` command-line)  
  🔗 [Download last release](https://github.com/Yubico/Yubikey-manager/releases/)

### 📹 Tutorial Video

Watch a step-by-step video covering all of the steps:  
👉 [Watch the tutorial video](https://drive.google.com/file/d/1qxiXehIEOEhaRE4ae_PsudLVubsnKyEN/view?usp=sharing)

[![YubiKey Tutorial](yubikey-tutorial.png)](https://drive.google.com/file/d/1qxiXehIEOEhaRE4ae_PsudLVubsnKyEN/view?usp=sharing)

### Steps to repeat on all your YubiKeys

1. Verify your device is genuine:  
   🔗 [Yubico genuine check](https://www.yubico.com/genuine/)

2. Set a PIN on the YubiKey  
   - Use Yubico Authenticator → **Passkeys** tab  
   - PIN should be **4–6 digits**  
   - ⚠️ Recommended: use the **same PIN on all your YubiKeys** for simplicity

3. Enforce the PIN request when using FIDO (optional but recommended)  

```bash
ykman fido config toggle-always-uv
```
> 💡 Not supported on old Yubikeys, firmware prior to 5.7
>

### ✅ Your YubiKeys are ready

Now register them on **all of your accounts**.

You can check which apps and platforms support YubiKeys here:  
👉 https://safecheck.opsek.io/

#### Example: Set up a YubiKey on Gmail (Google Account)

1. Go to **Google Account Settings**
2. Open **Security**
3. Go to **Signing in to Google**
4. Enable **2-Step Verification (2FA)** if it’s not already enabled
5. Navigate to **Passkeys & Security Keys**
6. Click **Add security key**  
   - *or* **Add a passkey** and **save it directly on the YubiKey**
7. Insert your YubiKey and follow the on-screen instructions
8. Once registered, rename your YubiKeys (Nano, 5C NFC 1, 5C NFC backup)

#### ⚠️ Important Notes

If you registered your YubiKey and **it did not ask you for a PIN**, you probably missed a step or did not complete the setup correctly. Make sure a **FIDO2 PIN is set** on the YubiKey and that you are adding a **security key / passkey**, not a less secure method.

**Do NOT store passkeys or security keys in 1Password or any password manager**  
Password managers should **only** be used to store passwords.  
Your YubiKey should be the **only place** where the passkey/security key is stored.

#### Best practices
- Register **at least two YubiKeys** (primary + backup)
- Keep your backup key in a safe, separate location
- Test sign-in with each key after setup

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## 2️⃣ Sign your commits with your YubiKey

### Prerequisites

Before you begin, make sure you have:

- Enabled the **OpenPGP interface** on your YubiKey (via Yubico Authenticator)  
  🔗 https://www.yubico.com/products/yubico-authenticator/

- Installed **GnuPG (GPG)**  
  GitHub: https://github.com/gpg/gnupg

- Installed **YubiKey Manager** (for `ykman`)  
  GitHub: https://github.com/Yubico/YubiKey-manager

### 🔐 YubiKey GPG (ed25519) → Signed Git Commit

#### 1) Configure GPG for terminal prompts (no GUI)

```bash
mkdir -p ~/.gnupg && chmod 700 ~/.gnupg

cat > ~/.gnupg/gpg.conf <<'EOF'
use-agent
pinentry-mode loopback
EOF

# allow loopback in the agent (no GUI pinentry needed)
grep -q '^allow-loopback-pinentry' ~/.gnupg/gpg-agent.conf 2>/dev/null || \
  echo 'allow-loopback-pinentry' >> ~/.gnupg/gpg-agent.conf

# reload agent
gpgconf --kill gpg-agent
```

#### 2) (Optional) Clean-slate the YubiKey OpenPGP applet

> ⚠️ Destroys any existing GPG keys on the key.
> 

```bash
ykman openpgp reset -f
ykman openpgp access change-pin
ykman openpgp access change-admin-pin

```

#### 3) Set strong key algorithms & generate **on the card**

```bash
gpg --card-edit
```

At the `gpg>` prompt, type exactly this (press ⏎ after each line):

```
admin
key-attr
```

Then, when prompted **for each slot**, choose:

- **Algorithm** → `2` (ECC)
- **Curve** → `1` (Curve25519)

You should end up with:

```
Signature key ....: ed25519
Encryption key....: cv25519
Authentication....: ed25519
```

Now generate the keys on the card:

```
generate
```

- **Make off-card backup?** → `n`
- Enter **Name** and **Email** (use your GitHub email)
- Choose an **expiry** (e.g., 1y)

- Enter **Admin PIN** and **User PIN** when prompted.  

  **Defaults for a new YubiKey OpenPGP applet:**
  - **User PIN:** `123456`
  - **Admin PIN:** `12345678`

> ⚠️ **Important:** These default PINs are **not secure**. You **must change them immediately** using:
>
> ```bash
> ykman openpgp access change-pin
> ykman openpgp access change-admin-pin
> ```
>
> Choose **strong, memorable PINs** for both User and Admin.


Exit:

```
quit
```

#### 4) Verify keys are on the card

```bash
gpg --card-status
```

Expect to see `ED25519 / CV25519` for the three slots.

#### 5) Export your **public** key and add to GitHub (for commit verification)

```bash
EMAIL="you@example.com"
gpg --armor --export "$EMAIL" > ~/YubiKey-gpg-public.asc
cat ~/YubiKey-gpg-public.asc
```

➡️ copy everything and paste at: https://github.com/settings/keys  →  "New GPG key"

#### 6) Configure Git to sign commits with your YubiKey

1. Show the YubiKey key information:

```bash
gpg --card-status
```

2. Find the line starting with General key info:

```bash
pub  ed25519/AAAAAAAAAAAAAAAA 2025-10-17 John Doe <john@example.com>
```

3. Copy the key ID (the part after the /):
```bash
AAAAAAAAAAAAAAAA
```

4. Configure Git to use this key for signing:

```bash
git config --global user.signingkey AAAAAAAAAAAAAAAA
git config --global commit.gpgsign true
git config --global gpg.program gpg
git config --global user.email "john@example.com"
```

5. Test signing a commit

```bash
git commit -S -m "test: signed commit"
```

6. Verify the signature:

```bash
git log --show-signature -1
```

You should see a **“Good signature”** message.
