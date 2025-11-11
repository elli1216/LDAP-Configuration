# OpenLDAP Configuration

This guide documents the installation and configuration of an **OpenLDAP server** on **Ubuntu Server** and its integration with an **Ubuntu Client** for centralized authentication.

---

## Prerequisites

<div align="center">

<details>
<summary><b>Important: Complete Prerequisites First</b></summary>

<br>

Before proceeding with this LDAP configuration guide, please ensure you have completed the server and client setup:

**[Linux Ubuntu Server and Client Setup](https://github.com/elli1216/ubuntu-server-and-client-setup)**

This repository contains essential Ubuntu server and client setup commands that must be completed before configuring LDAP.

</details>

</div>

---

## Overview

<div align="left">

OpenLDAP (Lightweight Directory Access Protocol) allows centralized management of users, groups, and authentication across multiple systems.

This setup includes:

<ul>
  <li><strong>Ubuntu Server</strong> as the LDAP server</li>
  <li><strong>Ubuntu Client</strong> configured to authenticate via the LDAP server</li>
  <li><strong>Networking & DNS/NAT configuration</strong> for client connectivity</li>
</ul>

</div>

---

## Server Configuration (Ubuntu Server)

### **1. Install OpenLDAP**

<div align="left">

```bash
sudo apt install slapd ldap-utils -y
```

</div>

<blockquote>
<strong>What this does:</strong> Installs the <strong>LDAP server (slapd)</strong> and essential <strong>LDAP utilities</strong>.
</blockquote>

---

### **2. Configure LDAP Server**

<div align="left">

```bash
sudo dpkg-reconfigure slapd
```

</div>

<div align="left">

<strong>During setup:</strong>

<ul>
  <li>Define the <strong>Base DN</strong>, e.g. <code>dc=dar1,dc=lan</code></li>
  <li>Set the <strong>Admin password</strong></li>
  <li>Skip database removal prompts if upgrading</li>
</ul>

</div>

---

### **3. Create Organizational Units**

<div align="left">

<strong>Step 1:</strong> Create a new file:

```bash
nano base.ldif
```

<strong>Step 2:</strong> Add the following content:

```ldif
dn: ou=People,dc=dar1,dc=lan
objectClass: organizationalUnit
ou: People

dn: ou=Groups,dc=dar1,dc=lan
objectClass: organizationalUnit
ou: Groups
```

<strong>Step 3:</strong> Add the base structure to LDAP:

```bash
sudo ldapadd -x -D cn=admin,dc=dar1,dc=lan -W -f base.ldif
```

</div>

---

### **4. Create a User**

<div align="left">

<strong>Step 1:</strong> Generate a hashed password:

```bash
slappasswd
```

<blockquote>
<strong>Example output:</strong>
<pre><code>{SSHA}9BRR7nK3W8cGd3i6R1Npz+12Hg4bZ5L/</code></pre>
</blockquote>

<strong>Step 2:</strong> Create the user definition file:

```bash
nano user.ldif
```

<strong>Step 3:</strong> Add the following content (replace the password hash with your generated one):

```ldif
dn: uid=lee1user,ou=People,dc=dar1,dc=lan
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: Lee One
sn: User
uid: lee1user
uidNumber: 10000
gidNumber: 10000
homeDirectory: /home/lee1user
loginShell: /bin/bash
userPassword: {SSHA}9BRR7nK3W8cGd3i6R1Npz+12Hg4bZ5L/
```

<strong>Step 4:</strong> Add the user to LDAP:

```bash
sudo ldapadd -x -D cn=admin,dc=dar1,dc=lan -W -f user.ldif
```

</div>

---

### **5. Open Firewall Port**

<div align="left">

```bash
sudo ufw allow 389/tcp
```

</div>

<blockquote>
<strong>Purpose:</strong> Allows LDAP traffic through <strong>port 389</strong> for client connections.
</blockquote>

---

## Client Configuration (Ubuntu Client)

### **1. Install LDAP Client Packages**

<div align="left">

```bash
sudo apt install libnss-ldap libpam-ldap ldap-utils nscd -y
```

</div>

<blockquote>
<strong>What this does:</strong> Installs necessary modules for <strong>NSS</strong> and <strong>PAM</strong> to support LDAP authentication.
</blockquote>

---

### **2. Configure LDAP Connection**

<div align="left">

<strong>When prompted during installation, enter:</strong>

<table>
  <tr>
    <th>Parameter</th>
    <th>Value</th>
  </tr>
  <tr>
    <td><strong>URI</strong></td>
    <td><code>ldap://10.10.10.1/</code></td>
  </tr>
  <tr>
    <td><strong>Base DN</strong></td>
    <td><code>dc=dar1,dc=lan</code></td>
  </tr>
</table>

</div>

<blockquote>
<strong>Purpose:</strong> Links the client to the server's directory service.
</blockquote>

---

### **3. Update NSS Configuration**

<div align="left">

```bash
sudo nano /etc/nsswitch.conf
```

<strong>Edit the following lines to include <code>ldap</code>:</strong>

```text
passwd:         files systemd sss ldap
group:          files systemd sss ldap
shadow:         files systemd sss ldap
gshadow:        files systemd ldap
```

</div>

---

### **4. Enable Auto Home Directory Creation**

<div align="left">

```bash
sudo nano /etc/pam.d/common-session
```

<strong>Add this line at the bottom:</strong>

```text
session optional pam_mkhomedir.so skel=/etc/skel/ umask=0022
```

</div>

<blockquote>
<strong>Purpose:</strong> Ensures LDAP users get a home folder (e.g., <code>/home/lee1user</code>) upon first login.
</blockquote>

---

### **5. Restart LDAP Services**

<div align="left">

```bash
sudo systemctl restart nscd
```

</div>

<blockquote>
<strong>Purpose:</strong> Applies the new configurations.
</blockquote>

---

## Network & Internet Access Configuration

### **Routing Setup (on the Server)**

<div align="left">

<strong>Step 1:</strong> Enable IP forwarding:

```bash
sudo sysctl -p
```

<strong>Step 2:</strong> Add a NAT rule for internet sharing:

```bash
sudo iptables -t nat -A POSTROUTING -s 10.10.10.0/24 -o enp0s3 -j MASQUERADE
```

<strong>Step 3:</strong> Save iptables rules persistently:

```bash
sudo sh -c "iptables-save > /etc/iptables/rules.v4"
```

<blockquote>
<strong>Purpose:</strong> Manually saved all the new rules to the persistent file.
</blockquote>

<strong>Step 4:</strong> Fix Server's Own DNS:

```bash
sudo nano /etc/resolv.conf
```

<strong>Add the following line:</strong>

```text
nameserver 8.8.8.8
```

<blockquote>
<strong>Purpose:</strong> Opened the server's DNS config. We added <code>nameserver 8.8.8.8</code> so the server itself could find public domains.
</blockquote>

<strong>Step 5:</strong> Fix dnsmasq Forwarding:

```bash
sudo nano /etc/dnsmasq.conf
```

<strong>Add the following line:</strong>

```text
server=8.8.8.8
```

<blockquote>
<strong>Purpose:</strong> Edited the <code>dnsmasq</code> config to fix "Temporary failure" errors. We added <code>server=8.8.8.8</code> to force <code>dnsmasq</code> to use Google for public lookups.
</blockquote>

<strong>Step 6:</strong> Apply Configs:

```bash
sudo systemctl restart dnsmasq
```

<blockquote>
<strong>Purpose:</strong> Restarted <code>dnsmasq</code> to apply the new <code>server=8.8.8.8</code> rule.
</blockquote>

```bash
sudo reboot
```

<blockquote>
<strong>Purpose:</strong> A full server reboot was needed to correctly load all rules (<code>ip_forward</code> and <code>iptables</code>).
</blockquote>

</div>

<blockquote>
<strong>Overall Purpose:</strong> Allows internal clients (<code>10.10.10.0/24</code>) to access the internet through the server's external interface.
</blockquote>

---

### **Client DNS Fix**

<div align="left">

<strong>If the client can't resolve domains, fix DNS:</strong>

```bash
sudo systemctl stop systemd-resolved
sudo rm /etc/resolv.conf
sudo nano /etc/resolv.conf
```

<strong>Add the following line:</strong>

```text
nameserver 10.10.10.1
```

</div>

<blockquote>
<strong>Purpose:</strong> Ensures the client uses the server for DNS resolution.
</blockquote>

---

### **Server DNS Forwarding**

<div align="left">

<strong>Tell the DNS service (<code>dnsmasq</code>) where to forward external requests:</strong>

```bash
sudo nano /etc/dnsmasq.conf
```

<strong>Add the following line:</strong>

```text
server=8.8.8.8
```

</div>

---

### **Set Up `nslcd` (Step-by-Step) on the Client VM**

<div align="left">

<strong>Step 1:</strong> Install the <code>nslcd</code> Package

This was the step we just completed. We had to fix the client's internet access first, then kill the <code>apt</code> lock.

```bash
# On the CLIENT VM
sudo apt install nslcd
```

<strong>Step 2:</strong> Run the Interactive Configuration Wizard

As soon as <code>nslcd</code> finished installing, a purple "Package configuration" screen appeared. This wizard automatically creates the configuration file for <code>nslcd</code> (which is located at <code>/etc/nslcd.conf</code>).

We gave it this information:

<ol>
  <li><strong>LDAP server URI:</strong> <code>ldap://10.10.10.1/</code></li>
  <li><strong>LDAP search base:</strong> <code>dc=dar1,dc=lan</code></li>
  <li><strong>LDAP version:</strong> <code>3</code></li>
  <li><strong>LDAP root account:</strong> <code>cn=admin,dc=dar1,dc=lan</code></li>
  <li><strong>LDAP root password:</strong> <code>(Your-LDAP-Admin-Password)</code></li>
</ol>

<strong>Step 3:</strong> Restart All Client Services

This is the final step in the <code>ldap_client_setup.md</code> guide. To make sure all services (including <code>nslcd</code>) are working together, we restart them.

```bash
# On the CLIENT VM
sudo systemctl restart nslcd
sudo systemctl restart nscd
```

</div>

<blockquote>
<strong>Purpose:</strong> <code>nslcd</code> is now installed, configured with your server's details, and running.

You can <strong>verify it's working</strong> with this command:

```bash
# On the CLIENT VM
sudo systemctl status nslcd
```

It should now show <code>active (running)</code> in green.

</blockquote>

---

## Verification

<div align="left">

### **Test LDAP Lookup (on the Client)**

```bash
getent passwd lee1user
```

<blockquote>
<strong>Expected output:</strong>
<pre><code>lee1user:x:10000:10000:Lee One:/home/lee1user:/bin/bash</code></pre>
</blockquote>

### **Test Login**

You should now be able to log in as <code>lee1user</code> on the client.

</div>

---

## phpLDAPadmin Installation and Configuration

phpLDAPadmin is an extremely useful tool because it lets you manage your LDAP users, groups, and permissions using a simple web interface instead of complicated `.ldif` files and command-line tools.

Since your server is already routing traffic (we fixed that networking issue!), this setup should go smoothly.

Here is the step-by-step process to install and configure phpLDAPadmin on your **Ubuntu Server VM** and access it from your **Ubuntu Client VM**.

---

### **1. Install phpLDAPadmin and Apache Web Server**

You will run these commands on your **Ubuntu Server VM** using SSH.

<div align="left">

```bash
# On the SERVER VM:

# 1. Install phpLDAPadmin. This also installs the Apache web server and PHP dependencies.
sudo apt install phpldapadmin -y

# 2. Check the Apache web server status. It should be running immediately.
sudo systemctl status apache2

# 3. Open the firewall for HTTP traffic (port 80)
# This allows your client's web browser to access the interface.
sudo ufw allow 80/tcp
```

</div>

---

### **2. Configure phpLDAPadmin for Your Domain**

We need to tell the web interface your LDAP domain name (`dar1.lan`).

<div align="left">

```bash
# On the SERVER VM:

# Open the main configuration file for phpLDAPadmin
sudo nano /etc/phpldapadmin/config.php
```

</div>

You need to find the configuration section that defines the host and base domain.

1. Find the lines that start with `$servers->setValue('server','host',...`
2. Find the lines that start with `$servers->setValue('server','base',...`

Modify those lines to reflect your environment (using `10.10.10.1` and `dar1.lan`):

```php
// Old (default) line:
// $servers->setValue('server','host','127.0.0.1');

// New line: Change the host to your static IP address:
$servers->setValue('server','host','10.10.10.1');


// Old (default) line:
// $servers->setValue('server','base',array('dc=example,dc=com'));

// New line: Change the base to your domain components (dc=dar1,dc=lan):
$servers->setValue('server','base',array('dc=dar1,dc=lan'));
```

Save and exit `nano` (`Ctrl+O`, `Enter`, `Ctrl+X`).

---

### **3. Allow Client Access (Disable Security)**

By default, Apache is set to only allow connections from the local host (`127.0.0.1`) to the phpLDAPadmin interface. We need to tell it to allow connections from your entire internal network (`10.10.10.0/24`).

<div align="left">

```bash
# On the SERVER VM:

# Open the Apache configuration file for phpldapadmin
sudo nano /etc/apache2/conf-available/phpldapadmin.conf
```

</div>

Find the following block of code (it should be near the top). It starts with `<Directory /usr/share/phpldapadmin>`.

Add the Require all granted line right after AllowOverride None so the block looks exactly like this:

```
<Directory /usr/share/phpldapadmin/htdocs/>
    DirectoryIndex index.php
    Options +FollowSymLinks
    AllowOverride None
    Require all granted  <-- ADD THIS LINE HERE
```

Save and exit `nano`.

---

### **4. Restart Apache and Access the GUI**

<div align="left">

```bash
# On the SERVER VM:

# Restart the web server to apply the changes
sudo systemctl restart apache2
```

</div>

### **Demonstration (On the Ubuntu Client VM)**

1. **On your Ubuntu Client VM**, open the web browser (e.g., Firefox).

2. In the address bar, type the following URL (using your server's static IP):

   `http://10.10.10.1/phpldapadmin`

3. The phpLDAPadmin interface should now load.

4. On the left, click **"login"**.

   - **Login DN:** `cn=admin,dc=dar1,dc=lan`
   - **Password:** The administrator password you set during the initial `slapd` setup.

You will now be logged in and can see your `ou=People` and `ou=Groups` containers, and manage your user accounts like `lee1user` through the graphical interface.

---

## Summary

<div align="center">

<table>
  <tr>
    <th>Role</th>
    <th>System</th>
    <th>Key Tasks</th>
  </tr>
  <tr>
    <td><strong>Server</strong></td>
    <td>Ubuntu Server</td>
    <td>Install & configure slapd, create base OUs, add user entries, open firewall</td>
  </tr>
  <tr>
    <td><strong>Client</strong></td>
    <td>Ubuntu Client</td>
    <td>Install libnss/pam modules, configure NSS & PAM, link to LDAP, enable home auto-creation</td>
  </tr>
  <tr>
    <td><strong>Network</strong></td>
    <td>Both</td>
    <td>Configure NAT, IP forwarding, and DNS for proper connectivity</td>
  </tr>
</table>

</div>

---

## Notes

<div align="left">

<ul>
  <li>Base DN used in this guide: <strong><code>dc=dar1,dc=lan</code></strong></li>
  <li>Example user: <strong>lee1user</strong></li>
  <li>LDAP Port: <strong>389 (TCP)</strong></li>
  <li>Verify all IPs match your network setup</li>
  <li>Restart services after each major configuration step</li>
</ul>

</div>

---

## References

<div align="left">

<ul>
  <li><a href="https://ubuntu.com/server/docs/service-ldap">Ubuntu OpenLDAP Server Guide</a></li>
  <li><a href="https://wiki.debian.org/LDAP/PAM">Debian LDAP Client Configuration</a></li>
  <li><a href="https://help.ubuntu.com/community/UFW">Linux UFW Firewall Documentation</a></li>
</ul>

</div>

---

<div align="center">

<strong>Author:</strong> <em>Darl Ellison Floresca</em><br>
<strong>Project:</strong> LDAP Configuration<br>

</div>
