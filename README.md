# Phishing Lab

This is self-hosted phishing simulation lab built with [Gophish](https://getgophish.com/),
running in my Docker, with two different email delivery paths for testing, 
one fully local, one real SMTP over the internet to a sandboxed inbox.

I'm building this as part of a broader SOC / detection-engineering
portfolio. The goal isn't just "stand up Gophish", it's having a
repeatable, documented initial-access vector I can run campaigns from and
validate detection coverage against, alongside my other lab projects (SIEM
lab, Sigma rule detection-as-code, purple-team automation).

> **Authorized use only.** This lab is for testing against infrastructure I
> own and control (my own AD lab, my own test inboxes/sandboxes). It is not
> configured or intended to send phishing emails to real, unconsenting
> targets. If you reuse this setup, that's on you to keep in scope.

## What's in here

```
phishing-lab/
├── docker-compose.yml      # gophish + mailhog containers
├── config.json.example     # Gophish config template (copy to config.json)
├── .gitignore               # keeps real config, DBs, and certs out of git
└── docs/
    ├── setup.md             # step-by-step install walkthrough
    ├── troubleshooting.md   # real issues I hit, and how I fixed them
    └── sending-profiles.md  # MailHog (local) vs Mailtrap (real SMTP) setup
```

## Architecture

```
                 ┌─────────────────────────┐
                 │   Gophish (container)    │
                 │  admin UI → :3333         │
                 │  landing pages → :8080/443│
                 └────────────┬──────────────┘
                              │
                 SMTP (sending profile)
                              │
              ┌───────────────┴────────────────┐
              │                                 │
   ┌──────────▼──────────┐          ┌───────────▼────────────┐
   │  MailHog (container)│          │   Mailtrap (external)   │
   │  local only, :1025   │          │  real SMTP, sandboxed    │
   │  view at :8025        │          │  inbox, nothing delivered│
   └──────────────────────┘          │  to a real mailbox       │
                                       └──────────────────────────┘
```

Both Gophish and MailHog run on the same host via Docker Compose. Mailtrap
is an external service — no extra container needed, just a Sending Profile
pointed at Mailtrap's SMTP credentials.

## Quick start

```bash
git clone <your-repo-url> phishing-lab
cd phishing-lab
cp config.json.example config.json
docker compose up -d
docker compose logs gophish | grep -i password
```

Then open `https://<your-server-ip>:3333`, log in with `admin` and the
password from the logs, and change it immediately.

Full walkthrough: [docs/setup.md](docs/setup.md)

## Why two sending profiles?

- **MailHog** — everything stays on the box. No DNS lookups out, no real
  SMTP handshake, nothing that could ever land in a real inbox. This is the
  safe default for iterating on templates and landing pages.
- **Mailtrap** — exercises the real path: outbound DNS resolution, a real
  authenticated SMTP session over the internet, delivery to an actual
  (sandboxed) mailbox. Useful when I want to validate the full chain —
  including anything my SIEM/detection stack might see on the wire — without
  any risk of spamming a real person or getting a personal account flagged.

Details on both: [docs/sending-profiles.md](docs/sending-profiles.md)

## Lessons learned

The build wasn't totally frictionless, and I kept notes on everything that
went wrong rather than just the happy path:

- A broken host DNS resolver (pointed at an internal-only DNS server) blocked
  Docker from pulling images entirely — nothing to do with Docker itself.
- Docker silently creates an empty file when a bind-mount source doesn't
  exist yet, which crashed Gophish on startup with a cryptic JSON error.
- Recovering a forgotten Gophish admin password meant going straight to the
  SQLite database and writing a new bcrypt hash by hand, since there's no
  password-reset email flow out of the box.

Full writeups, with the actual commands and output: [docs/troubleshooting.md](docs/troubleshooting.md)

## Roadmap

- [ ] Wire campaign results into the ELK SIEM lab for detection validation
- [ ] Build a Sigma rule to detect Gophish's default email headers/landing
      page fingerprints, then iterate once a real template is in use
- [ ] Chain into the purple-team automation project as a phishing-based
      initial-access vector

## Credentials and secrets

Nothing in this repo contains real credentials, IPs, or passwords 
`config.json`, the Gophish database, and any TLS certs are excluded via
`.gitignore`. If you fork this, generate your own Mailtrap sandbox and your
own `config.json` rather than expecting either to be included here.
