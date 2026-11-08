# Running your own Klarinotte server

> This repository holds the guide, the licence and example files for the
> self-hosted Klarinotte server. The server is the Docker image
> `ghcr.io/klarinotte/klarinotte-server` (amd64 and arm64); its source is not published here.
> Problems and questions: https://github.com/klarinotte/klarinotte-feedback/issues.

Klarinotte keeps your notes on your devices and syncs them, end-to-end
encrypted, through a server. This guide is for running that server yourself:
for your family, friends or a small team, on a small server or a Raspberry Pi.
You get the same app in the browser, served by your server, and the same sync.

- **Free to run** for yourself and your own people, at any size, under the
  [Klarinotte Server Licence](https://github.com/klarinotte/klarinotte-server/blob/main/LICENSE)
  (also `/klarinotte-server-licence.txt` on your server). Hosting it for other
  people or organisations needs an agreement with us:
  [contact us](https://www.klarinotte.com/contact).
- **No email.** Your server never sends any: you give people their sign-up
  codes and, if they forget their password, a reset link.
- **You are the data controller** for the accounts on your server (emails,
  sign-in records). Notes and attachments are encrypted on people's devices;
  you can't read them.

## 1. What you need

- A machine with **Docker**: a Linux computer or VPS, or a Raspberry Pi with
  a 64-bit OS (Raspberry Pi OS or Debian) and 2 GB+ RAM (tested on a
  Raspberry Pi 4). On a Pi, keep Docker's data on an **SSD** (recommended),
  not the SD card: the database writes constantly, and SD cards wear out and
  corrupt data on power loss.
- An **HTTPS address** your devices can reach. The app needs HTTPS: browsers
  allow its encryption, Web Locks and storage only on secure pages. Plain
  `http://` on a LAN address does not work. Pick one of four ways (§2):

  | | Who can reach it | You need |
  |---|---|---|
  | **A. Tailscale** | Only devices on your tailnet | A Tailscale account; the Tailscale app on every device |
  | **B. Cloudflare Tunnel** | Anyone on the internet | A domain on Cloudflare; no open ports |
  | **C. Tailscale Funnel** | Anyone on the internet | A Tailscale account; Tailscale on the server only; no domain, no open ports |
  | **D. Domain + port forwarding** | Anyone on the internet | A domain or a free dynamic-DNS name, ports 80 and 443 forwarded to the machine |

- For a first try on your own computer, without any of that:
  `docker run --rm --name klarinotte-try -p 8080:8080 -v klarinotte_try:/data ghcr.io/klarinotte/klarinotte-server`,
  then, in another terminal, a sign-up code for yourself:
  `docker exec klarinotte-try klarinotte-admin register you@example.com`
  (§3), and open `http://localhost:8080`.

## 2. Install

Each way below has its own compose file in the
[repository](https://github.com/klarinotte/klarinotte-server): `tailscale/`
(A and C), `cloudflare-tunnel/` (B) and `caddy/` (D). The steps download it
into a folder named `klarinotte`, and every command in this guide runs there.
Keep that name, even when you switch to another way later: Docker names the
data volume after the folder (`klarinotte_klarinotte_data`), so a folder with
another name starts with an empty server.

On first start the server creates its database and its keys (`secrets.json`)
in the `klarinotte_data` volume. Whichever way below you choose, check after
starting that it is ready:

```sh
docker compose logs klarinotte      # "Klarinotte on https://… " when it's ready
```

A server that can't start (a wrong setting, a data folder it can't write)
restarts over and over, and `docker compose ps` still shows it as "Up": the
logs say why ("could not start: …").

Prefer a folder on the host to a named volume (`./data:/data`)? Create it first
and give it to the container's user, or the server can't write there:
`mkdir data && sudo chown 1000:1000 data && chmod 700 data`.

**Never publish the server's port (8080) to the network.** With
`TRUST_PROXY=1` the server believes the client address the proxy passes on; a
request that reached 8080 directly could pretend to come from anywhere (and
dodge the sign-in rate limits). Only Caddy, `tailscale serve` / `tailscale
funnel` or `cloudflared` may reach it.

### A. Tailscale (your own devices only)

Nothing is opened to the internet; only devices signed in to your tailnet reach
the server.

1. Install Tailscale on the server and sign it in:

   ```sh
   curl -fsSL https://tailscale.com/install.sh | sh
   sudo tailscale up                   # open the link it prints
   ```

2. In the Tailscale admin console, on the **DNS** page: enable **MagicDNS**,
   then under **HTTPS Certificates** select **Enable HTTPS**. Machine names end
   up in public certificate logs: rename the machine first if its name says
   too much. You don't need `tailscale cert` (Tailscale's docs mention it):
   `tailscale serve` gets and renews the certificate itself.
3. Forward HTTPS to the server, in the background (it stays on after a
   reboot):

   ```sh
   sudo tailscale serve --bg 8080
   sudo tailscale serve status         # shows https://<machine>.<tailnet>.ts.net
   ```

4. On the server, download the
   [compose file for Tailscale](https://github.com/klarinotte/klarinotte-server/blob/main/tailscale/docker-compose.yml) and,
   in any editor, set `PUBLIC_URL` in it to the address `tailscale serve
   status` showed:

   ```sh
   mkdir -p klarinotte && cd klarinotte
   curl -fsSLO https://raw.githubusercontent.com/klarinotte/klarinotte-server/main/tailscale/docker-compose.yml
   nano docker-compose.yml             # PUBLIC_URL: https://<machine>.<tailnet>.ts.net
   ```

   It publishes the port on `127.0.0.1` only, where `tailscale serve` reaches
   it and the rest of your network doesn't.

5. Start it: `docker compose up -d`.

Every device that uses the server needs the Tailscale app, signed in. For
family or friends with their own Tailscale account, share the machine with
them (admin console → **Machines** → the machine's menu → **Share**); they open
the same `https://<machine>.<tailnet>.ts.net` address.

To stop it: `sudo tailscale serve reset` (the server keeps running; nothing
reaches it until you run step 3 again). `tailscale serve status` shows what is
on.

### B. Cloudflare Tunnel (reachable anywhere, no open ports)

`cloudflared` connects out to Cloudflare, and Cloudflare forwards requests for
your name through it. Your domain must be on Cloudflare (its DNS managed
there).

1. In the Cloudflare dashboard, go to **Networking** → **Tunnels** →
   **Create a tunnel**, give it a name, select **Create Tunnel** and choose
   **Docker**. From the command it shows, copy only the token (the long value
   after `--token`).
2. On the server, download the
   [compose file for a Cloudflare Tunnel](https://github.com/klarinotte/klarinotte-server/blob/main/cloudflare-tunnel/docker-compose.yml),
   and, in any editor, set `PUBLIC_URL` in it to your domain and put the
   token in a file named `.env` next to it (keep it private: it runs your
   tunnel):

   ```sh
   mkdir -p klarinotte && cd klarinotte
   curl -fsSLO https://raw.githubusercontent.com/klarinotte/klarinotte-server/main/cloudflare-tunnel/docker-compose.yml
   nano docker-compose.yml             # PUBLIC_URL: https://<your-domain>
   (umask 077 && nano .env)            # one line: TUNNEL_TOKEN=<token>
   ```

3. Start it: `docker compose up -d`. Back in the dashboard, wait for the
   tunnel to connect and select **Continue**.
4. Open the tunnel (**Networking** → **Tunnels** → its name; not
   **Networking** → **Routes**, which is for private networks). On its
   **Routes** tab, **Add route** → **Published application**:
   your subdomain and domain (as in `PUBLIC_URL`), **Service URL**
   `http://klarinotte:8080`.

To take the server off the internet, `docker compose stop cloudflared`
(`docker compose start cloudflared` brings it back). To end the tunnel for
good, delete it in the dashboard and remove the `cloudflared` service.

Two things to know:

- **Cloudflare decrypts the traffic at its edge.** It sees what your server
  sees: addresses, times, sizes, account emails and reset links in URLs — not
  the content of notes and attachments, which are encrypted on the devices.
  Because it also delivers the web app to the browser, it could deliver an
  altered one, as your server could. If you don't want to trust a third party
  with that, use A, C or D, where the encryption ends on your machine. Whoever
  runs your server's name (Tailscale for `ts.net`, a dynamic-DNS service) could
  still get a certificate for it, but it would show in public certificate
  logs; only your own domain avoids that. Cloudflare does add a script of its
  own (bot detection) to the page when the domain has it on; the app's
  security policy blocks it, which shows as one error in the browser console.
- **Uploads are limited to 100 MB per request** on Cloudflare's Free and Pro
  plans. The app allows attachments up to 100 MB, so one close to that size
  can be refused.

### C. Tailscale Funnel (reachable anywhere, no domain, no open ports)

Funnel publishes the server's Tailscale name on the internet. Only the server
needs Tailscale: phones and computers open the address in a browser, with no
Tailscale app. The encryption ends on your server; Tailscale's relays pass the
traffic on without decrypting it.

1. Do steps 1 and 2 of A (install Tailscale, enable MagicDNS and HTTPS).
2. Publish the server, in the background (it stays on after a reboot):

   ```sh
   sudo tailscale funnel --bg 8080     # open the link it prints, if any
   sudo tailscale funnel status        # shows https://<machine>.<tailnet>.ts.net
   ```

   The first time, it prints a link: open it and approve adding Funnel to your
   tailnet's access policy. The public name can take up to 10 minutes to work.
3. Download the same compose file as A and set `PUBLIC_URL` to your `ts.net`
   address; change nothing else (Funnel passes the client address on too, and
   reaches the port on `127.0.0.1`):

   ```sh
   mkdir -p klarinotte && cd klarinotte
   curl -fsSLO https://raw.githubusercontent.com/klarinotte/klarinotte-server/main/tailscale/docker-compose.yml
   nano docker-compose.yml             # PUBLIC_URL: https://<machine>.<tailnet>.ts.net
   ```

   Start it: `docker compose up -d`.

Already using A? Run step 2 only: it replaces `tailscale serve`, and your
tailnet devices keep the same address.

Two things to know:

- **The address is public.** Anyone can open the sign-in page; only people you
  register can create an account (§3). The machine name is now in public DNS
  too.
- **Funnel's bandwidth is limited** by Tailscale (the limits are not
  published), and the address is always a `ts.net` name. For heavy use or your
  own name, use B or D.

To stop publishing but keep the server for your tailnet devices, run
`sudo tailscale serve --bg 8080` (it replaces Funnel). The command Funnel
suggests, `sudo tailscale funnel --https=443 off`, and `sudo tailscale funnel
reset` turn off tailnet access too.

### D. Domain + port forwarding (Caddy)

Not yet tested by us; A, B and C were.

1. **A name for your server.** Your own domain with a DNS record pointing at
   your public IP address, or a free dynamic-DNS name (from a service such as
   DuckDNS, or your router's built-in one). At home your IP address usually
   changes now and then: dynamic DNS keeps the name pointing at it.
2. **Forward ports 80 and 443** (TCP) on your router to the machine. Give the
   machine a fixed LAN address first (a DHCP reservation in the router).
   Caddy needs both ports to get the certificate.
3. On the server, download the
   [compose file for Caddy](https://github.com/klarinotte/klarinotte-server/blob/main/caddy/docker-compose.yml) and its
   [`Caddyfile`](https://github.com/klarinotte/klarinotte-server/blob/main/caddy/Caddyfile):

   ```sh
   mkdir -p klarinotte && cd klarinotte
   curl -fsSLO https://raw.githubusercontent.com/klarinotte/klarinotte-server/main/caddy/docker-compose.yml
   curl -fsSLO https://raw.githubusercontent.com/klarinotte/klarinotte-server/main/caddy/Caddyfile
   ```

4. In both, in any editor, replace `<your-domain>` with your name.
5. Start it: `docker compose up -d`. Then check Caddy too:
   `docker compose logs caddy` should show it obtaining a certificate for your
   name (a Caddyfile left unedited only shows up there).

Caddy gets and renews the HTTPS certificate. If your provider shares one
public address between customers (CGNAT: the router's WAN address differs from
what "what is my IP" sites show), forwarding can't work: use A, B or C.

## 3. Accounts

There is no admin page; you use a command on the server:

```sh
docker compose exec klarinotte klarinotte-admin <command>
```

| Command | What it does |
|---|---|
| `register <email>` | A 6-digit code for that email, valid up to 7 days (`--days N`). They sign up in the app with exactly this email and enter the code. Start with your own account. |
| `reset-link <email>` | A one-time password-reset link (1 hour, `--hours N`) for someone who forgot their password. |
| `users` | Everyone's email, when they signed up, when they were last seen, storage used. |
| `delete-user <email> --yes` | Deletes an account and the notebooks it owns — for everyone they share them with — by running the daily cleanup now (its other work too; it prints what it did). Without `--yes` it only says what would go. |
| `backup` | A consistent copy of everything (§6). |
| `restore <backup> --yes` | Puts a backup back, with the server stopped (§7). |
| `models …` | Downloads the optional search and OCR models (§5). |

Things to tell the people you invite:

- **Their Recovery Key is the only way to keep their notes through a password
  reset.** The app shows it once at sign-up; they should store it safely.
  With a reset link but without the key, they can only reset the account,
  which deletes the notebooks they own from the server (their devices keep
  their copies).
- On a phone, they open your address in the browser and add it to the home
  screen (it installs as an app). The desktop app can't connect to your server
  yet.
- Sharing a notebook works by email address within your server. You vouch for
  each address when you register it: anyone you register as
  `alice@example.com` receives what is shared with that address.

## 4. Settings

Set them under `environment:` in the compose file, then
`docker compose up -d` again.

| Setting | |
|---|---|
| `PUBLIC_URL` | The address people open, e.g. `https://notes.example.com`. Required in practice: reset links use it. |
| `TRUST_PROXY` | `1` when every request comes through a reverse proxy (Caddy, `tailscale serve` / `funnel`, `cloudflared`: all pass the client address in `X-Forwarded-For`). |
| `TERMS_URL`, `PRIVACY_URL` | Optional pages of your own, shown on first start and in Settings. Set them **before** people start using the server: someone who already got past the first-start notice is not asked again. |
| `PORT`, `DATA_DIR` | Default `8080` and `/data`; you rarely change them. |

**Another reverse proxy instead of Caddy** (nginx, Traefik): forward to port
8080, keep `TRUST_PROXY=1`, make sure it passes the client address in
`X-Forwarded-For` (nginx: `proxy_set_header X-Forwarded-For
$proxy_add_x_forwarded_for;`), allow request bodies of 101 MB (nginx:
`client_max_body_size 101m;`), and don't publish port 8080.

**Proxy access logs** contain reset links in the URL (`/reset-password?token=…`).
A link works once and for an hour, but treat those logs as sensitive.

## 5. Optional: semantic search and OCR

Both are off on a new server: the app says so ("ask your administrator").
Download what you want once; the files stay in the data volume.

```sh
docker compose exec klarinotte klarinotte-admin models                 # what's installed
docker compose exec klarinotte klarinotte-admin models --search        # semantic search (~135 MB)
docker compose exec klarinotte klarinotte-admin models --ocr eng,deu   # OCR languages (~10–45 MB each)
```

OCR languages the app offers: `eng spa fra deu ita por ron rus ara fas chi_sim
chi_tra jpn kor hin ben tur vie pol ukr nld swe ces nor`.

This is the only time your server contacts the internet on its own:
`huggingface.co` (and its CDN `cdn-lfs.hf.co` / `*.hf.co`) for search,
`github.com` and `raw.githubusercontent.com` for OCR. Without internet access,
download the same files elsewhere into a folder laid out as
`Xenova/multilingual-e5-small/…` and `tessdata/<lang>.traineddata.gz`, then copy
it in as the server's user (so the server can read the files):

```sh
tar -C ./models -c . | docker compose exec -T klarinotte sh -c 'mkdir -p /data/models && tar -x -C /data/models'
```

## 6. Backups

```sh
docker compose exec klarinotte klarinotte-admin backup
```

It writes `/data/backups/backup-<time>/` while the server keeps running — the
database, `secrets.json` and every stored file (notes and attachments) the
database refers to — checks
that the copy is complete, and prints the command to copy it off the server:

```sh
docker compose cp klarinotte:/data/backups/backup-<time> ./
```

Then delete it from the server (`docker compose exec klarinotte rm -r
/data/backups/backup-<time>`): it shares files with the live data and keeps
deleted attachments on disk. Downloaded models aren't included (download them
again). A backup needs free disk space for one copy of the database (the
attachments are linked, not copied). A folder ending in `.partial` is an
interrupted backup: delete it.

- **A backup holds `secrets.json`, which lets whoever has it sign in as anyone
  on your server.** Store backups encrypted or somewhere only you can reach.
- **`docker compose down -v` deletes the data volume.** Never add `-v`.

## 7. Restoring

For when the server's data is lost or broken — not to undo a mistake. People's
devices keep their notes; a restore brings back the server's side (accounts,
sharing, keys) and makes every device sync again. Never restore by copying
files back: the devices would not know, and notes would go missing.

1. Stop the server: `docker compose stop klarinotte`.
2. If the backup is not on the server any more, copy it into the data volume:

   ```sh
   tar -C ./backup-<time> -c . | docker compose run --rm -T klarinotte sh -c 'mkdir -p /data/backups/backup-<time> && tar -x -C /data/backups/backup-<time>'
   ```

3. Restore (without `--yes` it only says what will happen):

   ```sh
   docker compose run --rm klarinotte klarinotte-admin restore backup-<time> --yes
   ```

4. Start it: `docker compose start klarinotte`.

It checks the backup first (complete, not damaged, not from a newer version)
and refuses while the server is running. The data it replaces is moved to
`/data/replaced-<time>/`, not deleted: delete that folder once everything
works. If it stops halfway, the server won't start until you run the same
command again.

**Then tell everyone:**

- Sign in again, on every device, with the password and Recovery Key **as they
  were at the backup** (the app says the account was reset and asks for a new
  password: use the one from then).
- **Anyone who reset their account or signed up after the backup: export your
  notes (Export Notebook in Settings, for each notebook) before signing in.** The app may offer to replace
  or delete this device's notes.
- What was deleted since the backup comes back — notes, notebooks, accounts.
  Delete them again (for accounts: `delete-user`).
- Removals of people from shared notebooks, password changes and account resets
  since the backup are undone: redo them now. A removed member is a member
  again until you do (for a notebook whose key was changed since, the owner
  sees them on the Resume list: remove them in Members before choosing
  Resume).
- Notebooks created since come back once each device reopens the app; shares
  made since show as private copies. Version history since the backup is gone.
- Sign-up codes and reset links must be issued again.
- An account deletion pending in the backup gets its 7 days again (the command
  names them); whoever had cancelled it since cancels it again when signing in.

The command lists the notebooks whose key was changed since the backup, with
their owner. On devices that already had the newer key, those notebooks pause
with a notice that the server's copy is older; until the owner resumes them,
those devices keep their notes but don't sync that notebook. The **owner**
opens the notebook on one device only, the one with the newest key, and
chooses **Resume**: it changes the key again, shares it with the people listed
(check that list: a member removed after the backup is back on it), and
uploads the notebook; the other devices that had the newer key follow at their
next sync. If none of the owner's devices shows the notice, the owner removes
the members whose devices do and invites them again. A device that never got
the newer key (offline since before it was changed) syncs that notebook as
usual until the owner's Resume, then pauses it with a notice that the key was
changed by someone who didn't hold the previous one, or that the server's copy
was altered. After a restore that is expected, not tampering. Don't press
Resume on that device: on the owner's own device, remove the notebook and
download it again; a member asks the owner to remove them and invite them
again. Edits made on such a device between the restore and the Resume don't
carry over, so export anything changed there first.
Every device uploads its notes again, so for about two days the server needs
room for roughly one more copy of the data per device.

## 8. Upgrades

```sh
docker compose pull && docker compose up -d
```

A new version upgrades the database on start, after saving a copy of it in
`/data/backups`. Upgrades go forward only: an older version refuses a database
a newer one has upgraded. Pin a version in the compose file
(`klarinotte-server:<version>`) to upgrade on purpose.

To hear about new versions, watch the
[releases](https://github.com/klarinotte/klarinotte-server/releases) (on
GitHub: **Watch** → **Custom** → **Releases**). Each one says what changed.

## 9. Limits

- One server instance; no clustering. Plenty for a family or a small team.
- No email, no billing, no admin web page.
- Everyone has the same, generous limits (100,000 items, 100 GB, 25 devices,
  3 years of note history per person). Disk size is the real limit: a full
  disk stops sync.
- The desktop app can't connect yet.

## 10. Help and problems

The self-hosted server is a beta, supported on a best-effort basis. Report
problems and ask questions in the
[klarinotte-feedback issues](https://github.com/klarinotte/klarinotte-feedback/issues);
say that you run your own server, and which version (Settings → General in
the app shows it).

Found a security problem? Don't post it there: tell us privately through the
[contact page](https://www.klarinotte.com/contact).

## Licences

The [Klarinotte Server Licence](https://github.com/klarinotte/klarinotte-server/blob/main/LICENSE), also
`/klarinotte-server-licence.txt` on your server.
Open-source components: `/third-party-licenses.txt` (the app) and
`/app/SERVER-NOTICES.txt` in the image (the server); Node.js and its components:
`/usr/local/LICENSE`. The base image's Debian packages keep their copyright
files in `/usr/share/doc/*/copyright`; list them with
`docker compose exec klarinotte dpkg-query -W`. Their sources are at
`https://snapshot.debian.org` (Debian 12 "bookworm"); we will also send them on
request for three years after each image release: ask through the
[contact page](https://www.klarinotte.com/contact).
