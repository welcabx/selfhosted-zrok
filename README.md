# Self-Hosting zrok 

This is a manual setup guide in self-hosting [zrok]('https://zrok.io') on a VPS.

> [!IMPORTANT]
> - **VPS:** Akamai Cloud
> - **Domain:** `sub.domain.com` and its wildcard. DNS managed on Cloudflare
> - **Approach:** Manual installation of OpenZiti's controller, router and zrok

---

## 1. Akamai Cloud VPS

- Signed up for Akamai Cloud, activated $100 / 60-day trial credit (requires a payment method on file; not charged during trial)
- Created a Linode VPS under Compute:
  - **Region:** `Singapore` (The lowest ping. Check their speedtest page.)
  - **OS:** `Ubuntu 24 LTS`
  - **Linode Plan:** `Nanode 1 GB` under `Shared CPU` tab
  - **Linode Label:** `zrok-sg` (Feel free to name it different)
  - **SSH keys:** Added public SSH key and checked it
  - **Disk Encryption:** checked `encrypt disk`
  - **Network Connection:** `Public internet`
  - **IP Address:** `Auto assigned`
  - **Network interface type:** `Linode interfaces`
  - **Public interface firewall:** Create firewall, set it to `custom`, added label as `zrok-fw` and submit

## 2. Setup SSH 
Go to the VPS console > `Networking` > `Firewalls`:
- Choose the firewall created and `add an inbound rule`
- Choose `Preset` to `SSH`
- Get the `CIDR` of your device by checking `whatismyip`
- Copy and paste your device `CIDR` to `Source` select 'IP/Netmask' instead of using default `All IPv4, All IPv6`
- Click on `Add Rule` and then `Save changes`

**Inbound rule added:**

| Label | Protocol | Port | Source          |
|---|---|---|-----------------|
| SSH | TCP | 22 | deviceIP/subnet |


## 3. DNS (Cloudflare)

vicably.com's DNS stays on Cloudflare (no nameserver delegation needed):

- **Wildcard record:** `A` `*.zrok` → Linode's public IP — **DNS only (grey cloud)**, not proxied
- **Controller record:** `A` `admin.zrok` → same IP (covered by the wildcard, used as the controller's FQDN instead of a separate subdomain)

Verified with `dig` / dnschecker.org before proceeding.

Cloudflare API token generated (Edit zone DNS, scoped to vicably.com) — reserved for the TLS/Caddy step later.

## 4. OpenZiti Controller

```bash
curl -sS https://get.openziti.io/install.bash | sudo bash -s openziti
sudo apt-get install openziti-controller
```

Bootstrap answers:
- Configure ziti-controller now? **Y**
- Joining an existing cluster? **N** (this is the first/only node)
- Permanent external address: **admin.zrok.vicably.com**
- Install OpenZiti Console (ZAC)? **N** (skipped — not required for zrok)
- Enable automatic certificate renewal timer? **Y**

Verified:
```bash
sudo systemctl status ziti-controller.service   # active (running)
ziti edge login admin.zrok.vicably.com:1280 -u admin -p '<password>' -y   # success
```

Controller edge API port: **1280** (added to firewall).

## 5. OpenZiti Router

```bash
sudo apt-get install openziti-router
```

Bootstrap answers:
- Configure ziti-router now? **Y**
- Permanent external address: **admin.zrok.vicably.com**
- Router port: **3022** (default)
- Enrollment token: generated from a second SSH session:
  ```bash
  ziti edge create edge-router router1 -o /tmp/router1.jwt
  ```
  then supplied `/tmp/router1.jwt` as the path at the prompt.

Verified:
```bash
sudo systemctl status ziti-router.service     # active (running)
ziti edge list edge-routers                   # router1 shown, connected
```

## 6. zrok Binary — In Progress

```bash
sudo apt install zrok
zrok version
```

Confirmed this is the correct manual-path package (not the `zrok2` bundled-dependencies variant, which auto-installs PostgreSQL/RabbitMQ/InfluxDB via its own bootstrap flow).

**Next up:**
- [ ] Create working directory for zrok config (e.g. `/opt/zrok/etc`)
- [ ] Write `etc/ctrl.yml` (zrok controller config — endpoint + `ziti` stanza with admin credentials)
- [ ] Run `zrok admin bootstrap etc/ctrl.yml`
- [ ] Create the public frontend (`etc/http-frontend.yml`, `host_match: zrok.vicably.com`)
- [ ] Put a reverse proxy (Caddy/Nginx) in front for TLS, using Cloudflare DNS-01 challenge for the wildcard cert
- [ ] Point the Termux zrok client at the new instance (`zrok config set apiEndpoint ...`, `zrok enable <token>`)

---

*Credentials (admin passwords, tokens, JWTs) are intentionally not recorded in this file — keep those in a secure password manager instead.*