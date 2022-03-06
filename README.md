Ansible Docker mailserver
=========================
Ansible role is based on top of project [docker-mailserver/docker-mailserver](https://github.com/docker-mailserver/docker-mailserver).

Role will create all needed prerequisites to run mail server in Docker but doesn't install the Docker itself.

All data are stored on host to prevent data loss during Docker restart.

Currently supported
-------------------
- Account management
  - [x] Multiple accounts
  - [ ] Multiple domains
  - [ ] LDAP
- Let's Encrypt Key renew automation
  - [x] Standalone
  - [ ] [Nginx Proxy](https://docker-mailserver.github.io/docker-mailserver/edge/config/security/ssl/#example-using-nginx-proxy-and-acme-companion-with-docker)
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
| `mail_accounts`       | `[]`        | List of mail accounts according to [Mail account format](https://github.com/hmlkao/ansible-docker-mailserver#mail-account-format)
| `mail_domains`        | `[]`        | List of mail domains (the first one should be your MX however certificate will be issued for all of them)
| `mail_cert_email`     | `""`        | Email used for Let's Encrypt account
| `mail_persist_folder` | `/opt/mail` | (optional) Persistent folder for mail data, configuration, etc. on host
| `mail_docker_image`   | `docker.io/mailserver/docker-mailserver` | (optional) Shouldn't be changed
| `mail_docker_tag`     | `latest`    | (optional) Image Docker tag used to run mail server
| `mail_dkim_size`      | `2048`      | (optional) DKIM key size (available values are 1024, 2048, 4096)
| `mail_amavis_config`  | `""`        | (optional) Configure Amavis overrides

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
---
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
      - Beware of that the record is divided by 250 characters so you have to [concat it together](https://docker-mailserver.github.io/docker-mailserver/edge/config/best-practices/dkim/#configuration-using-a-web-interface)
    - TXT record should look something like
      ```
      mail._domainkey IN     TXT     "v=DKIM1; h=sha256; k=rsa; p=MIIB...jwfx"
      ```
      - `mail` is DKIM selector, not subdomain so there have to be `mail._domainkey` for all, NO `server1._domainkey` or whatever else
      - Custom selector [cannot be used](https://github.com/docker-mailserver/docker-mailserver/issues/1304).
1. Add TXT record with [DMARC](https://dmarc.org/) to your DNS
    What happen with mails which doesn't meet SPF and DKIM validation.
    ```
    _DMARC IN     TXT     "v=DMARC1; p=quarantine; pct=100; rua=mailto:abuse+rua@example.com; ruf=mailto:abuse+ruf@example.com; fo=1"
    ```
    - `ruf` - Reporting URI for forensic reports
    - `rua` - Reporting URI of aggregate reports
    - Verify in commandline if is TXT applied
      ```
      dig mail._domainkey.example.com TXT
      ```
1. Configure reverse DNS for host public IP
    - Ask your provider to configure reverse DNS
    - You should get something similar (where `<1.2.3.4>` is your public server IP)
      ```
      # dig -x <1.2.3.4> +short
      server1.example.com.
      ```

Test your configuration
=======================
There are many tools to test your mail server, eg.:
- [dkimvalidator.com](https://dkimvalidator.com)
  - Generate email address which you can use to verify your SPF/DKIM config
- check-auth@verifier.port25.com
  - Just send mail to this mail address from your account mail and your will receive report back to your mail address
- [mxtoolbox.com](https://mxtoolbox.com)
  - Test your SMTP configuration
  - Especially [Deliverability test](https://mxtoolbox.com/deliverability) is useful (test SPF and DKIM)
  - Just send mail to ping@tools.mxtoolbox.com from your account mail and your will receive report back to your mail address
  - NOTE: MXToolBox DKIM validation fails (eg. [docker-mailserver/docker-mailserver](https://github.com/docker-mailserver/docker-mailserver/issues/1172), [serverfault.com](https://serverfault.com/questions/1005818/dkim-validating-but-mxtoolbox-reports-as-dkim-signature-not-verified) even if another validators works well, don't know why but it looks that Google DKIM validation fails too (just send mail to any Gmail adrress and you will get report to mail according to your [DMARC configuration](https://github.com/hmlkao/ansible-docker-mailserver#what-next))
- [checktls.com](https://www.checktls.com/TestReceiver)
  - Test connection to your SMTP port
  - Fill your `domain.tld` (eg. `example.com`) to the field
- [Google Postmaster tools](https://toolbox.googleapps.com/apps/checkmx/)
  - Fill your `domain.tld` (eg. `example.com`) to the domain field
  - Fill `mail` to DKIM selector field
  - **Doesn't work for me from unknown reason**
- [Google Toolbox](https://toolbox.googleapps.com/)
  - Mail troubleshooting
- Test via [OpenSSL](https://docker-mailserver.github.io/docker-mailserver/edge/config/security/ssl/#testing-a-certificate-is-valid)

More complex example
====================
1. Create file `host_vars/server1.example.com.yml` encrypted by [`ansible-vault`](https://docs.ansible.com/ansible/latest/user_guide/vault.html) first.
    ```
    # ansible-vault edit host_vars/server1.example.com.yml
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
1. Create file `vault-pass.txt` with your Ansible Vault (add this file to `.gitignore`)
    ```
    # echo 'my-secret-vault-password' > vault-pass.txt
    ```
1. Set up Ansible with ansible.cfg
    ```
    # cat > ansible.cfg <<EOF
    [defaults]
    vault_password_file = vault-pass.txt
    EOF
    ```
1. Variable `mail_accounts` is then available in the role for host `server1.example.com`.
    ```
    ---
    - name: More complex installation
      hosts: server1.example.com
      roles:
        - role: hmlkao.docker_mailserver
      vars:
        mail_domains:
          - server1.example.com                 <-- my MX server
          - mail.example.com                    <-- pretty name for email clients
        mail_cert_email: my-mail@somewhere.com  <-- notification mail for Let's Encrypt
        mail_persist_folder: /usr/local/mail    <-- path to folder on your host
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
   - Email Address: `user1@example.com`
1. Receiving Email
   - Server Type: IMAP
   - Server: `mail.example.com`
   - Username: `user1@example.com`
   - Encryption method: TLS on dedicated port
     - Port should change to 993
   - Authentication: Password
1. Sending Email
   - Server Type: SMTP
   - Server: `mail.example.com`
   - Port: 587
   - Check "Server requires authentication"
   - Encryption method: STARTTLS after connecting
   - Authentication:
     - Type: Login
     - Username: `user1@example.com`

Gmail app for Android
---------------------
Version: 2020.03.01.300951155.release

1. Add another account
1. Other
1. Fill your email > Manual setup
1. What type of account is this?: Personal (IMAP)
1. Fill your password > Next
1. Server: `mail.example.com`
1. IMAP port: 465 (SSL/TLS)
1. Outgoing server
   - Server: `mail.example.com`

You would be able to receive/send mails now.

Mail (Default MacOS client)
---------------------------
Version: 15.0 (3693.40.0.1.81) / 2022-02-14

1. Choose a Mail account provider... > Other Mail Account...
1. Fill
    * Name: < your name >
    * Email address: < the whole email address >
    * Password: < password to your mail account >
1. When is error message shown
    * Choose "IMAP"
    * Fill address of your mail server > according to MX record
1. Sign in

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
TLS certificates are issued by Let's Encrypt issuer by `certbot/certbot` Docker image according to [these instructions](https://docker-mailserver.github.io/docker-mailserver/edge/config/security/ssl/#example-using-docker-for-lets-encrypt).

This role creates `systemd` renew *service* and *timer* which will run the service once per day. No cronjob configuration needed.

Mail accounts
-------------
They are generated directly to file `postfix-accounts.cf` according to [these instructions](https://docker-mailserver.github.io/docker-mailserver/edge/config/user-management/accounts/).

Troubleshooting
===============
You can see what happen in Docker logs on host
```
docker logs mail-server
```

**When you found some bug in the role create an [issue on GitHub](https://github.com/hmlkao/ansible-docker-mailserver/issues) please.**

Sources
=======
- [RSA sign and verify using Openssl : Behind the scene](https://medium.com/@bn121rajesh/rsa-sign-and-verify-using-openssl-behind-the-scene-bf3cac0aade2)
  - Interesting article about how RSA signing (used by DKIM) works

Contributing
============
All helping hands are appreciated. Check [CONTRIBUTING.md](https://github.com/hmlkao/ansible-docker-mailserver/blob/master/CONTRIBUTING.md)
