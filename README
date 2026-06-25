# 3x-UI VLESS REALITY Proxy — Oracle Free Tier VPS Setup Guide

> **Goal:** A hardened, HTTPS-only 3x-UI panel on Oracle Cloud running VLESS+REALITY on port 443 — the gold standard for censorship circumvention.
>
> **Assumptions:** Ubuntu Server 22.04/24.04, Oracle Cloud Free Tier (A1 Flex ARM *recommended* or E2.1 Micro AMD). You will generate an SSH key pair during VM creation.

---

## ⚠️ Oracle-Specific Warning — Two Firewall Layers

Oracle Cloud has **two independent firewalls**. You must open ports in **both** or nothing will work:

1. **Oracle Cloud Security List** — network-level, managed in the OCI web console
2. **OS-level firewall** — iptables/UFW on the server itself (Oracle Ubuntu ships with restrictive default rules)

Missing either one is the most common reason people get stuck.

---

## PHASE 0 — Create Your Oracle Free Tier VPS

> **Oracle's Always Free tier** gives you two instance type options. The **A1 Flex (ARM)** is dramatically better — up to 4 OCPUs and 24 GB RAM, all free. The **E2.1 Micro (AMD)** is the reliable fallback with 1/8 OCPU and 1 GB RAM. Use A1 if your chosen region has available quota.

### Step 0A — Create an Oracle Cloud account

1. Go to **https://cloud.oracle.com** and click **Start for free**.
2. Fill in your details. A credit card is required for identity verification — you **will not** be charged for Always Free resources.
3. Choose a **Home Region** geographically close to your intended users. **This cannot be changed after signup**, so choose carefully.
4. Complete email verification and the setup wizard until you reach the OCI Console dashboard.

---

### Step 0B — Create a VM instance

1. In the OCI Console, click the **☰ (hamburger) menu → Compute → Instances → Create Instance**.
2. **Name:** Give it something neutral (e.g., `node-01`). Avoid names like `vpn`, `proxy`, `xray`.
3. **Image and shape** — click **Edit**:
   - **Image:** Click **Change image** → switch from Oracle Linux to **Ubuntu** → select **Ubuntu 22.04** or **24.04** → click **Select image**.
   - **Shape:** Click **Change shape**:

     | Option | Category | Shape | Recommended specs |
     |--------|----------|-------|-------------------|
     | **A1 ARM (recommended)** | Ampere | `VM.Standard.A1.Flex` | 1–2 OCPU, 6–12 GB RAM |
     | **E2 Micro (fallback)** | AMD | `VM.Standard.E2.1.Micro` | Fixed (1/8 OCPU, 1 GB) |

     > If A1 shows "Out of capacity" try different **Availability Domains** (AD-1, AD-2, AD-3) or try again in a few hours — free A1 capacity fluctuates.

4. **Networking:**
   - Let OCI create a new VCN automatically, or choose an existing one.
   - Ensure **"Assign a public IPv4 address"** is set to **Yes**.

5. **SSH Keys** — click **Edit**:
   - Select **Generate a key pair for me**.
   - Click **Save private key** — download the `.key` file immediately. **This is the only chance to download it.**
   - Optionally save the public key file as well.

6. **Boot volume:** Default 50 GB is fine. Leave unchanged.

7. Click **Create**. The instance starts in **Provisioning** state and becomes **Running** in 1–3 minutes.

> After the instance is Running, note the **Public IP** shown in the instance details page. If no public IP is shown, follow Step 0C below.

---

### Step 0C — Reserve and assign a static public IP

> By default, Oracle assigns an *ephemeral* public IP that may change if the instance is stopped or terminated. Assigning a **Reserved** public IP gives you a permanent static address that survives reboots and can be re-attached to a new instance without changing your DNS records.

1. In the OCI Console, go to **Networking → Virtual Cloud Networks** → click your VCN name.
2. In the left sidebar under **Resources**, click **Subnets** → click your **public subnet**.
3. Click the **IP Addresses** tab (this shows all private IPs allocated in the subnet).
4. Locate the private IP address of your instance in the list.
5. Click the **⋮ (three-dot)** Actions menu on that row → click **Edit**.
6. Under **Public IP type**, change the selection from **Ephemeral** to **Reserved Public IP**.
7. Select **Create a new reserved public IP**, give it a name (e.g., `vpn-node-ip`), then click **Update**.

Your instance now has a permanent, static public IP. Note this IP — you will use it for SSH and DNS records.

> **Alternative path (via the instance):** Compute → Instances → [your instance] → scroll down to **Primary VNIC** → click the VNIC link → **IPv4 Addresses** tab → ⋮ → **Edit** → assign Reserved Public IP from there.

---

## PHASE 1 — Oracle Cloud Console (Web UI)

### Step 1 — SSH into your server

**On your local machine (Linux/macOS Terminal or Windows PowerShell):**

```bash
chmod 600 /path/to/your/private_key.key
ssh -i /path/to/your/private_key.key ubuntu@YOUR_VPS_PUBLIC_IP
```

**What this does:** `chmod 600` restricts the key file's permissions — SSH refuses to use keys that are world-readable. The `ssh` command opens an encrypted tunnel to your Oracle VM. `ubuntu` is the default username on Oracle Ubuntu images. Replace the key path and IP with your values.

**Windows PowerShell note:** Set permissions with:
```powershell
icacls "C:\path\to\key.key" /inheritance:r /grant:r "$env:USERNAME:(R)"
```

**Customizable?** You can create additional users with different usernames later. Key-based auth is far more secure than password auth — keep it.

**Reversible?** Disconnect anytime with `exit`. No server-side changes are made by this command alone.

---

### Step 2 — Open ports in Oracle Cloud Security List

**Do this in the OCI Console (browser), not in the terminal.**

1. Go to: **Networking → Virtual Cloud Networks → Your VCN → Security Lists → Default Security List**
2. Click **Add Ingress Rules** and add each rule below one at a time:

| Rule | Source CIDR | Protocol | Port | Purpose |
|------|-------------|----------|------|---------|
| SSL cert | `0.0.0.0/0` | TCP | `80` | Let's Encrypt HTTP challenge |
| Proxy (TCP) | `0.0.0.0/0` | TCP | `443` | VLESS REALITY |
| Proxy (UDP) | `0.0.0.0/0` | UDP | `443` | VLESS REALITY + QUIC |
| Panel admin | `0.0.0.0/0` | TCP | `PANEL_PORT` | 3x-UI web panel *(see security note)* |
| Subscription | `0.0.0.0/0` | TCP | `SUB_PORT` | Client subscription feed *(see note)* |

> **🔒 Security tip — randomize your panel port:** Instead of a predictable port like `2053`, pick a random 4–5 digit port (e.g., `31847` or `54219`). Bots continuously scan common ports — an obscure port dramatically reduces your panel's exposure. Whatever port you choose here, use the same number in the UFW step and when installing 3x-UI.

> **📋 Subscription port note:** A common choice is `2096` but any unused port works. Open whatever port you later configure in **Panel Settings → Subscription**. If you don't plan to share subscription links, you can skip this rule for now and add it later.

> **Port 443 UDP explained:** Standard VLESS REALITY operates over TCP. Opening UDP 443 additionally supports QUIC-based transports and is recommended for future compatibility. Add it as a **separate ingress rule** with Protocol = UDP and Port = 443.

For each rule: **Source Type = CIDR**, **Source CIDR = `0.0.0.0/0`**, set **IP Protocol** and **Destination Port Range** as shown.

**What this does:** The Oracle Cloud Security List is a cloud-level packet filter. Without these rules, Oracle drops packets before they even reach your VM's OS. Port 80 is needed for SSL certificate issuance via HTTP challenge. Port 443 TCP+UDP handles your proxy traffic. The panel port opens the 3x-UI admin interface.

**Customizable?** Restrict the panel port to your own home IP for extra security: change Source CIDR to `YOUR.HOME.IP/32` (e.g., `203.0.113.5/32`) so only you can reach the login page.

**Reversible?** Delete any ingress rule from the Security List at any time to immediately block that port at the network level.

---

## PHASE 2 — Server Preparation (In Your SSH Session)

### Step 3 — Become root

```bash
sudo su
```

**What this does:** Switches your session to the root user, giving full system permissions without needing to prefix every command with `sudo`. All following steps assume you are root.

**Customizable?** Prefix each command with `sudo` instead if you prefer not to stay as root.

**Reversible?** Type `exit` to return to the `ubuntu` user at any time.

---

### Step 4 — Update the system

```bash
apt update && apt upgrade -y
```

**What this does:** `apt update` refreshes the local package list from Ubuntu's repositories. `apt upgrade -y` installs all available security and feature updates. Always do this before exposing a new server to the internet.

**Customizable?** Remove `-y` to review what gets upgraded before it installs.

**Reversible?** Package upgrades are generally not reversible. This is standard best practice and should always be done first.

---

### Step 5 — Fix Oracle's iptables (CRITICAL for Oracle Cloud Ubuntu)

Oracle Ubuntu images ship with restrictive iptables rules that block all incoming ports by default. You must flush them:

```bash
iptables -F INPUT
iptables -P INPUT ACCEPT
ip6tables -F INPUT
ip6tables -P INPUT ACCEPT
```

Then save these open rules so they persist across reboots:

```bash
apt install iptables-persistent -y
netfilter-persistent save
```

When the installer asks to save current IPv4/IPv6 rules, answer **Yes** to both.

**What this does:** `iptables -F INPUT` flushes all existing INPUT chain rules. `iptables -P INPUT ACCEPT` sets the default policy to accept all incoming traffic. `netfilter-persistent save` writes these rules to `/etc/iptables/rules.v4` and `/etc/iptables/rules.v6` so they reload on reboot. Without this step, UFW rules (next step) are silently blocked by Oracle's underlying iptables.

**Customizable?** You can add specific iptables rules later to rate-limit connections. UFW will layer on top.

**Reversible?** To restore Oracle's restrictive defaults you'd need to re-add them manually. For a proxy server, open INPUT rules are correct.

---

### Step 6 — Configure UFW firewall

Replace `PANEL_PORT` with your chosen panel port number and `SUB_PORT` with your subscription port:

```bash
apt install ufw -y
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 443/udp
ufw allow PANEL_PORT/tcp
ufw allow SUB_PORT/tcp
ufw enable
```

Type `y` when asked to confirm enabling UFW.

**Example** — if your panel port is `31847` and subscription port is `2096`:
```bash
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 443/udp
ufw allow 31847/tcp
ufw allow 2096/tcp
ufw enable
```

**What each rule does:**

| Command | Purpose |
|---------|---------|
| `ufw allow 22/tcp` | Keep SSH accessible so you don't lock yourself out |
| `ufw allow 80/tcp` | SSL certificate issuance (Let's Encrypt HTTP challenge) |
| `ufw allow 443/tcp` | VLESS REALITY proxy traffic (TCP) |
| `ufw allow 443/udp` | VLESS REALITY proxy traffic (QUIC/UDP) |
| `ufw allow PANEL_PORT/tcp` | 3x-UI admin panel |
| `ufw allow SUB_PORT/tcp` | Subscription link URL |

**Customizable?** Restrict the panel port to your IP only for maximum security:
```bash
ufw allow from YOUR.HOME.IP.ADDRESS to any port PANEL_PORT proto tcp
```

**Reversible?** `ufw disable` turns off UFW entirely. `ufw delete allow 443/tcp` removes a specific rule.

---

### [Optional] Step 6B — Customize your SSH port for security

> Changing the SSH port from the default `22` to a non-standard port cuts automated brute-force attacks and port-scanner noise dramatically. Strongly recommended for any long-running server.

**1. Open the SSH configuration file:**
```bash
nano /etc/ssh/sshd_config
```

**2. Find the line `#Port 22`** (it may be commented out with `#`). Remove the `#` and change the port number:
```
Port 47832
```
Replace `47832` with any unused port of your choice. Avoid ports already claimed: `80`, `443`, your panel port, your subscription port.

**3. Save and exit:** Press `Ctrl+X` → `Y` → `Enter`.

**4. Restart the SSH service:**
```bash
systemctl restart sshd
```

**5. Open the new SSH port in UFW:**
```bash
ufw allow 47832/tcp
```

**6. Open the new SSH port in Oracle Security List:**
Go to **OCI Console → Networking → VCN → Security Lists → Default Security List → Add Ingress Rules** → add **TCP port `47832`**.

**7. ⚠️ Test the new port FIRST — open a new terminal window and confirm it works:**
```bash
ssh -i /path/to/key.key -p 47832 ubuntu@YOUR_VPS_PUBLIC_IP
```
If it connects successfully, you can now safely remove port 22.

**8. Remove the old port 22 rule from UFW:**
```bash
ufw delete allow 22/tcp
```

**9. Remove the port 22 ingress rule** from the Oracle Security List (click the ⋮ menu on that rule → Delete).

> ⚠️ **Recovery:** If you accidentally lock yourself out, go to **OCI Console → Compute → Instances → [your instance] → Console connection** to access the VM via browser without SSH.

**Future SSH connections** will now use:
```bash
ssh -i /path/to/key.key -p 47832 ubuntu@YOUR_VPS_PUBLIC_IP
```

**Reversible?** Set `Port 22` back in `/etc/ssh/sshd_config`, restart sshd, then re-open port 22 in UFW and Oracle Security List.

---

## PHASE 3 — 3x-UI Installation

### Step 7 — Install 3x-UI

```bash
bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
```

The installer will ask a few interactive questions:
- **Custom port?** → Enter your chosen panel port (e.g., `31847` or whatever you opened in Steps 2 and 6)
- **Custom path?** → Accept the random path it generates and **write it down immediately**
- At the end, it prints your **username, password, and panel URL in green text** — save all three

**What this does:** Downloads and runs the official 3x-UI installer. It installs the 3x-UI binary, registers it as a systemd service (auto-starts on reboot), generates random credentials, and starts the panel. Xray-core (the actual proxy engine) is bundled inside.

**Customizable?** Everything set during install (port, path, credentials) can be changed later via the `x-ui` management menu.

**Reversible?** To uninstall: run `x-ui` → select the uninstall option. Removes the binary, service, and all data.

---

### Step 8 — Verify the panel port

```bash
x-ui
```

This opens the management menu. If you didn't set the port during install, select **option 9 (Change Port)** and enter your panel port. Then select **1 (Start)** or **2 (Restart)** to apply.

**What this does:** `x-ui` is the management CLI installed by 3x-UI. It lets you start/stop the service, reset credentials, manage SSL, change settings, and more — without editing config files manually.

**Reversible?** Change the port again at any time via `x-ui → option 9`.

---

## PHASE 4 — Domain & SSL Setup

> SSL is **not optional** — running the panel over plain HTTP exposes your login credentials and reveals that you are operating a proxy panel, which can have serious consequences in censored environments.

### Step 9 — Get a free domain

Go to **https://dash.domain.digitalplat.org/** and register a free domain. Fake registration info is fine. Choose a neutral, non-suspicious name — avoid words like `proxy`, `vpn`, `bypass`, `xtls`, `v2ray`, `tunnel`, `tor`. Something innocuous like `mycloudfiles.qzz.io` or `docstore-net.42web.io` works well.

**No commands needed** — done entirely in your browser.

---

### Step 10 — Get your server's public IP

```bash
curl https://ipinfo.io/ip
```

Returns your VPS's public IPv4 address. You'll need this for the DNS A record in the next step.

---

### Step 11 — Set up Cloudflare and DNS

Cloudflare acts as your DNS provider and can optionally hide your real server IP behind their proxy network.

**1. Create a free Cloudflare account** at **https://cloudflare.com**

**2. Add your domain:**
- Dashboard → **Add a Site** → enter your DigitalPlat domain (e.g., `yourdomain.qzz.io`)
- Select the **Free plan**
- Cloudflare will scan for existing DNS records (likely finds none). Click **Continue to activation**.

**3. Get your Cloudflare nameservers:**
- Cloudflare gives you two nameserver addresses (e.g., `anna.ns.cloudflare.com` and `bob.ns.cloudflare.com`)
- Copy both of them

**4. Point your domain to Cloudflare:**
- Go back to **DigitalPlat → My Domains → Manage** your domain
- Find the **Nameservers (NS)** section
- Replace **NS1** and **NS2** with Cloudflare's two nameserver addresses → Save
- Back in Cloudflare, click **Done, check nameservers**
- Wait **5–30 minutes** for nameserver propagation

**5. Add DNS records in Cloudflare → DNS → Records → Add Record:**

**Record 1 — Direct server IP (for SSL and VLESS connections):**
```
Type:    A
Name:    files          ← or any neutral subdomain prefix you like
Content: YOUR_VPS_PUBLIC_IP
Proxied: OFF (grey cloud)   ← must be grey/unproxied for this record
TTL:     Auto
```

**Record 2 — Apex domain through Cloudflare proxy (hides your IP):**
```
Type:    CNAME
Name:    @
Content: files.yourdomain.qzz.io   ← the full subdomain from Record 1
Proxied: ON (orange cloud)
```

**What this does:** The **grey-cloud A record** (`files.yourdomain.qzz.io`) exposes your real server IP directly — required for SSL certificate issuance and for the VLESS REALITY proxy connection (which must bypass Cloudflare's proxy). The **orange-cloud CNAME** (`yourdomain.qzz.io`) routes lookups through Cloudflare, masking your real IP from casual observers.

> **Which domain to use where:**
> - Use `files.yourdomain.qzz.io` (grey cloud) → in your VLESS client config and for SSL cert issuance
> - Use `yourdomain.qzz.io` (orange cloud) → optionally for the panel URL in browser (IP stays hidden)

**Reversible?** Delete DNS records in Cloudflare anytime. Updating DigitalPlat's nameservers back to their defaults removes Cloudflare from the chain.

---

### Step 12 — Issue SSL certificate via Cloudflare DNS

**First, create a Cloudflare API Token:**

1. Go to **https://dash.cloudflare.com/profile/api-tokens**
2. Click **Create Token** → choose the **Edit zone DNS** template
3. Under **Zone Resources** → **Specific zone** → select your domain
4. Click **Continue to Summary** → **Create Token**
5. Copy the token immediately — it is **shown only once**

> **Security note:** This scoped token only allows editing DNS records for your specific zone. Never use your Global API Key here — it grants full account control.

**On the server:**

```bash
x-ui
```

Select option **20 (Cloudflare SSL Certificate)**. When prompted:
- Choose **`t`** for API Token (recommended)
- Enter your full subdomain: `files.yourdomain.qzz.io` (the grey-cloud A record — this is what the certificate is issued for)
- Paste your Cloudflare API Token

The script uses the ACME protocol with Cloudflare DNS-01 challenge — it temporarily creates a DNS TXT record to prove domain ownership, then issues a Let's Encrypt certificate. Your server **does not need to be reachable on port 80** for this method. The certificate auto-renews silently every 90 days.

After completion, return to the `x-ui` menu → navigate to **Panel Settings → SSL Certificate** (or equivalent option) and confirm the certificate paths are set. The panel will restart and serve HTTPS only.

> **Panel URL after SSL:**
> ```
> https://files.yourdomain.qzz.io:PANEL_PORT/your-random-path/
> ```

**Alternative — HTTP challenge via certbot (uses port 80):**

If you prefer not to use Cloudflare API tokens, you can use certbot instead. Port 80 is already open from Steps 2 and 6:
```bash
apt install certbot -y
certbot certonly --standalone -d files.yourdomain.qzz.io
```
Certificates are saved to `/etc/letsencrypt/live/files.yourdomain.qzz.io/`. Point the panel to `fullchain.pem` and `privkey.pem`.

**Reversible?** Certs can be removed via `x-ui → SSL management`. The panel falls back to plain HTTP without a valid certificate.

---

## PHASE 5 — Create Your Proxy Inbound

### Step 13 — Access the panel

From the `x-ui` menu, select **10 (View Current Settings)** to see your panel URL. Open it in a browser:

```
https://files.yourdomain.qzz.io:PANEL_PORT/your-random-path/
```

Log in with the credentials printed during installation (or reset them via `x-ui → option 6`).

**Immediately after logging in: go to Panel Settings → Authentication and change your username and password to something strong and unique.**

---

### Step 14 — Create VLESS REALITY inbound

In the panel, go to the **Inbounds** tab → click **Add Inbound** and fill in:

```
Remark:   XTLS REALITY     (any label you like)
Port:     443
Protocol: VLESS
Security: Reality
Target:   www.microsoft.com:443   (or any reliable unblocked HTTPS site)
SNI:      www.microsoft.com       (must match the Target domain exactly)
```

Click **Get New Cert** — this generates the Reality x25519 key pair (public/private keys for the handshake). Leave remaining defaults, then click **Create**.

**What this does:** Creates a VLESS proxy listener on port 443. REALITY is a TLS masquerading technique — your server presents a valid TLS certificate from the Target domain (e.g., Microsoft), so to any deep-packet inspection or censorship system, the traffic looks identical to normal HTTPS traffic to Microsoft's servers. This is why REALITY is the strongest anti-censorship protocol available — there is no VPN fingerprint to detect.

**Good Target/SNI choices:** `www.microsoft.com`, `www.amazon.com`, `cdn.jsdelivr.net`. Avoid Google (often blocked in censored regions).

**Reversible?** Delete or toggle off the inbound from the panel at any time. Changing Target/SNI requires editing the inbound and restarting Xray.

---

### Step 15 — Get your client config

1. In the Inbounds table, find your REALITY inbound and click the **+** icon under the **ID** column to expand the client list.
2. Click the **QR code icon** next to the default client entry.
3. Click the QR code image to copy the `vless://...` string to clipboard — or scan it directly with your mobile client app.

**Customizable?** Click the 3-dot menu on the inbound to **Add Client** — each client can have its own traffic quota, expiry date, and IP limit. This lets you create separate configs per device or per person.

**Reversible?** Delete or disable individual clients from the panel at any time.

---

### Step 15B — Configure subscription URL (Optional)

The subscription URL lets your clients automatically receive config updates without you re-sending QR codes.

1. In the panel, go to **Panel Settings → Subscription**.
2. Enable the subscription service.
3. Set the **Subscription Port** to the port you opened in Steps 2 and 6 (e.g., `2096`).
4. Optionally set a custom subscription path (a random path is more secure than a predictable one).
5. Save and restart the panel.
6. Each client's personal subscription URL now appears in their row in the Inbounds client list — share it with them.

Most clients (v2rayN, NekoBox, V2Box) support importing via subscription URL and can auto-refresh on a schedule to pick up new configs automatically.

> ✅ **Confirm** that the subscription port is open in **both** the Oracle Security List (Step 2) and UFW (Step 6) before testing.

---

## PHASE 6 — Connect from Your Device

### Step 16 — Install a client app

| Platform | Recommended App | Download |
|----------|----------------|----------|
| Windows | v2rayN | https://github.com/2dust/v2rayN/releases |
| Android | NekoBox | https://github.com/MatsuriDayo/NekoBoxForAndroid/releases |
| iOS | V2Box | App Store search: "V2Box" |
| macOS | V2RayXS | https://github.com/tzmax/V2RayXS/releases |

All of these support the Xray core and VLESS REALITY.

---

### Step 17 — Import your config

**Option A — QR Code (mobile):** Open app → Add Server → Scan QR Code → point at the QR from Step 15.

**Option B — Clipboard (desktop):** Copy the `vless://...` string → open app → Add Server → Import from clipboard.

**Option C — Subscription URL:** Open app → Subscriptions → Add → paste the subscription URL from Step 15B. The app imports all configs automatically and can refresh on a schedule.

After importing manually (Options A or B), **edit the server entry** and set the **Server Address** field to `files.yourdomain.qzz.io` (the grey-cloud A record — the direct IP subdomain, not the orange-cloud proxied domain). REALITY must connect directly to your server, not through Cloudflare.

**What this does:** Setting the address to the direct subdomain ensures the VLESS handshake reaches your server without Cloudflare in the path, which would break the REALITY protocol.

**Reversible?** Delete the server entry in the app at any time.

---

### Step 18 — Connect and test

1. In your client app, select the imported server and click **Connect**.
2. Visit **https://ipinfo.io** — it should show your VPS's IP address, not your real home IP.
3. Test speed at **https://fast.com** or **https://speedtest.net**.

---

## Ongoing Maintenance

### Update 3x-UI

```bash
bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
```

Running the installer again upgrades to the latest version while preserving your existing configuration and inbounds.

### Useful x-ui commands

```bash
x-ui              # Opens the interactive management menu
x-ui status       # Shows service status
x-ui restart      # Restarts the service
x-ui log          # View live logs
```

### Check which ports are listening

```bash
ss -tlnp          # TCP listening ports
ss -ulnp          # UDP listening ports
ufw status        # Current UFW rules
```

---

## Troubleshooting Quick Reference

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| Can't reach panel in browser | Panel port not open in Oracle Security List | Re-check Phase 1, Step 2 |
| Panel loads but proxy traffic fails | Port 443 TCP or UDP not open | Verify both TCP and UDP 443 in Security List and UFW |
| Panel reachable but shows HTTP warning | SSL not configured | Complete Phase 4 |
| Cloudflare SSL option fails | Wrong option number or token scope | Use **option 20** in x-ui; verify token has DNS Edit permission for your zone |
| Certbot fails on port 80 challenge | Port 80 not open in Security List or UFW | Add TCP 80 in both (Steps 2 & 6) |
| Subscription URL not reachable | Sub port blocked | Confirm sub port is open in Oracle Security List AND in UFW |
| Client connects but can't browse | Wrong SNI/Target or DNS not propagated yet | Wait 30 min for DNS, re-check SNI matches Target exactly |
| Connection works but very slow | Oracle's 480 Mbps free tier bandwidth cap | Expected — hardware limit of Always Free tier |
| `x-ui` command not found | Still logged in as `ubuntu`, not root | Run `sudo su` first |
| Locked out of SSH after port change | New port not tested before removing port 22 | Use **OCI Console → Compute → Instances → Console Connection** to recover |
| A1 instance shows "Out of capacity" | ARM quota unavailable in chosen region/AD | Try other Availability Domains (AD-1, AD-2, AD-3), or use E2.1 Micro |

---

*Guide compiled from official 3x-UI wiki, install scripts, OCI documentation, and Wispy Docs XTLS REALITY guide.*
