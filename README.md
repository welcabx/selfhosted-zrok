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
| HTTP | TCP | 80 | All IPv4/v6 |
| HTTPS | TCP | 443 | All IPv4/v6 |
| ziti-controller | TCP | 1280 | All IPv4/v6 |
| ziti-router | TCP | 3022 | All IPv4/v6 |

> Added ziti controller and router ports to avoid issues: #3 and #5 

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
    zrok admin create frontend <frontendID> public http://{token}.zrok.mydomain.com:8080
    ```
  - > [!TIP]
    > If you are not able to save the frontend ID you can retrieve by doing the following:
    ```bash
    ziti edge login localhost:1280 -u admin -p <zitiPassword>
    ziti edge list identities
    ```
    > Issue retrieving frontend ID, please refer to [Issue #1](https://github.com/welcabx/selfhosted-zrok/issues/1)

## 9. Configure and start the public frontend
- Create a http fronted config file in `/opt/zrok/etc/http-frontend.yml`
  ```bash
  v:  4
  host_match: zrok.mydomain.com
  address: 0.0.0.0:8080
  ```

- Ensure that the `zrok controller` is still running.
  - Open new SSH terminal
  - export the `ZROK_ADMIN_TOKEN` and `ZROK_API_ENDPOINT`
  - initiate public frontend
    - ```bash
      zrok access public /opt/zrok/etc/http-frontend.yml
      ```
      
## 10. Create a user account
Ensure that `ZROK_ADMIN_TOKEN` and `ZROK_API_ENDPOINT` are exported.
```bash
zrok admin create account <email> <password>
```
> Example output: SuGzRPjVDIcF

Save it for later.

## 11. Create Cloudflare API Token
- Go to `https://dash.cloudflare.com/profile/api-tokens`
- Click `Create token` and use the `Edit zone DNS` template
- Under `Zone resources`, select `include` > `specific zone` > choose mydomain.com
- Click `continue to summary` > `create token` and copy/save the token generated.

## 12. Setting up TLS using Caddy
- verify your VPS architecture
  - ```bash
    uname -m
    ```
- Go to `https://caddyserver.com/download`
  - select the `platform` based on your VPS architecture
  - search `cloudflare` on `filter packages and modules`
  - click and ensure that the `caddy-dns/cloudflare` is highlighted.
  - hover to the `download` button, `right-click` and `copy link`


- Download caddy binary and create config folder
  - ```bash
    curl -o caddy "<paste the caddy link here>"
    chmod +x caddy
    sudo mv caddy /usr/local/bin/caddy
    mkdir -p /etc/caddy
    ```
  
- Store the cloudflare API token
  - ```bash
    echo "CLOUDFLARE_API_TOKEN=yourtokenhere" > /etc/caddy/cloudflare.env
    chmod 400 /etc/caddy/cloudflare.env
    ```

- Create the Caddyfile
  - ```bash
    nano /etc/caddy/Caddyfile
    ```
  - Paste this in:
    - ```bash
      {
        email my@mydomain.com
      }
      
      zrok.mydomain.com {
        reverse_proxy 127.0.0.1:18080
        tls {
          dns cloudflare {env.CLOUDFLARE_API_TOKEN}
        }
      }
      
      *.zrok.mydomain.com {
        reverse_proxy 127.0.0.1:8080
        tls {
          dns cloudflare {env.CLOUDFLARE_API_TOKEN}
        }
      }
      ```
      
- Ensure that the `controller` and `frontend` are running. Check if the associated ports are listening.
  - ```bash
    ss -tulnp | grep zrok
    ```
    > Listening ports should be `18080` and `8080`
  - If not, run them each on a separate SSH terminal:
    - ```bash
      zrok controller /opt/zrok/etc/ctrl.yml
      zrok access public /opt/zrok/etc/http-frontend.yml 
      ```

- Run `caddy` manually
  - ```bash
    export $(cat /etc/caddy/cloudflare.env | xargs)
    caddy run --config /etc/caddy/Caddyfile
    ```
  - It might take up to `60 seconds`
  - If you see `certificate obtained successfully for zrok.mydomain.com` TLS is completed.

- Go back to your VPS console,`Add inbound rule` firewall for `http` and `https`
- verify TLS by opening a new terminal
  - ```bash
    curl -I https://zrok.mydomain.com
    ```
  - If you get `http` response back, TLS and proxying are working.

## 13. zrok controller, frontend and caddy as systemd

- create the following files at `/etc/systemd/system/`
  - zrok-controller.service
    ```bash
    [Unit]
    Description=zrok Controller
    After=network.target
  
    [Service]
    Type=simple
    ExecStart=/usr/bin/zrok controller /opt/zrok/etc/ctrl.yml
    Restart=on-failure
    RestartSec=5
    User=root
    WorkingDirectory=/opt/zrok/etc
  
    [Install]
    WantedBy=multi-user.target
    ```
  
  - zrok-frontend.service
    ```bash
    [Unit]
    Description=zrok Frontend
    After=network.target zrok-controller.service
    Requires=zrok-controller.service
  
    [Service]
    Type=simple
    ExecStart=/usr/bin/zrok access public /opt/zrok/etc/http-frontend.yml
    Restart=on-failure
    RestartSec=5
    User=root
    WorkingDirectory=/opt/zrok/etc
  
    [Install]
    WantedBy=multi-user.target
    ```
  
  - caddy.service
    ```bash
    [Unit]
    Description=Caddy
    After=network.target zrok-frontend.service
  
    [Service]
    Type=simple
    ExecStart=/usr/local/bin/caddy run --config /etc/caddy/Caddyfile
    EnvironmentFile=/etc/caddy/cloudflare.env
    Restart=on-failure
    RestartSec=5
    User=root
  
    [Install]
    WantedBy=multi-user.target
    ```
    
- Enable and start all of them
  ```bash
  systemctl daemon-reload
  systemctl enable --now zrok-controller
  systemctl enable --now zrok-frontend
  systemctl enable --now caddy
  ```
  
- Verify the services are running
  ```bash
  systemctl status zrok-controller zrok-frontend caddy
  ss -tlnp
  ```