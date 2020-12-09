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
openssl genrsa -out /etc/dkim/example.com.key 1024
openssl rsa -in /etc/dkim/example.com.key -pubout -out /etc/dkim/example.com.pub 

cat /etc/dkim/example.com.pub

-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDThHqiM610nwN1nmV8OMc7PaPO
uJWVGRDz5bWj4cRjTTmQYjJQd02xrydRNrRhjEKBm2mMDArNWjoM3jvN04ZifqJx
DmKr7X8jsYi+MPEHz6wZxB8mayDK6glYTCyx//zl1luUdvm26PutA38K8cgnb7iT
kfVP2OqK6sHAdXjnowIDAQAB
-----END PUBLIC KEY-----
```

Then we can create the DNS TXT record by extracting the public key out of the armor delimiters and formatting the content so it displays as follows:

```dns
20201207._domainkey.example.com. IN TXT "v=DKIM1;k=rsa;p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDThHqiM610nwN1nmV8OMc7PaPOuJWVGRDz5bWj4cRjTTmQYjJQd02xrydRNrRhjEKBm2mMDArNWjoM3jvN04ZifqJxDmKr7X8jsYi+MPEHz6wZxB8mayDK6glYTCyx//zl1luUdvm26PutA38K8cgnb7iTkfVP2OqK6sHAdXjnowIDAQAB;"
```

The name of the record is just an arbitrary selector in this case its the date
`20201207` followed by `_domainkey`.

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

Couple of files here, the first is the `/etc/rspamd/actions.conf` this is pretty self
explanatory and just sets the limits for when it will reject, or add junk
headers.

```
actions {
    reject = 15; # Reject when reaching this score
    add_header = 6; # Add header when reaching this score
    greylist = 4; # Apply greylisting when reaching this score (will emit `soft reject action`)
[...]
```

Secondly, we really want RSpamd to also do our DKIM signing. This is done by providing the configuration to match the DKIM key. Create the config file `/etc/rspamd/local.d/dkim_signing.conf` with the following content:

```
domain {
    example.com {
        path = "/etc/dkim/example.com.key";
        selector = "20201207";
    }
}
```

You can see how you can add multiple domain blocks

More information can be found [in the docs](https://www.rspamd.com/doc/modules/dkim_signing.html)

#### OpenSMTPD

I am going to just dump in the configuration file for `/etc/smtpd.conf` there
should be no need to change it, it's just for reference. 

