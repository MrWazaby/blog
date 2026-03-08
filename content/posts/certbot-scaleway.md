---
title: "Certbot certificates with Scaleway DNS challenge"
date: 2026-03-08T17:00:00+02:00
draft: false
tags: ["opensource", "tools"]
author: "Alexandre"
---

If you’re running a homelab behind a NAT or a firewall, exposing port 80/443 for the classic HTTP‑01 challenge is often impossible. That’s exactly the situation I ran into when I started consolidating my services on a private network. The good news? Let’s Encrypt also supports **DNS‑01** validation, which lets you prove domain ownership by creating a temporary TXT record. Since my zones live on **Scaleway DNS**, I tweaked the original [`geerlingguy/ansible-role-certbot`](https://github.com/geerlingguy/ansible-role-certbot) to talk to Scaleway’s API out‑of‑the‑box. The result is the [`ansible‑role‑certbot‑scaleway`](https://codeberg.org/wazaby/ansible-role-certbot-scaleway) role.

Below is the step‑by‑step recipe I use to spin up fresh certificates on a completely isolated host, without ever opening HTTP ports.

## What you’ll need

| Component              | What to do                                                                                                                     |
|------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| **Ansible**            | Install ≥ 2.9 on your control machine (`pip install ansible`).                                                                 |
| **Target host**        | Any modern Linux distro (Ubuntu 20.04+, Debian 11+).                                                                           |
| **Scaleway account**   | Your domain must be managed in Scaleway DNS.                                                                                   |
| **Scaleway API token** | Create a *Domain*‑scoped token in the Scaleway console → *Security → API Keys*. Keep it safe; we’ll stash it in Ansible Vault. |
| **Root / sudo**        | Required for installing Certbot and setting up the renewal cron job.                                                           |


## Quick peek at the role

The fork lives at <https://codeberg.org/wazaby/ansible-role-certbot-scaleway>. Its features are:

* **Installation methods** – `package`, `snap`, or `source`.  
* **Renewal automation** – configurable cron schedule (`certbot_auto_renew_*`).  
* **Certificate creation** – `standalone`, `webroot`, **and** `dns‑scaleway`.  
* **Service stop/reload hooks** – pause services while Certbot runs, then bring them back up.  
* **Wildcard support** – thanks to DNS‑01.


## Pull the role into your repo

```yaml
---
roles:
  - name: ansible-role-certbot-scaleway
    scm: git
    src: https://codeberg.org/wazaby/ansible-role-certbot-scaleway.git
    version: 5.5.0
```

```bash
ansible-galaxy install -r requirements.yml
```

## Add the Scaleway token into your Vault

If you do not have an Ansible vault or any other secret engine you can:

```bash
ansible-vault create path/to/your/vault.yml
```

Inside it:
```yaml
certbot_dns_scaleway_api_token: YOUR_SCW_API_TOKEN
```

## Add your certificates variable

On your host_vars you can request one (or many) certificate(s) like that:
```yaml
# certbot
certbot_create_if_missing: true
certbot_create_method: dns-scaleway
certbot_admin_email: YOUR_EMAIL
certbot_certs:
  - domains:
    - domain.tld
    - www.domain.tld
  - domains:
    - another.domain.tld
```

## Call the role in your playbook

You can then call the role in your playbook to generate certificates. Do not forget to iunclude variables from your vault:

```yaml
- name: 'Create certificates'
  hosts: CHANGE_ME
  become: true
  vars_files:
    - "path/to/your/vault.yml"
  roles:
    - role: ansible-role-certbot-scaleway
      tags: certs
```

## Run the playbook

```bash
ansible-playbook -i inventory your_playbook.yml --ask-vault-pass 
```

What the role does behind the scenes:
1. Installs Certbot with the packages from your OS.
2. Installs the certbot-dns-scaleway plugin using pip.
3. Loops over each entry in certbot_certs and call certbot to request the certs.
4. Creates a cron job that runs certbot renew.
5. Stop/start any services you listed (or nginx by default).

## Verify everything worked

### Inspect the files 

```bash
sudo ls -l /etc/letsencrypt/live/domain.tld/
```
You’ll find cert.pem, privkey.pem, fullchain.pem, etc.

### Test renewal (dry‑run)

```bash
sudo certbot renew --dry-run
```
If the dry‑run succeeds: good! The nightly cron will take care of real renewals automatically.

---

That’s all there is to it. With just a few lines of YAML you can secure every service in a closed‑off homelab using your Scaleway domain.

[Comment this article on Mastodon.](https://h4.io/@wazaby/116194455839927717)
