# Troubleshooting

Real issues I actually hit while building this lab, kept here so future-me
(or anyone else) doesn't have to re-diagnose them from scratch.

## "failed to resolve reference ... dial tcp: lookup registry-1.docker.io"

**Symptom:** `docker compose up -d` fails trying to pull the image, with a
DNS lookup timeout buried in the error.

**Cause:** This isn't a Docker problem — it's the host's DNS resolver
failing. My server's `systemd-resolved` was pointed at an internal lab DNS
server (my AD lab's domain controller) that has no route to resolve public
internet domains like `registry-1.docker.io`.

**Diagnose it:**

```bash
ping -c 3 8.8.8.8          # confirms basic network connectivity works
nslookup registry-1.docker.io   # confirms DNS specifically is broken
resolvectl status          # shows which DNS server is actually in use
```

If ping works but nslookup times out or returns SERVFAIL, it's DNS, not
networking.

**Fix (temporary, until reboot):**

```bash
sudo resolvectl dns <your-interface> 8.8.8.8 8.8.4.4
sudo resolvectl flush-caches
```

**Fix (Docker-specific, in case host DNS can't be changed):**

Edit `/etc/docker/daemon.json`:

```json
{
  "dns": ["8.8.8.8", "8.8.4.4"]
}
```

Then `sudo systemctl restart docker`.

**Permanent fix:** add the DNS servers to your netplan config so it survives
a reboot (`/etc/netplan/*.yaml` → `nameservers: addresses:`), then
`sudo netplan apply`. If you also rely on an internal DNS server for local
hostnames (like an AD lab domain), keep that as a secondary resolver rather
than replacing it outright.

## "unexpected end of JSON input" — Gophish container stuck restarting

**Symptom:** `docker compose up -d` succeeds, but `docker ps` shows the
`gophish` container cycling through `Restarting (1)`. Logs show:

```
level=fatal msg="unexpected end of JSON input"
```

**Cause:** The compose file mounts `./config.json` into the container, but
if that file doesn't exist on the host yet, Docker silently creates an
**empty file** (or sometimes an empty directory) at that path instead of
failing. Gophish then tries to parse an empty file as JSON and crashes.

**Fix:**

```bash
docker compose down
rm -rf config.json          # clear whatever Docker created
docker run --rm --entrypoint cat gophish/gophish /opt/gophish/config.json > config.json
cat config.json             # confirm it actually has content now
docker compose up -d
```

This is also exactly why `config.json.example` exists in this repo — copy
it manually instead of letting Docker improvise one for you.

## Lost the admin password after changing it

**Symptom:** Changed the admin password, then forgot it. Gophish has no
"forgot password" email flow — there's no SMTP configured for that by
default, and even if there were, it's not a feature it ships with.

**Fix:** reset the password hash directly in the SQLite database. This
works because I have root on the host and full access to the container's
volume — it is **not** something that generalizes to resetting someone
else's account on a system you don't control (e.g. a real email provider).

```bash
# 1. Copy the live DB out of the container
docker cp gophish:/opt/gophish/gophish.db ./gophish.db.bak

# 2. Generate a new bcrypt hash for your new password
python3 -c "import bcrypt; print(bcrypt.hashpw(b'YourNewPassword', bcrypt.gensalt()).decode())"

# 3. Write the new hash into the users table
sqlite3 gophish.db.bak "UPDATE users SET hash='<hash-from-step-2>', password_change_required=0, account_locked=0 WHERE username='admin';"

# 4. Push the modified DB back into the container
docker stop gophish
docker cp gophish.db.bak gophish:/opt/gophish/gophish.db
docker start gophish
```

Note: the column is `hash`, not `password` — check the schema first with
`sqlite3 gophish.db.bak ".schema users"` if you're on a different Gophish
version and the column names have changed.

## MailHog web UI gives ERR_SSL_PROTOCOL_ERROR

**Symptom:** browsing to the MailHog UI throws an SSL error.

**Cause:** the browser auto-upgraded the request to `https://`, but MailHog's
web UI only serves plain HTTP.

**Fix:** type the full `http://` prefix explicitly rather than just the
IP:port, or open it in a private/incognito window to avoid a cached HTTPS
redirect.

## Gophish rejects the Sending Profile "From" field

**Symptom:** `Invalid SMTP From address because it is not an email address`
when sending a test email, even though the field looks fine.

**Cause:** Gophish validates this field strictly against `Name <email>`
syntax. In my case the fastest fix was retyping the field manually instead
of pasting it, in case of a stray invisible character — but if you hit this,
first just confirm the field is literally in the form:

```
Display Name <user@domain.tld>
```

A bare display name with no angle-bracket email, or a malformed email inside
the brackets, will trigger this every time.
