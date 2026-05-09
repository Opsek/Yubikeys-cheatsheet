# YubiKeys-cheatsheet

This repo contains tips on how to properly use your YubiKeys.

- 1️⃣ Step by step YubiKey setup
- 2️⃣ Tutorial on how to sign GitHub commits with YubiKey
- More to come 😉

## YubiKey Interfaces / Applications

| Interface / App | What it is | Phishing-resistant | How you use it | Where / When to use it | Concrete examples |
|-----------------|------------|--------------------|---------------|------------------------|-------------------|
| **FIDO2** | Modern passwordless and 2FA authentication standard (WebAuthn + CTAP2) | **Yes** | Touch the YubiKey when prompted by the browser or OS | Best choice for web logins and OS authentication | Register passkeys on Google, GitHub, Microsoft, AWS; passwordless login or strong 2FA |
| **FIDO U2F** | Legacy FIDO standard (CTAP1), requires password first | **Yes** | Enter username/password, then touch the YubiKey | Older services that don't support full FIDO2 | GitHub legacy security key login; older VPNs, NAS devices |
| **OATH (TOTP / HOTP)** | One-time password generator stored on the YubiKey | **No** | Use **Yubico Authenticator** to read 6- or 8-digit codes | When a service only supports OTP-based 2FA | Generate TOTP for GitHub, Google, servers; fallback if passkeys/security keys aren't supported |
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

> 🍎 **macOS users:** Run all `ykman` commands below from the **built-in macOS Terminal app** (`/Applications/Utilities/Terminal.app`).
> Third-party terminals such as **iTerm2**, **Warp**, **Alacritty**, **Kitty**, **Hyper**, or VS Code's integrated terminal can interfere with USB/smart-card access and may cause `ykman` to hang, fail to detect the YubiKey, or silently swallow PIN prompts.
> If a `ykman` command misbehaves, switch to Terminal.app and try again before debugging anything else.

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
4. Enable **2-Step Verification (2FA)** if it's not already enabled
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

- Installed **GnuPG (GPG)**: on macOS: `brew install gnupg`  
  GitHub: https://github.com/gpg/gnupg

- Installed **YubiKey Manager** (for `ykman`): on macOS: `brew install ykman`  
  GitHub: https://github.com/Yubico/YubiKey-manager

> 🍎 **macOS users — important:** Run every `ykman` command in this section from the **built-in macOS Terminal app** (`/Applications/Utilities/Terminal.app`), not from iTerm2, Warp, Alacritty, Kitty, Hyper, or VS Code's integrated terminal.
> Third-party terminals frequently cause `ykman` to hang, fail to see the YubiKey, or silently fail PIN prompts because of how they sandbox USB / smart-card access. If a `ykman` command behaves strangely, the first thing to try is running it from Terminal.app.

### 🔐 YubiKey GPG (ed25519) → Signed Git Commit

#### 1) Configure GPG pinentry

GPG needs a **pinentry program** to securely prompt you for your PIN. Pick the path that matches your setup:

##### 🍎 macOS (recommended: native GUI popup)

Install `pinentry-mac` so GPG can show a native macOS popup when your PIN is needed:

```bash
brew install pinentry-mac

mkdir -p ~/.gnupg && chmod 700 ~/.gnupg

# Tell gpg-agent to use pinentry-mac
echo "pinentry-program $(brew --prefix)/bin/pinentry-mac" >> ~/.gnupg/gpg-agent.conf

# Reload the agent so the change takes effect
gpgconf --kill scdaemon
gpgconf --kill gpg-agent
```

> ⚠️ **Do NOT add `pinentry-mode loopback` to `gpg.conf` on macOS.** It forces GPG to prompt inside the terminal and **breaks PIN entry inside `gpg --card-edit`** on recent GnuPG versions (2.4+/2.5+): keypresses are silently swallowed and the prompt appears frozen. If you hit this, jump to [🛠️ Troubleshooting](#%EF%B8%8F-troubleshooting-pin-prompt-doesnt-respond).

##### 🐧 Linux (desktop)

Most distros install `pinentry-gtk` or `pinentry-qt` automatically with GPG. Nothing to do: just make sure `gpg-agent` is running.

##### 🖥️ Headless / SSH session (no GUI available)

**Only use this if you truly have no GUI** (e.g., signing over SSH on a remote box). It forces terminal-only PIN prompts:

```bash
mkdir -p ~/.gnupg && chmod 700 ~/.gnupg

cat > ~/.gnupg/gpg.conf <<'EOF'
use-agent
pinentry-mode loopback
EOF

grep -q '^allow-loopback-pinentry' ~/.gnupg/gpg-agent.conf 2>/dev/null || \
  echo 'allow-loopback-pinentry' >> ~/.gnupg/gpg-agent.conf

gpgconf --kill gpg-agent
```

#### 2) (Optional) Clean-slate the YubiKey OpenPGP applet

> ⚠️ Destroys any existing GPG keys on the key.
>
> 🍎 **macOS:** run these from **Terminal.app**, not iTerm2/Warp/etc. (see note at the top of this section).

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

> ### 🔑 About the "passphrase" prompt inside `gpg --card-edit`
>
> When you're in the `gpg --card-edit` interface, the **"passphrase"** being requested is actually the **PIN for your YubiKey's OpenPGP applet**: not your SSH passphrase or your GPG key passphrase.
>
> The OpenPGP applet uses **two separate PINs**:
>
> - **User PIN** (default: `123456`): used for day-to-day operations like signing, decrypting, and authenticating.
> - **Admin PIN** (default: `12345678`): used for administrative changes like modifying key attributes, generating keys on the card, or resetting the User PIN.
>
> Since `key-attr` is an **admin-level operation**, the prompt is asking for the **Admin PIN**. If you've never changed it, enter `12345678`. You'll also be prompted for the **User PIN** (`123456` by default) during key generation. You should change both PINs (see step 2 above, or use `passwd` inside `gpg --card-edit`) before loading any real keys onto the device.
>
> 💡 On macOS, the PIN prompt appears as a **`pinentry-mac` popup window**: not inside the terminal. If no popup appears and typing in the terminal does nothing, see [🛠️ Troubleshooting](#%EF%B8%8F-troubleshooting-pin-prompt-doesnt-respond).

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
> 🍎 **macOS:** run these two commands from **Terminal.app**, not iTerm2 / Warp / Alacritty / VS Code's integrated terminal. Third-party terminals can cause `ykman` to hang or fail to see the YubiKey (see the note at the top of this section).
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

#### 5) Export your **Primary Public Key** and add to GitHub (for commit verification)

You need to export the **Primary Public Key** (a.k.a. the **master / primary key**), **not** one of the subkeys. GitHub matches commit signatures against the primary key listed on your account.

##### Find your Primary Public Key ID

Run:

```bash
gpg --list-keys --keyid-format=long "you@example.com"
```

You'll see something like:

```
pub   ed25519/ABCDEF1234567890 2025-10-17 [SC]
      F1E2D3C4B5A697887766554433221100AABBCCDD
uid   [ultimate] John Doe <john@example.com>
sub   cv25519/1111222233334444 2025-10-17 [E]
sub   ed25519/5555666677778888 2025-10-17 [A]
```

The **Primary Public Key** is the one on the line starting with `pub` — in this example, `ABCDEF1234567890`. The lines starting with `sub` are subkeys (encryption `[E]` and authentication `[A]`); **don't use those**.

> 💡 You can also use the **full 40-character fingerprint** (the line directly under `pub`) instead of the short ID — it's unambiguous and recommended.

##### Export it

Replace `<PrimaryPublicKey>` with the ID (or fingerprint) you just found:

```bash
gpg --armor --export-options export-minimal --export <PrimaryPublicKey> > ~/YubiKey-gpg-public.asc
cat ~/YubiKey-gpg-public.asc
```

The `--export-options export-minimal` flag strips unnecessary signatures and produces a smaller, cleaner public key block — easier for GitHub to ingest.

➡️ copy everything (including the `-----BEGIN PGP PUBLIC KEY BLOCK-----` and `-----END PGP PUBLIC KEY BLOCK-----` lines) and paste at: https://github.com/settings/keys → **"New GPG key"**.

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

You should see a **"Good signature"** message.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## 🛠️ Troubleshooting: PIN prompt doesn't respond

**Symptom:** You run `gpg --card-edit`, type `admin` → `key-attr`, and when GPG asks for the PIN (or "passphrase"), **nothing happens when you type**: no characters, no dots, no error, no popup. Pressing Enter does nothing and the terminal appears frozen.

**Cause (macOS):** Your `~/.gnupg/gpg.conf` contains `pinentry-mode loopback`, which tells GPG to prompt inside the terminal instead of using a pinentry program. On recent GnuPG versions (2.4+ / 2.5+), **this terminal prompt does not work inside `gpg --card-edit`'s interactive REPL**: keypresses are swallowed and you're stuck.

**Fix:** Disable loopback mode and use `pinentry-mac` instead (the native macOS GUI popup).

### Step-by-step fix (macOS)

1. **Confirm which `gpg` you're running**: it should be Homebrew's, not GPG Suite's:
   ```bash
   which gpg
   gpg --version
   ```
   Expected: `/opt/homebrew/bin/gpg` (Apple Silicon) or `/usr/local/bin/gpg` (Intel), version ≥ 2.4. If you see `/usr/local/MacGPG2/...`, uninstall GPG Suite: it conflicts with Homebrew's `gpg` and is a frequent cause of pinentry weirdness.

2. **Install `pinentry-mac`:**
   ```bash
   brew install pinentry-mac
   which pinentry-mac   # should print /opt/homebrew/bin/pinentry-mac
   ```

3. **Wire it into `gpg-agent.conf`** (skip if already done):
   ```bash
   echo "pinentry-program $(brew --prefix)/bin/pinentry-mac" >> ~/.gnupg/gpg-agent.conf
   ```

4. **Check your current config for the loopback lines:**
   ```bash
   cat ~/.gnupg/gpg.conf
   cat ~/.gnupg/gpg-agent.conf
   ```
   If you see `pinentry-mode loopback` in `gpg.conf` or `allow-loopback-pinentry` in `gpg-agent.conf`, **that's the problem.**

5. **Disable loopback mode** (back up first):
   ```bash
   cp ~/.gnupg/gpg.conf ~/.gnupg/gpg.conf.bak
   sed -i '' 's/^pinentry-mode loopback/#pinentry-mode loopback/' ~/.gnupg/gpg.conf
   sed -i '' 's/^allow-loopback-pinentry/#allow-loopback-pinentry/' ~/.gnupg/gpg-agent.conf
   ```

6. **Fully restart the agent and smartcard daemon:**
   ```bash
   gpgconf --kill scdaemon
   gpgconf --kill gpg-agent
   gpgconf --launch gpg-agent
   ```

7. **Unplug and replug the YubiKey**, then retry:
   ```bash
   gpg --card-edit
   ```
   When GPG asks for the PIN, a **`pinentry-mac` GUI popup** will appear. Enter your PIN there: not in the terminal.

### Still stuck?

- **Test `pinentry-mac` directly** to confirm it runs at all:
  ```bash
  /opt/homebrew/bin/pinentry-mac
  ```
  Then type `GETPIN` + Enter. A popup should appear. Type `BYE` + Enter to exit. If no popup appears, macOS is blocking `pinentry-mac` itself: check **System Settings → Privacy & Security** for a blocked-app notification.

- **Check for leftover GPG Suite** hijacking the socket:
  ```bash
  ls /Library/LaunchAgents/ 2>/dev/null | grep -i gpg
  ls ~/Library/LaunchAgents/ 2>/dev/null | grep -i gpg
  ls /usr/local/MacGPG2/ 2>/dev/null
  ```
  If any of these exist, remove the LaunchAgents and uninstall GPG Suite completely, then repeat the fix.

- **Try from macOS Terminal.app** if you're using a third-party terminal (iTerm2, Warp, Alacritty, Kitty, Hyper, VS Code integrated terminal). These can interfere with USB / smart-card access and cause `ykman` and `gpg` to behave unpredictably with the YubiKey.

- **Duplicate `pinentry-program` lines** in `gpg-agent.conf` can also cause issues. Open the file and make sure there's only one.
