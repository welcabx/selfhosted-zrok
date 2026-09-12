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


## 3. DNS setup via Cloudflare

`yourdomain.com` DNS stays on Cloudflare. There is no nameserver delegation needed.
We will use the subdomain as the domain on this setup. In this case, we will use `zrok.mydomain.com` and its wildcard.
> You can choose whatever subdomain you want.

On the cloudflare's account home, click your registered domain.
- Go to `DNS` > `Records` > `Add Record`


**Added the following records:**

| Type | Name | IPv4 address | Proxy status |
|---|---|---|---|
| A | zrok.mydomain.com | vpsPublicIP | DNS only |
| A | *.zrok.mydomain.com | vpsPublicIP | DNS only |

Verified with `dig` or `dnschecker.org` before proceeding.

## 4. Deploy OpenZiti Controller
SSH to your VPS and do the following
```bash
curl -sS https://get.openziti.io/install.bash | sudo bash -s openziti
sudo apt-get install openziti-controller
```

Bootstrap answers:
- Configure ziti-controller now? **Y**
- Joining an existing cluster? **N**
- Permanent external address: **admin.zrok.mydomain.com** 
  - We use `admin` but you can use anyname
- Enter the unique name for this node in the cluster: **Press Enter**
- Enter the trust domain shared by all nodes in the cluster: **Press Enter**
- Enter the controller port [1280]: **Press Enter**
- Enter the name of the default user [admin]: **Press Enter**
- Generate a random password for 'admin': **Y** 
  - Save the output in a safe place
- Configure the OpenZiti Console (web UI)? **N**
- Enable automatic certificate renewal timer? **Y**

Verified ziti controller is running:
```bash
sudo systemctl status ziti-controller.service   # active (running)
```
Login to the ziti controller
```bash
ziti edge login localhost:1280 -u admin -p '<password>' -y   # success
```


## 5. Deploy OpenZiti Router
Create a ziti router's enrollment token.
```bash
ziti edge create edge-router "router1" -o /tmp/router1.jwt
```

Install ziti router
```bash
sudo apt-get install openziti-router
```

Bootstrap answers:
- Configure ziti-router now? **Y**
- Permanent external address: **router1.zrok.mydomain.com**
- Router port: **3022** (default)
- Enrollment token: /tmp/router1.jwt

Verified:
```bash
sudo systemctl status ziti-router.service     # active (running)
ziti edge list edge-routers                   # router1 shown, connected
```

## 6. Install zrok Binary

```bash
sudo apt install zrok
zrok version
```

## 7. Configure controller
- create working directory `/opt/zrok/etc`
- make a zrok controller configuration file `etc/ctrl.yml`
- generate a random admin secrets and save it on safe place.
  - ```LC_ALL=C tr -dc _A-Z-a-z-0-9 < /dev/urandom | head -c32```
- configure the controller file
  - ```bash
    v:  4
    
    admin:
      secrets:
        - yourRandomsecretGenerated
    
    endpoint:
      host: 0.0.0.0
      port: 18080
    
    invites:
      invites_open: true
    
    store:
      path: zrok.db
      type: sqlite3
    
    ziti:
      api_endpoint: "https://127.0.0.1:1280"
      username: admin
      password: "yourZitiPassword"
    ```
    
- export zrok token and endpoint based on the `ctrl.yml`
  - ```bash
    export ZROK_ADMIN_TOKEN=adminToken
    export ZROK_API_ENDPOINT=http://127.0.0.1:18080
    ```
    
- bootstrap openziti for zrok
  - ```bash
    zrok admin bootstrap /opt/zrok/etc/ctrl.yml
    ```
  - Notice the warning:
    ```WARNING zrok/controller.Bootstrap: missing public frontend for ziti id 'sqJRAINSiB'; please use 'zrok admin create frontend sqJRAINSiB public https://{token}.your.dns.name' to create a frontend instance```

    
- Run zrok controller
  - ```bash
    zrok controller /opt/zrok/etc/ctrl.yml
    ```

## 8. Create zrok frontend

Let the zrok controller run and open a new terminal. SSH to the VPS and do the following:

- export zrok token and endpoint based on the `ctrl.yml`
  - ```bash
    export ZROK_ADMIN_TOKEN=adminToken
    export ZROK_API_ENDPOINT=http://127.0.0.1:18080
    ```
- create the public frontend by using the ziti frontend id earlier. The one that shows a `warning` after we run `zrok admin bootstrap`.
  - ```bash
    zrok admin create frontend <frontendID> public http://{token}.zrok.quigley.com:8080
    ```
  - > [!TIP]
    > If you are not able to save the frontend ID you can retrieve by doing the following:\
    ```bash
    ziti edge login localhost:1280 -u admin -p <zitiPassword>
    ziti edge list identities
    ```
    > Issue retrieving frontend ID, please refer to [Issue #1](https://github.com/welcabx/selfhosted-zrok/issues/1)

**Next up:**
- [ ] Create working directory for zrok config (e.g. `/opt/zrok/etc`)
- [ ] Write `etc/ctrl.yml` (zrok controller config — endpoint + `ziti` stanza with admin credentials)
- [ ] Run `zrok admin bootstrap etc/ctrl.yml`
- [ ] Create the public frontend (`etc/http-frontend.yml`, `host_match: zrok.vicably.com`)
- [ ] Put a reverse proxy (Caddy/Nginx) in front for TLS, using Cloudflare DNS-01 challenge for the wildcard cert
- [ ] Point the Termux zrok client at the new instance (`zrok config set apiEndpoint ...`, `zrok enable <token>`)

---

*Credentials (admin passwords, tokens, JWTs) are intentionally not recorded in this file — keep those in a secure password manager instead.*