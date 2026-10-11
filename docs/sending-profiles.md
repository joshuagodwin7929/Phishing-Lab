# Sending Profiles

I tested two delivery paths with this lab. Both are legitimate ways to run
Gophish safely which one to use depends on what you're actually testing.

## Option A MailHog (fully local, no real delivery)

Good for: testing templates, landing pages, and campaign mechanics without
anything touching the real internet. This is the default/safest option and
what the `docker-compose.yml` in this repo spins up out of the box.

**How it works:** MailHog runs as a container on the same Docker network as
Gophish and acts as a dummy SMTP server. It accepts mail and holds onto it,
but never actually relays anywhere.

**Gophish Sending Profile settings:**

| Field | Value |
|---|---|
| Name | `Local Test (MailHog)` |
| From | `IT Support <it-support@your-lab-domain.local>` |
| Host | `mailhog:1025` |
| Username / Password | leave blank |
| Ignore Certificate Errors | checked |

Since both containers share the `phishing-lab_default` Docker network, you
can reference MailHog by its service name (`mailhog`) instead of an IP.

**Viewing captured mail:**

```
http://<your-server-ip>:8025
```

(Must be `http://`, not `https://` — MailHog's UI doesn't serve TLS. See
[troubleshooting.md](troubleshooting.md) if your browser auto-upgrades it.)

## Option B — Mailtrap (real SMTP, sandboxed inbox)

Good for: testing the actual delivery path — real DNS resolution, real
outbound SMTP auth, real network egress — without risking a personal mailbox
getting flagged, and without the risk of real phishing mail landing in
someone's actual inbox.

**Setup:**

1. Create a free account at [mailtrap.io](https://mailtrap.io)
2. Create a Sandbox (Email Testing) project
3. Open the sandbox → **Integration** tab → select **SMTP**
4. Copy the Host, Port, Username, and Password it gives you

**Gophish Sending Profile settings:**

| Field | Value |
|---|---|
| Name | `Mailtrap Test` |
| From | `Display Name <user@example.com>` — must be valid `Name <email>` syntax |
| Host | `sandbox.smtp.mailtrap.io:2525` |
| Username | *(from your Mailtrap sandbox — not committed to this repo)* |
| Password | *(from your Mailtrap sandbox — not committed to this repo)* |
| Ignore Certificate Errors | checked |

**Viewing captured mail:** back in your Mailtrap sandbox's inbox view —
every send shows up there with full headers, spam-score analysis, and a
rendered preview, regardless of what recipient address you used.

> **A note on credentials:** Mailtrap sandbox credentials are tied to your
> personal account and are not included anywhere in this repo. If you fork
> or reuse this setup, generate your own sandbox and credentials rather than
> asking for mine.

## Which one should I use?

- Building/testing templates and landing pages → **MailHog**
- Validating the full delivery chain (DNS, SMTP auth, network egress) for a
  detection use case → **Mailtrap**
- Sending to a real inbox you control, for a fully end-to-end test → a real
  SMTP relay (Gmail/Outlook with an app password, or an internal mail
  server) — not covered in this repo, but the Sending Profile fields work
  the same way once you have real SMTP credentials.
