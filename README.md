# YubiKeys-cheatsheet

This repo contains tips on how to properly use your YubiKeys.

- Step by step YubiKey setup
- Tutorial: how to sign GitHub commits with YubiKey
- More to come

---


## 1️⃣ YubiKey set up

Before using your YubiKey to sign Git commits, you need to set up your devices.  

### Download required tools

- **Yubico Authenticator** (for OpenPGP interface & passkeys)  
  🔗 [Download here](https://www.yubico.com/products/yubico-authenticator/#h-download-yubico-authenticator)

- **YubiKey Manager** (for `ykman` command-line)  
  🔗 [Download release 5.8.0](https://github.com/Yubico/Yubikey-manager/releases/tag/5.8.0)

---

### Steps to repeat on all your YubiKeys

1. Verify your device is genuine:  
   🔗 [Yubico Genuine Check](https://www.yubico.com/genuine/)

2. Set a PIN on the YubiKey  
   - Use Yubico Authenticator → **Passkeys** tab  
   - PIN should be **4–6 digits**  
   - ⚠️ Recommended: use the **same PIN on all your YubiKeys** for simplicity

3. Enable user verification for FIDO (optional but recommended)  

```bash
ykman fido config toggle-always-uv
```

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
