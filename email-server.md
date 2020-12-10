# Email server setup and management

### Terminology

- **MTA** The Mail Transfer Agent relays mail between our server and the wider Internet, whether it's delivering an outside email to one of our users, or sending an email from one of our users. Accepted incoming mail gets added to the MTA's queue on the server.
- **MDA** The Mail Delivery Agent takes mail from the MTA's queue and saves it to individual mailboxes on our server.
- **IMAP Server**: Manages users and their mailboxes as they check their email over IMAP connections.

## DNS & Stuff

### Certificates

#### DKIM

There are two main certificates that need managing for this system. This first
is the DKIM certificate.

```
mkdir /etc/dkim # Only if its not present already

openssl genrsa -out /etc/dkim/example.com.dkim.key 1024
openssl rsa -in /etc/dkim/example.com.dkim.key -pubout -out /etc/dkim/example.com.dkim.pub

chmod 0440 /etc/dkim/example.com.dkim.key
chown root:_rspamd /etc/dkim/example.com.dkim.key

cat /etc/dkim/example.com.dkim.pub

-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDThHqiM610nwN1nmV8OMc7PaPO
uJWVGRDz5bWj4cRjTTmQYjJQd02xrydRNrRhjEKBm2mMDArNWjoM3jvN04ZifqJx
DmKr7X8jsYi+MPEHz6wZxB8mayDK6glYTCyx//zl1luUdvm26PutA38K8cgnb7iT
kfVP2OqK6sHAdXjnowIDAQAB
-----END PUBLIC KEY-----
```

Then we can create the DNS TXT record by extracting the public key out of the armor delimiters and formatting the content so it displays as follows:

```dns
dkim._domainkey.example.com. IN TXT "v=DKIM1;k=rsa;p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDThHqiM610nwN1nmV8OMc7PaPOuJWVGRDz5bWj4cRjTTmQYjJQd02xrydRNrRhjEKBm2mMDArNWjoM3jvN04ZifqJxDmKr7X8jsYi+MPEHz6wZxB8mayDK6glYTCyx//zl1luUdvm26PutA38K8cgnb7iTkfVP2OqK6sHAdXjnowIDAQAB;"
```

The name of the record is just an arbitrary `selector` in this case it's
`dkim` followed by `_domainkey`.

You can create multiple DKIM certs just change the name of the file to reflect
the domain in use. We'll use these to configure RSpamd later.


#### TLS/SSL

This certificate is specific to mail server address (`mail.example.com`) We use
Letsencryt to manage the certificates.

Firstly making sure the utility is installed:

```
apt install certbot
```

Next make sure open port 80 for verfication:

```
ufw allow 80
```

Now we can run the utility and follow the prompts:

```
certbot certonly
```

When it comes to renewing the certificates:

```
certbot renew
```

### Other DNS

#### SPF

SPF is a mechanism which makes it possible for destination mail servers to determine if a machine was allowed to send mail on behalf of a domain.

There are multiple ways to setup SPF, in our setup we will simply set it up so ONLY our mail server can send mail on my behalf:

```dns
example.com.      IN TXT  "v=spf1 mx -all"
```

#### DMARC

DMARC allows instructing a destination mail server what to do with senders that fail SPF and DKIM tests. Most notably it allows instructing them to reject such senders.

We have just setup the most dummy record, one that will make all Big Mailer Corps see that we care about DMARC even though we don't know what to do with it:

```dns
_dmarc.example.com.   IN TXT    "v=DMARC1;p=none;pct=100;rua=mailto:postmaster@example.com;"
```

#### DNS & rDNS

We obviously need and `A` record for `mail.example.com` pointing to our servers
IP address and the reverse DNS so that the IP address of our server resolves to
our hostname. 

For digitalocean check out [their docs](https://www.digitalocean.com/docs/networking/dns/how-to/manage-records/#ptr-rdns-records).

## Components

The server is made up of a number of moving parts that talk to each other.

- OpenSMTPD is the MDA and MTA
- RSpamd is a powerful anti-spam and filtering layer. Left alone it does and
  pretty good job
- Dovecot is the IMAP server

## Installation & Config

### Installing the required packages

There are a few things we need to get going but this list will just get
everything on in one hit:

```
apt install redis-server rspamd opensmtpd opensmtpd-filter-rspamd opensmtpd-filter-senderscore dovecot-imapd
```

#### RSpamd

##### Actions

The `/etc/rspamd/actions.conf` file is where we set the thresholds for certain actions. This should be pretty self
explanatory and just sets the limits for when it will reject, or add junk headers.

```
actions {
    reject = 15; # Reject when reaching this score
    add_header = 6; # Add header when reaching this score
    greylist = 4; # Apply greylisting when reaching this score (will emit `soft reject action`)
[...]
```

##### Options

I had some issues with the Nameserver resolution but it was easily fixed by adding in Cloudflare's public nameservers. So edit `/etc/rspamd/options.inc`

```
[...]
dns {
    timeout = 1s;
    sockets = 16;
    retransmits = 5;
    nameserver = ["1.1.1.1:53", "1.0.0.1:53"]; # Add this line to fix the issue.
}
[...]
```

##### DKIM Signing

We really want RSpamd to also do our DKIM signing and checking. This is done by providing the configuration to match the DKIM key. Create the config file `/etc/rspamd/local.d/dkim_signing.conf` with the following content:

```
selector = "dkim";
path = "/etc/dkim/$domain.$selector.key";
use_domain = "envelop";
check_pubkey = true;
```

This matches to to our `/etc/dkim/example.com.dkim.key` key that we generated earlier.

##### SPF Checks

This is just overriding some of the defaults nothing to change here: `/etc/rspamd/local.d/spf.conf`

```
spf_cache_size = 1k; # cache up to 1000 of the most recent SPF records
spf_cache_expire = 1d; # default max expire for an element in this cache
max_dns_nesting = 10; # maximum number of recursive DNS subrequests
max_dns_requests = 30; # maximum count of DNS requests per record
min_cache_ttl = 5m; # minimum TTL enforced for all elements in SPF records
disable_ipv6 = false; # disable all IPv6 lookups
```

#### OpenSMTPD

I am going to just dump in the configuration file for `/etc/smtpd.conf` there
should be no need to change it apart from replacing `mail.example.com` with the
actual production mail server domain.

There are also some `filters` that apply junk tags to messages that meet certain
criteria, you can swap the comments to reject these messages, but the more
conservative option is default.

```
pki mail.example.com cert "/etc/letsencrypt/live/mail.example.com/fullchain.pem"
pki mail.example.com key "/etc/letsencrypt/live/mail.example.com/privkey.pem"

# filter check_dyndns phase connect match rdns regex { '.*\.dyn\..*', '.*\.dsl\..*' } disconnect "550 no residential connections"
filter check_dyndns phase connect match rdns regex { '.*\.dyn\..*', '.*\.dsl\..*' } junk

# filter check_rdns phase connect match !rdns disconnect "550 no rDNS is so 80s"
filter check_rdns phase connect match !rdns junk

# filter check_fcrdns phase connect match !fcrdns disconnect "550 no FCrDNS is so 80s"
filter check_fcrdns phase connect match !fcrdns junk

# filter senderscore proc-exec "filter-senderscore -blockBelow 10 -junkBelow 70 -slowFactor 5000"
filter senderscore proc-exec "filter-senderscore -junkBelow 70 -slowFactor 5000"

filter rspamd proc-exec "filter-rspamd"

table domains file:/etc/mail/domains
table virtuals file:/etc/mail/virtuals
table credentials passwd:/etc/mail/credentials

listen on eth0 tls pki mail.example.com filter { check_dyndns, check_rdns, check_fcrdns, senderscore, rspamd }
listen on eth0 port submission tls-require pki mail.example.com auth <credentials> filter rspamd

action "domain_mail" maildir "/var/vmail/%{dest.domain}/%{dest.user}" virtual <virtuals>
action "outbound" relay

match for local reject
match from any for domain <domains> action "domain_mail"
match from any auth for any action "outbound"
```

This config is based from a number of sources, namely:
- [Poolp.org](https://poolp.org/posts/2019-09-14/setting-up-a-mail-server-with-opensmtpd-dovecot-and-rspamd/)
- [Vultr](https://www.vultr.com/docs/an-openbsd-e-mail-server-using-opensmtpd-dovecot-rspamd-and-rainloop)
- [OpenSMTPD Docs](https://man.openbsd.org/smtpd.conf)

##### Configuring the virtual users and domains

Because we are not using the system users and instead creating virtual users, we
need to create a system user to manage all the mail and then create the config
files for OpenSMTPD and Dovecot to know how to authenticate users.

Start with creating the system user:

```
adduser --system --home /var/vmail --disabled-login --uid 2000 --group vmail
```

Next up lets create our domains config file. This is just a list of domains that
we will accept mail for, all other mail will be rejected. So create
`/etc/mail/domains` and add a list of domains, with each new domain on a new
line:

```
example.com
```

Now lets get out credentials setup, this file is shared by both OpenSMTPD and
Dovecot to authenticate our users.

We start by generating our password to enter into `/etc/mail/credentials` config
file using a website like [bcrypt-generator](https://bcrypt-generator.com)

Then we add the new password into the `/etc/mail/credentials` file:

```
username@example.com:<generated_password>:vmail:2000:2000:/var/vmail/example.com/username::userdb_mail=maildir:/var/vmail/example.com/username
```

You can enter multiple entries on each line, obviously changing `username` for
the email you want and `example.com` for the domain. 

The second part of the declaration just defines where we want to store the
messages. We have used a self explanatory directory structure.

Finally, we have to setup our virtuals, but think of these and aliases really.
Open `/etc/mail/virtuals` and add one map per line. Should be self explanatory,
but minimum we should collect `abuse`, `noc`, `security`, `postmaster` and
`hostmaster`. Any account that isn't a forwarder should reference our system user vmail like the last line here.

```
abuse@example.com: username@example.com
noc@example.com: username@example.com
security@example.com: username@example.com
postmaster@example.com: username@example.com
hostmaster@example.com: username@example.com
username@example.com: vmail
```

Now lets check out config:

```
smtpd -n
```

Should be all ok!

#### Dovecot

Dovecot can be a bit hungry with resources so we want to pull on it's reins a
bit! Edit `/etc/security/limits.conf` and add to the list of examples: 

```
dovecot          soft    nofile          1024
dovecot          hard    nofile          2048
```

This step is not required and these numbers can be fine tuned as required.

##### Authentication

Open `/etc/dovecot/conf.d/10-auth.conf` and uncomment the line `disable_plaintext_auth = yes`

At the bottom of this file we have a set of include statements. Comment out the
`auth-system` configuration and remove the comment for the `auth-passwdfile`
config.

Next `/etc/dovecot/conf.d/10-mail.conf` and change the `mail_location = maildir:/var/vmail/%d/%n`

Further down uncomment and change `mmap_disable = yes`

Further down still uncomment and ammend `first_valid_uid` and `first_valid_gid` to be
`2000`.

Now lets set the SSL options in `/etc/dovecot/conf.d/10-ssl.conf`
First option `ssl = required`
Then we change the paths to our certificate and key:
```
ssl_cert = </etc/letsencrypt/live/mail.example.com/fullchain.pem
ssl_key = </etc/letsencrypt/live/mail.example.com/privkey.pem
```

Finally we just need to configure the passwd authentication to look at the
correct credentials file we used earlier. Edit
`/etc/dovecot/conf.d/auth-passwdfile.conf.ext` and change the paths to the
crentials file created earlier:

```
passdb {
  driver = passwd-file
  args = scheme=CRYPT username_format=%u /etc/mail/credentials
}

userdb {
  driver = passwd-file
  args = username_format=%u /etc/mail/credentials

  # Default fields that can be overridden by passwd-file
  #default_fields = quota_rule=*:storage=1G

  # Override fields from passwd-file
  #override_fields = home=/home/virtual/%u
}
```

#### Restart all services

Once all the configuration is complete we just need to restart all the services:

```
service opensmtpd restart
service dovecot restart
service rspamd restart
```

### Adding domains or users

The key configs that need to be updated are:

- `/etc/rspamd/local.d/dkim_signing.conf` to add the DKIM key
- `/etc/mail/domains` to add the accepted domain
- `/etc/mail/credentials` to add the accounts
- `/etc/mail/virtuals` to add any aliases/forwarders

Then make sure you restart the services to pick up the new config:

```
service opensmtpd restart
service dovecot restart
service rspamd restart
```

## Firewall

These are the bare minimum firewall rules that we need open:

```
ufw allow 22
ufw allow 25
ufw allow 465
ufw allow 587
ufw allow 993
```

However whenever you need to generate certifgicates with `certbot` you may have to open port `80` and close it again afterwards

```
ufw allow 80
```

```
ufw delete allow 80
```

## Client configuration

##### Incoming Mail Server (IMAP)

**Username:** john@example.com

**Password:** <password>
  
**Hostname:** mail.example.com

**Port:** 993 & Use TLS/SSL

**Authentication:** Password

##### Outgoing Mail Server (SMTP)

**Username:** john@example.com

**Password:** <password>
  
**Hostname:** mail.example.com

**Port:** 587 & Use TLS/SSL

**Authentication:** Password
