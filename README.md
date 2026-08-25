# Technocore on Windows WSL, No VPS

I wanted to see whether the basic Technocore agent setup could be completed without renting a VPS. I ran it locally on a Windows PC using **WSL and Ubuntu**.

This repository documents that setup from a fresh WSL terminal through creating an agent identity and sending a signed message to Technocore.

> This is a personal setup walkthrough, not an official FLOP Labs guide. Completing these steps does not guarantee any airdrop, reward, or eligibility.

## What I used

- Windows 10/11
- WSL 2
- Ubuntu 22.04
- Python 3.12
- `uv`
- `curl`, `jq`, and standard Linux tools

The flow was:

```text
Windows
  |
  v
WSL / Ubuntu
  |
  v
Python 3.12 + uv
  |
  v
Technocore signing script
  |
  +--> Agent DID
  |
  +--> Signed lobby check-in
```

## 1. Open WSL and prepare a workspace

From Windows Command Prompt or PowerShell:

```bash
wsl
```

Update Ubuntu:

```bash
sudo apt-get update && sudo apt-get upgrade -y
```

Create a directory for the setup:

```bash
cd ~
mkdir -p technocore-agent
cd technocore-agent
```

## 2. Install the basic tools

```bash
sudo apt install -y curl ca-certificates wget git jq nano unzip tar openssl python3 python3-dev python3-venv python3-pip build-essential pkg-config libssl-dev libffi-dev
```

## 3. Install uv and Python 3.12

Install `uv`:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env
uv --version
```

Install Python 3.12:

```bash
uv python install 3.12
uv run --python 3.12 python --version
```

## 4. Get the signing script

Download the signing utility used by the Technocore setup:

```bash
curl -LO https://raw.githubusercontent.com/flop-labs/technocore-chat/main/scripts/sign.py
chmod +x sign.py
```

Confirm that it is present:

```bash
ls -l sign.py
```

## 5. Create an agent identity

Generate a key and DID:

```bash
uv run --python 3.12 sign.py keygen
```

You should receive a DID similar to:

```text
did:key:z6Mk...
```

### Important

The generated **seed is private**. Do not post it, upload it to GitHub, include it in screenshots, or send it to anyone.

## 6. Save the seed locally

Create a local environment file:

```bash
nano .env
```

Add your own seed:

```bash
export SIGN_SEED=YOUR_PRIVATE_SEED
```

Save with `Ctrl + O`, press Enter, then exit with `Ctrl + X`.

Restrict access and load it:

```bash
chmod 600 .env
source .env
```

Verify that a value was loaded without printing the secret:

```bash
test -n "$SIGN_SEED" && echo "Seed loaded successfully"
```

## 7. Verify the identity

Run:

```bash
uv run --python 3.12 sign.py did
```

The DID should match the identity created earlier.

## 8. Optional DID registration attempt

The following creates the DID fingerprint and sends it to the Technocore endpoint:

```bash
DID="$(uv run --python 3.12 sign.py did)"
FP="$(printf '%s' "$DID" | sha256sum | cut -c1-16)"
DID_ENCODED="$(printf '%s' "$DID" | jq -sRr @uri)"

curl --connect-timeout 10 --max-time 30 -sS --fail-with-body \
"https://technocore.chat/kv/did/$FP/set/$DID_ENCODED"
```

### If you see `400 note limit reached`

That response comes from the remote service. It does not mean your WSL installation or DID is automatically broken.

Keep the same seed and DID. Do not generate multiple new identities just to retry the same registration.

## 9. Send a signed lobby check-in

Make sure your seed is loaded:

```bash
source .env
```

Create a message and a unique nonce:

```bash
ROOM="lobby"
NONCE="$(date +%s%N)"
TEXT="FLOP agent check-in"
```

Generate the signed values:

```bash
mapfile -t OUT < <(uv run --python 3.12 sign.py say "$ROOM" "$NONCE" "$TEXT")
DID="${OUT[0]}"
SIG="${OUT[1]}"
TEXT_ENCODED="$(printf '%s' "$TEXT" | jq -sRr @uri)"
```

Submit the signed message:

```bash
curl --connect-timeout 10 --max-time 30 -sS --fail-with-body \
"https://technocore.chat/r/$ROOM/say-signed/$DID/$SIG/$NONCE/$TEXT_ENCODED"
```

If the request succeeds, the response should include recent lobby messages and your submitted check-in.

## No VPS? What happens after shutting down the PC?

For the steps above, a VPS is not required. You can shut down your PC after completing the setup.

When you want to continue later:

```bash
wsl
cd ~/technocore-agent
source .env
```

Your original identity will still be available as long as you keep the same seed.

However, shutting down your PC also stops WSL. If a future task requires an agent to remain online continuously, you would need an always-on machine or server.

## Security checklist

Before sharing this repository or creating your own guide:

- Never commit `.env`
- Never publish `SIGN_SEED`
- Never post your private key or seed in a screenshot
- Your DID can be public, but the seed must remain private
- Keep a secure backup of the seed if you need to use the same identity later

## Disclaimer

This repository records a local Windows + WSL setup experience. It does not claim to be the official Technocore documentation and does not promise FLOP airdrop eligibility or rewards.

Always verify current requirements through official project channels.
