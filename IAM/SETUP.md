<div align="center">

# 🛠️ Setup Guide — Enterprise IAM Security Lab

### How to rebuild this lab from scratch, step by step.

> Read the [README](README.md) first to understand what you are building.

</div>

---

## ⚙️ Requirements

| Requirement | Detail |
|---|---|
| RAM | 16GB minimum, 20GB recommended |
| Disk Space | 100GB free minimum |
| Virtualization | VirtualBox — latest stable version |
| Host OS | Windows 10 or 11 |

**ISO files you need to download before starting:**
- Windows Server 2022 — from Microsoft Evaluation Center
- Ubuntu Server 22.04 — from ubuntu.com
- Windows 10 — from Microsoft
- pfSense — latest stable from pfsense.org

---

## Step 1 — VirtualBox Host Network

Before creating any virtual machine, set up the shared private network.

1. Open VirtualBox
2. Go to File then Host Network Manager
3. Click Create
4. Set IPv4 Address to 192.168.56.1
5. Set Subnet Mask to 255.255.255.0
6. Uncheck the DHCP Server option — all IPs will be set manually
7. Click Apply and close

---

## Step 2 — pfSense VM

**Create the VM:**
- RAM: 512MB
- Adapter 1: NAT
- Adapter 2: Host Only — select the network you just created
- Boot from pfSense ISO and follow the installer defaults

**After install — set LAN IP:**
1. At the pfSense console menu select option 2 — Set interface IP address
2. Choose the LAN interface
3. Set IP to 192.168.56.1 with subnet 24
4. No IPv6, no DHCP server

**Access the web interface:**

Open a browser on your host machine and go to https://192.168.56.1

Default username is admin and default password is pfsense.
Complete the setup wizard and set a strong admin password of your choice.

**Create Certificate Authority:**
1. System then Cert Manager then CAs tab
2. Click Add
3. Name: LabVPN-CA
4. Method: Create an internal Certificate Authority
5. Fill country and organisation with any values
6. Save

**Create Server Certificate:**
1. Certificates tab
2. Click Add then Create an internal Certificate
3. Name: LabVPN-Server
4. Certificate Authority: LabVPN-CA
5. Certificate Type: Server Certificate
6. Save

**Add RADIUS Server:**
1. System then User Manager then Authentication Servers
2. Click Add
3. Descriptive Name: FreeRADIUS
4. Type: RADIUS
5. Hostname: 192.168.56.108
6. Shared Secret: choose a strong secret and note it down — you will use the same value in FreeRADIUS clients.conf
7. Authentication Port: 1812
8. Protocol: PAP
9. Save

**Create OpenVPN Server:**
1. VPN then OpenVPN then Servers
2. Click Add
3. Server Mode: Remote Access — User Auth
4. Backend for authentication: FreeRADIUS
5. Protocol: UDP on IPv4 only
6. Interface: LAN
7. Local Port: 1194
8. Peer Certificate Authority: LabVPN-CA
9. Server Certificate: LabVPN-Server
10. IPv4 Tunnel Network: 10.8.0.0/24
11. IPv4 Local Network: 192.168.56.0/24
12. Save

---

## Step 3 — Windows Server VM

**Create the VM:**
- RAM: 2048MB
- Network Adapter: Host Only
- Boot from Windows Server ISO and install with Desktop Experience

**Set static IP:**
1. Open Network and Sharing Center
2. Change adapter settings
3. Right click the adapter then Properties then IPv4
4. IP: 192.168.56.109
5. Subnet: 255.255.255.0
6. Gateway: 192.168.56.1
7. DNS: 192.168.56.109 — points to itself since it will be the DNS server

**Install Active Directory:**
1. Open Server Manager
2. Add Roles and Features
3. Select Active Directory Domain Services
4. Complete installation
5. Click the flag notification then Promote this server to a domain controller
6. Select Add a new forest
7. Root domain name: lab.local
8. Set a DSRM password of your choice
9. Complete the wizard — server will restart

**Create Organizational Units:**
1. Open Active Directory Users and Computers from Tools
2. Right click lab.local then New then Organizational Unit
3. Create three OUs named exactly: HR, IT, Finance

**Create Users:**

Right click each OU then New then User:

    OU: HR
    First name: Ali
    User logon name: ali
    Password: choose a strong password and note it down
    Uncheck: User must change password at next logon

    OU: IT
    First name: Ahmed
    User logon name: ahmed
    Password: choose a strong password and note it down
    Uncheck: User must change password at next logon

    OU: Finance
    First name: Sara
    User logon name: sara
    Password: choose a strong password and note it down
    Uncheck: User must change password at next logon

**Create Security Groups:**

Right click each OU then New then Group:

    HR OU — Name: HR_Group — Type: Security — Scope: Global
    IT OU — Name: IT_Group — Type: Security — Scope: Global
    Finance OU — Name: Finance_GRoup — Type: Security — Scope: Global

> ⚠️ Finance_GRoup has a capital R. This exact spelling must be used everywhere.

**Add users to groups:**

Double click each group then Members tab then Add:

    HR_Group — add ali
    IT_Group — add ahmed
    Finance_GRoup — add sara and ali

**Create Service Account:**
1. Right click the Users container — not an OU — then New then User
2. First name: svc-authentik
3. User logon name: svc-authentik
4. Password: choose a strong password and note it down carefully
5. Uncheck must change password at next logon
6. Check password never expires
7. Finish — leave this account as a plain standard user with no extra permissions

> ⚠️ This service account password is the most important credential in the lab.
> You will use this same password in three places: Authentik LDAP source config,
> FreeRADIUS LDAP module config, and any ldapsearch test commands.
> If authentication ever breaks across any component, check this password first.
> It must be identical in Active Directory, Authentik, and FreeRADIUS.

---

## Step 4 — Ubuntu Server VM

**Create the VM:**
- RAM: 4096MB minimum — 6144MB recommended
- Network Adapter: Host Only
- Boot from Ubuntu Server ISO

**During install:**
- When asked about network note the interface name — it will be enp0s8
- Set static IP: 192.168.56.108
- Subnet: 255.255.255.0
- Gateway: 192.168.56.1
- DNS: 192.168.56.109

**After install — install all required software:**

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install docker.io docker-compose-plugin -y
sudo apt install freeradius freeradius-ldap -y
sudo apt install libpam-google-authenticator -y
sudo apt install python3 python3-pip -y
sudo apt install ldap-utils -y
pip3 install flask requests authlib
sudo systemctl enable docker
sudo systemctl start docker
```

---

## Step 5 — Authentik Setup

**Start Authentik containers:**

```bash
mkdir ~/authentik && cd ~/authentik
```

Go to the official Authentik documentation at goauthentik.io and download
their docker-compose.yml file. Place it in this folder. Then run:

```bash
sudo docker compose up -d
```

Wait 60 seconds then open http://192.168.56.108:9000 in a browser.
Complete the setup wizard to create your admin account.

**Configure LDAP Source:**
1. Admin panel then Directory then Federation and Social login
2. Click Create then LDAP Source
3. Name: Active Directory
4. Server URI: ldap://192.168.56.109
5. Bind DN: CN=svc-authentik,CN=Users,DC=lab,DC=local
6. Bind Password: enter the svc-authentik password you set in Step 3
7. Base DN: DC=lab,DC=local
8. User object filter: (objectClass=person)
9. Group object filter: (objectClass=group)
10. Save then click Sync Now

**Create OAuth2 Providers:**

Go to Applications then Providers then Create then OAuth2/OpenID Provider

    HR Provider:
    Name: HR Provider
    Client ID: hr-client
    Client Secret: Authentik generates this — copy and save it for the Flask app
    Redirect URIs: http://192.168.56.108:5001/callback

    IT Provider:
    Name: IT Provider
    Client ID: it-client
    Client Secret: copy and save it
    Redirect URIs: http://192.168.56.108:5002/callback

    Finance Provider:
    Name: Finance Provider
    Client ID: finance-client
    Client Secret: copy and save it
    Redirect URIs: http://192.168.56.108:5003/callback

**Create Applications:**

Go to Applications then Applications then Create:

    HR App — Slug: hr-app — Provider: HR Provider
    IT App — Slug: it-app — Provider: IT Provider
    Finance App — Slug: finance-app — Provider: Finance Provider

**Bind Groups to Applications:**

Click each app then Policy Bindings then Create:

    HR App — bind group — HR_Group
    IT App — bind group — IT_Group
    Finance App — bind group — Finance_GRoup

**Enable Two Factor Authentication:**
1. Flows and Stages then Stages
2. Find default-authentication-mfa-validation then edit it
3. Change Not configured action from Continue to Force configuration
4. Set Configuration Stage to default-authenticator-totp-setup
5. Save

---

## Step 6 — FreeRADIUS Setup

**Configure LDAP module:**

```bash
sudo gedit /etc/freeradius/3.0/mods-available/ldap
```

Find and set these fields — use the svc-authentik password you set in Step 3:

```
server = '192.168.56.109'
identity = 'CN=svc-authentik,CN=Users,DC=lab,DC=local'
password = 'YOUR_SVC_AUTHENTIK_PASSWORD'
base_dn = 'DC=lab,DC=local'
```

Enable the module:

```bash
sudo ln -s /etc/freeradius/3.0/mods-available/ldap /etc/freeradius/3.0/mods-enabled/ldap
```

**Add pfSense as trusted client:**

```bash
sudo gedit /etc/freeradius/3.0/clients.conf
```

Add at the bottom — use the same RADIUS shared secret you set in pfSense Step 2:

```
client pfsense {
    ipaddr = 192.168.56.1
    secret = YOUR_RADIUS_SHARED_SECRET
    shortname = pfsense
}
```

**Enable PAM module:**

```bash
sudo ln -s /etc/freeradius/3.0/mods-available/pam /etc/freeradius/3.0/mods-enabled/pam
```

**Configure PAM for FreeRADIUS:**

```bash
sudo gedit /etc/pam.d/radiusd
```

Add this as the very first line:

```
auth required pam_google_authenticator.so
```

**Add password splitting logic:**

```bash
sudo gedit /etc/freeradius/3.0/sites-enabled/default
```

Find the authorize section and add this block inside it:

```
if ((ok || updated) && &User-Password && !control:Auth-Type) {
    update request {
        &Tmp-String-1 := "%{User-Password}"
    }
    if (&Tmp-String-1 =~ /^(.*)([0-9]{6})$/) {
        update control {
            &Tmp-String-0 := "%{2}"
        }
        update request {
            &User-Password := "%{1}"
        }
    }
    update control {
        &Auth-Type := LDAP
    }
}
```

> ⚠️ This build uses FreeRADIUS 3.2.5 which uses POSIX regex not PCRE.
> Never use \d in regex here — always use [0-9] instead.
> Using \d will silently evaluate as false every time with no error message.

**Restart and test:**

```bash
sudo systemctl restart freeradius
radtest ali YOUR_USER_PASSWORD 127.0.0.1 0 YOUR_LOCALHOST_SECRET
```

You should see Access-Accept in the output.

---

## Step 7 — VPN Two Factor Authentication

**Create Linux user:**

```bash
sudo adduser ali
```

**Generate TOTP secret:**

```bash
sudo su - ali
google-authenticator
```

Answer all questions exactly like this:

```
Make tokens time-based: y
Update the file: y
Disallow reuse of tokens: y
Increase window: n
Enable rate-limiting: y
```

The terminal will show a secret key and a QR code.
Add the secret to your authenticator app and label it VPN-Ali.
Keep this entry completely separate from the web login TOTP entry.

Exit back to your normal user:

```bash
exit
```

**Fix file permissions so FreeRADIUS can read the secret:**

```bash
sudo chmod 644 /home/ali/.google_authenticator
sudo chmod 755 /home/ali
```

> ⚠️ Without these exact permissions FreeRADIUS cannot read the TOTP secret
> and every VPN login will silently fail even with the correct code.

**Test combined login:**

Replace YOUR_PASSWORD with ali's actual password and replace 123456 with
your real current TOTP code — type them together with no space between them:

```bash
radtest ali YOUR_PASSWORD123456 127.0.0.1 0 YOUR_LOCALHOST_SECRET
```

Should return Access-Accept.

---

## Step 8 — Flask Applications

**Create folders:**

```bash
mkdir -p ~/flask-apps/hr
mkdir -p ~/flask-apps/it
mkdir -p ~/flask-apps/finance
```

**HR App:**

```bash
sudo gedit ~/flask-apps/hr/app.py
```

Paste this content — replace the client secret placeholder with the
actual secret copied from Authentik in Step 5:

```python
from flask import Flask, redirect, url_for, session
from authlib.integrations.flask_client import OAuth

app = Flask(__name__)
app.secret_key = 'hr-secret-key'
oauth = OAuth(app)

authentik = oauth.register(
    name='authentik',
    client_id='hr-client',
    client_secret='PASTE_HR_CLIENT_SECRET_FROM_AUTHENTIK',
    server_metadata_url='http://192.168.56.108:9000/application/o/hr-app/.well-known/openid-configuration',
    client_kwargs={'scope': 'openid profile email'},
)

@app.route('/')
def index():
    user = session.get('user')
    if user:
        return '<h1>HR Portal</h1><p>Welcome, ' + user['preferred_username'] + '</p><a href="/logout">Logout</a>'
    return '<h1>HR Portal</h1><a href="/login">Login</a>'

@app.route('/login')
def login():
    return authentik.authorize_redirect(url_for('callback', _external=True))

@app.route('/callback')
def callback():
    token = authentik.authorize_access_token()
    session['user'] = token['userinfo']
    return redirect('/')

@app.route('/logout')
def logout():
    session.clear()
    return redirect('/')

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5001, debug=True)
```

Repeat for IT app — change port to 5002, client_id to it-client, slug to it-app

Repeat for Finance app — change port to 5003, client_id to finance-client, slug to finance-app

**Start all three apps — each in its own terminal:**

```bash
cd ~/flask-apps/hr && python3 app.py
cd ~/flask-apps/it && python3 app.py
cd ~/flask-apps/finance && python3 app.py
```

---

## Step 9 — Windows 10 Client

1. Set static IP: 192.168.56.107
2. Subnet: 255.255.255.0
3. Gateway: 192.168.56.1
4. DNS: 192.168.56.109
5. Install OpenVPN GUI from openvpn.net
6. In pfSense go to VPN then OpenVPN then Client Export
7. Download the client config file
8. Place it in C:\Program Files\OpenVPN\config\
9. Install the Authenticator extension in Chrome
10. Add VPN-Ali TOTP secret to the extension labeled VPN-Ali

---

## Step 10 — Startup Order After Any Reboot

Always start in this exact order and wait for each to fully boot:

    1. Windows Server
    2. pfSense — wait until https://192.168.56.1 loads in browser
    3. Ubuntu Server — then run:

```bash
cd ~/authentik && sudo docker compose up -d
sudo systemctl start freeradius
cd ~/flask-apps/hr && python3 app.py &
cd ~/flask-apps/it && python3 app.py &
cd ~/flask-apps/finance && python3 app.py &
```

    4. Windows 10

---

## ✅ Verification Checklist

**On Ubuntu:**

```bash
# Check all 4 Authentik containers are running
sudo docker ps

# Check FreeRADIUS is running
sudo systemctl status freeradius

# Test basic RADIUS authentication
radtest ali YOUR_USER_PASSWORD 127.0.0.1 0 YOUR_LOCALHOST_SECRET

# Test LDAP connectivity to Active Directory
ldapsearch -x -H ldap://192.168.56.109 \
  -D "CN=svc-authentik,CN=Users,DC=lab,DC=local" \
  -w YOUR_SVC_AUTHENTIK_PASSWORD \
  -b "DC=lab,DC=local" "(sAMAccountName=ali)"
```

**From Windows 10 browser:**

    Open http://192.168.56.108:5001
    Should redirect to Authentik login page

    Log in as ali with password and web login TOTP code
    Should see HR Portal welcome message

    Try http://192.168.56.108:5003 as ali
    Should see access denied — Finance app is not accessible to HR users

**From Windows 10 OpenVPN:**

    Connect using password immediately followed by VPN-Ali TOTP code with no space
    Open Command Prompt and run ipconfig
    Should see OpenVPN adapter with a 10.8.0.x address
    Ping 192.168.56.109 — should succeed through the tunnel

---

## 🔧 Troubleshooting

| Symptom | Most Likely Cause | Fix |
|---|---|---|
| FreeRADIUS fails to start | Duplicate file in sites-enabled folder | Remove any backup files from sites-enabled |
| LDAP authentication fails | svc-authentik password mismatch | Verify password is identical in AD and in ldap module config |
| VPN TOTP codes always rejected | TOTP secret out of sync | Re-enter secret key manually in Chrome Authenticator VPN-Ali entry |
| Finance user gets access denied | Wrong group name in Authentik binding | Confirm binding uses Finance_GRoup with capital R not Finance_Group |
| Regex splitting never triggers | Using PCRE shorthand in POSIX engine | Replace any \d with [0-9] in the unlang config |
| PAM check fails silently | File permission on .google_authenticator | Run chmod 644 on the file and chmod 755 on the home directory |

---

<div align="center">

*Part of the [SOC Journey](../README.md) cybersecurity portfolio*

</div>
