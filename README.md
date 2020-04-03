Ansible Docker mailserver
=========================
Ansible role is based on top of project [tomav/docker-mailserver](https://github.com/tomav/docker-mailserver).

Role will create all needed prerequisites to run mail server in Docker but doesn't install the Docker itself.

All data are stored on host to prevent data loss during Docker restart.

Currently supported
-------------------
- Account management
  - [x] Multiple accounts
  - [ ] LDAP
- Let's Encrypt Key renew automation
  - [x] Standalone
  - [ ] [Nginx Proxy](https://github.com/tomav/docker-mailserver/wiki/Configure-SSL#example-using-docker-nginx-proxy-and-letsencrypt-nginx-proxy-companion)
- Host OS support
  - [x] Debian based
  - [ ] Red Hat based
- Managable features
  - [ ] ClamAV
  - [ ] Fail2Ban
  - [x] OpenDKIM
  - [x] OpenDMARC
  - [ ] Postgrey
  - [ ] Spamassasin

Test environment
----------------
- Ubuntu 18.04 LTS

Variables
=========
| Variable name         | Default     | Description
| --------------------- | ----------- | --------------------------------------
| `mail_accounts`       | `[]`        | List of mail accounts according to [Mail account format](#Mail_Account_format)
| `mail_domains`        | `[]`        | List of mail domains (the first one should be your MX but certificate will be issued for all of them)
| `mail_cert_email`     | `""`        | Email used for Let's Encrypt account
| `mail_persist_folder` | `/opt/mail` | (optional) Persistent folder for mail data, configuration, etc.

Mail account format
===================
Each account in variable `mail_accounts` has these parameters:
- `username` - part of email before "@", eg. `user1`
- `domain` - part of email after "@", eg. `example.com`
- `password` - user password in plaintext
- `aliases` - list of aliases, each alias must be full email
- `restrict` - list of restrictions, may be `send` for suppress mail sending or `receive` for suppress mail receiving

Example
-------
```
mail_accounts:
  - username: user1
    domain: example.com
    password: aaaaa
    aliases:
      - admin@example.com
      - abuse@example.com
    restrict: []
```

Quick Start
===========
Install this Ansible role within the context of playbook
```
ansible-galaxy install hmlkao.docker_mailserver
```

Then create your playbook yaml
```
- name: Localhost installation
  hosts: localhost
  roles:
    - role: hmlkao.docker_mailserver
  vars:
    mail_accounts:
      - username: user1
        domain: example.com
        password: aaaaa
        aliases:
          - admin@example.com
          - abuse@example.com
        restrict: []
      - username: no-reply
        domain: example.com
        password: bbbbb
        aliases: []
        restrict:
          - receive
    mail_domains:
      - server1.example.com
    mail_cert_email: my-mail@somewhere.com
```

What next?
==========
Suppose that your server host is `server1.example.com`

1. Add MX record to your DNS
    ```
    example.com IN     MX      "1 server1.example.com."
    ```
1. Add TXT record with SPF to your DNS
    ```
    example.com IN     TXT     "v=spf1 a mx ~all"
    ```
1. Add TXT record with DKIM to your DNS
    - You can find DKIM key in folder defined by `mail_persist_folder` variable in file `config/opendkim/keys/<domain.tld>/mail.txt` on your host (server)
    - TXT record should look something like
      ```
      mail._domainkey IN     TXT     "v=DKIM1; h=sha256; k=rsa; p=MIIB...jwfx"
      ```
      There should be really `mail._domainkey` for all, NO `server1._domainkey` or whatever else
1. Add TXT record with DMARC to your DNS
    ```
    _DMARC IN     TXT     "v=DMARC1; p=quarantine; pct=5; rua=mailto:abuse@example.com; ruf=mailto:abuse@example.com; fo=1"
    ```
1. Configure reverse DNS for host public IP
    - Ask your provider to configure reverse DNS
    - You should get something similar
      ```
      # dig -x 1.2.3.4 +short
      server1.example.com.
      ```

Test your configuration
=======================
There is many tools to test your mail servre, eg.:
  - https://mxtoolbox.com
    - Test your SMTP configuration
    - expecially [Deliverability test](https://mxtoolbox.com/deliverability) is useful (test SPF and DKIM)
  - https://www.checktls.com
    - Test connection to your SMTP port
    - Fill your `domain.tld` (eg. example.com) to the field
  - [OpenSSL test](https://github.com/tomav/docker-mailserver/wiki/Configure-SSL#testing-certificate)

More complex example
====================
First we would create file `host_vars/mail_accounts.yml` and encrypt it by `ansible-vault`
```
# ansible-vault edit host_vars/mail_accounts.yml
mail_accounts:
  - username: user1
    domain: example.com
    password: aaaaa
    aliases:
      - admin@example.com
      - abuse@example.com
    restrict: []
  - username: no-reply
    domain: example.com
    password: bbbbb
    aliases: []
    restrict:
      - receive
```

Then we can use this file in playbook

```
- name: More complex installation
  hosts: server1.example.com
  roles:
    - role: hmlkao.docker_mailserver
  vars:
    mail_domains:
      - server1.example.com                 <-- my server
      - mail.example.com                    <-- pretty name for connection from clients
    mail_cert_email: my-mail@somewhere.com  <-- notification mail for Let's Encrypt
```

Client configuration
====================
Suppose your mail server is running on domain `server1.example.com` and we have some pretty CNAME like `mail.example.com` which points to our mail server and account with username `user1` is created.

Evolution (Default Gnome client)
--------------------------------
Version: 3.28.5-0ubuntu0.18.04.1

1. Edit > Accounts
1. Add "Mail Account"
1. Identity
  1. Email Address: `user1@example.com`
1. Receiving Email
  1. Server Type: IMAP
  1. Server: `mail.example.com`
  1. Username: `user1@example.com`
  1. Encryption method: TLS on dedicated port
    - Port should change to 993
  1. Authentication: Password
1. Sending Email
  1. Server Type: SMTP
  1. Server: `mail.example.com`
  1. Port: 587
  1. Check "Server requires authentication"
  1. Encryption method: STARTTLS after connecting
  1. Authentication:
    - Type: Login
    - Username: `user1@example.com`

Gmail app for Android
---------------------
TBD

Thunderbird (Mozilla client)
----------------------------
TBD

Mail (Default Windows 10 client)
--------------------------------
TBD

Under the hood
==============
Ports
-----
There are published only three ports
- **TCP/25** - SMTP port for receiving emails from other SMTP servers
- **TCP/587** - SMTP Submission port for sending emails from clients
- **TCP/993** - IMAP port for downloading emails to clients

Running server
--------------
Docker container named `mail-server` is configured to always restart even after server boot.

Persistent storage
------------------
All data (mail data, certificates, configuration) are stored on the host in folder defined in Ansible variable `mail_persist_folder`.

Certificates
------------
TLS certificates are issued by Let's Encrypt issuer by `certbot/certbot` Docker image according to [these instructions](https://github.com/tomav/docker-mailserver/wiki/Configure-SSL).

Ansible role creates `systemd` renew *service* and *timer* which will run the service once per day. No cronjob configuration needed.

Mail accounts
-------------
They are generated simply to file according to [these instructions](https://github.com/tomav/docker-mailserver/wiki/Configure-Accounts).

Troubleshooting
===============
You can see what happen in Docker logs on host
```
docker logs mail-server
```

**When you found some bug in the role create an [issue on GitHub(https://github.com/hmlkao/ansible-docker-mailserver/issues)] please.**

Contributing
============
All helping hands are appreciated. Check [CONTRIBUTING.md](https://github.com/hmlkao/ansible-docker-mailserver/blob/master/CONTRIBUTING.md)
