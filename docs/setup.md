# Setup

This walks through standing up the lab from zero. I'm running this on an
Ubuntu server that already hosts my ELK SIEM lab, with Docker installed.
Nothing here is Ubuntu-specific other than the DNS troubleshooting section —
swap in whatever package manager you use if you're on a different distro.

## Prerequisites

- Docker + Docker Compose installed
- Outbound internet access (to pull images from Docker Hub)
- A text editor on the box (`nano` is fine)

## 1. Clone this repo

```bash
git clone <your-repo-url> phishing-lab
cd phishing-lab
```

## 2. Create your real config.json

Gophish needs a `config.json` that isn't tracked in git (see `.gitignore`).
Copy the example and adjust if needed — the defaults work fine for a lab:

```bash
cp config.json.example config.json
```

## 3. Bring the stack up

```bash
docker compose up -d
docker ps
```

You should see two containers running: `gophish` and `mailhog`.

If the image pull hangs or fails with a DNS-looking error, see
[troubleshooting.md](troubleshooting.md) — I hit this myself and it's almost
always a local DNS resolution issue, not a Docker problem.

## 4. Grab the admin password

Gophish generates a random admin password on first run and prints it once to
the logs. Grab it before it scrolls away:

```bash
docker compose logs gophish | grep -i password
```

## 5. Log in

Open a browser to:

```
https://<your-server-ip>:3333
```

Accept the self-signed certificate warning, then log in with:

- **Username:** `admin`
- **Password:** the one from step 4

Change this password immediately under **Account Settings** — the generated
one only prints once and lives in your shell history/logs until you clear it.

## 6. Set up a Sending Profile

Two options, both documented in [sending-profiles.md](sending-profiles.md):

- **MailHog** — fully local, nothing leaves your network. Good default for
  lab testing.
- **Mailtrap** — real SMTP delivery over the internet to a sandboxed inbox.
  Good for testing the full delivery path without any risk of spamming a
  real target or getting a personal account flagged.

## What's next

From here it's the standard Gophish flow: Email Template → Landing Page →
Users & Groups → Campaign. This repo doesn't cover campaign content since
that's specific to whatever detection scenario you're testing — see the
[Gophish docs](https://docs.getgophish.com/) for that part.
