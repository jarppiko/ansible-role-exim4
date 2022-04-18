# Ansible Role: Exim

Installs Exim 4 ESMTP server on Debian/Ubuntu, but does not use the debian config packaging so
as to keep the config file simple. Adapting the role to other Linux systems shouldn't be very
hard but has not been done.

There are three major operational modes this supports:
 - as a smarthost server, forwarding all mail to another 'smarter' server;
 - as an SMTP server on the internet, using MX records and DKIM;
 - as a local user server, not on the internet;
 
For both the latter modes you can enable local users via IMAP and LMTP, or using files in a 
filesystem (old style). There is some flexibility in how the roles are defined. Smarthosts are
very useful for systems that might generate mail (e.g. a webserver) but which you don't want
to be handling mail. They end up as simple store-and-forward systems, adding resilience.

The role includes support for:

 - virtual email domains, based on MySQL user authentication;
 - basic support for creating exim mail filters and setting virtual-homedirs;
 - support for LMTP email delivery to e.g. Dovecot or Cyrus-IMAPd.
 - integrated spamassassin spam filtering (install SA sepatately);
 - integrated GNU 'mailman' routing and delivery  (install MM sepatately);
 - callouts to externally configured virus checking tools such as ClamAV;
 - lists of good and bad senders and hosts;
 - configurable aliases and rewrites per domain;
 - fairly advanced anti-spam measures even before SpamAssassin;
 - lots more!


## Requirements

You may need to open TCP port 25 (and/or 465, 587, dep on services) in your server firewall.

The role is preconfigured to support spamassassin scanning of incoming mail. Some builds of Exim
may not include 'spamd' support, or you may not want this, so there is a flag to disable it.

Dovecot support is present in the sense that there is support to send mail to an LMTP port,
which dovecot (and cyrus-imapd) like. Local mail files are also supported (set
exim\_deliver\_localuser true). For Dovecot, sending mail via LMTP creates the user mailbox, there is
no need to explicitly define it. For Cyrus, you do have to explicitly create a new user mailbox,
and that is not done in this role.

There is also support built in for Mailman, both the older MM2 and newer MM3, when the list
directory tree is visible to Exim (it is used to check which list domains exist). Configuration
of Mailman itself is not part of this role.

Spamassassin can also be enabled simply, but again configuration of SA itself is not included.

## Role Variables

Some of the main variables are listed below. See `defaults/main.yml` for the complete list
and further documentation.


### Smarthost Example

This configuration is about as simple as it gets: a basic SMTP-only smarthost server, suitable
for email relaying duties e.g. on a web server or print server. The default listen address is
the ipv4 localhost, so this server won't get mail from anywhere else.

    exim_server_name: "{{ hostname_fqdn }}"
    
    #    TCP/IP ports to listen for mail. Keep things simple with port 25:
    exim_listen_ports: [ 25 ]
    #    No listen_ports start off with TLS enabled:
    exim_tls_ports: []
    #    No local delivery => no local domains.
    exim_local_domains: []
    
    # What to do with incoming mail: here, a non-local smarthost.
    #    No to sending via MX records to the internet.
    exim_deliver_direct: no
    #    Yes to smarthost:
    exim_deliver_smarthost: yes
    #    No local users.
    exim_deliver_localuser: no
    
    # Where is the smarthost?
    exim_smarthost_address: post.example.org
    exim_smarthost_authenticated: no
    
    # No local spamd support - assume it's done on the smarthost.
    # Enable this if your smarthost might get upset with you for sending it spam!
    exim_use_spamassassin: no
    exim_use_dkim: no


You can add spamassassin and TLS encryption to this if you need. smarthosts don't support local 
users, so most other facilities don't make sense.


### Single domain Email Server Example

This is a simple server for one domain, listening on both SMTP and TLS ports. You will need
to set up a mysql database to provide credentials: put details in the __exim* variables.

exim_virtual_domains is the core dictionary enumerating the delivery domains names supported by
this server. At the top level, keys represent the domain name - the code calls it 'vtag' and it is
also used for domain-specific filenames. It should be short and purely alphabetic.

The value for each vtag is itself a dictionary, containing key 'domains' which lists the actual
domains covered. Also the local users, local aliases and address rewrites are defined.

Without at least one virtual domain item defined, you have no local users, and so no local
delivery. A smarthost wants an empty virtual domain setup because it only ever relays mail.


    exim_server_name: example.org
    
    # Which domain names are condsidered to be local (i.e. can deliver to / send from)
    exim_local_domains:
    - example.org
    
    # Host addresses on which to listen.
    exim_listen_addresses: 
    - 127.0.0.1
    - example.org
    
    # TCP/IP ports on which exim should listen for mail
    exim_listen_ports:
    - 25
    - 465
    - 587
    
    # Ports (already in exim_listen_ports) that are TLS by default.
    exim_tls_ports:
    - 587
    
    # Deliver mail using DNS and MX records (otherwise only deliver locally).
    exim_deliver_direct: yes
    
    # Mail delivery to IMAP server.
    exim_imap_transport: yes
    
    # Where is local mail delivered: expected to be using LMTP socket
    exim_imap_deliver_socket: /var/run/dovecot/lmtp

    # Pass the message on to SpamAssassin for spam checking.
    exim_use_spamassassin: yes
    
    # List of aliases which will all resolve to 'postmaster', which must exist as
    # an alias or user. This is a shortcut for local system processes that send
    # informational/system messages. If you need to override them, remove from this
    # list and include under each domain.
    exim_all_domain_aliases: 
      - abuse
      - MAILER-DAEMON
      - root
      - bin
    
    exim_all_blocked_senders:
      warez.me:        [ noreply ]
    
    exim_all_accepted_senders:
      bloomberg.net:      [ no-reply ]
    
    exim_virtual_domains:
      exampleorg:
        local_users:
          - ruth
          - john
          - judy
          
        local_aliases:
          - from: postmaster
            to: ruth
          - from: abuse
            to: john
            
        domains:
          # The first is used to qualify unqualified addresses, ie "jo" to "jo@dom.ain" in some cases.
          - example.org
          
        rewrites:
          - from: "*@*.example.org"
            to: "${local_part}@example.org"
            opts: q
    
    # TLS
    exim_tls_certificate_file: "/etc/letsencrypt/live/{{ exim_server_name }}/fullchain.pem"
    exim_tls_privatekey_file: "/etc/letsencrypt/live/{{ exim_server_name }}/privkey.pem"
    
    # Store credentials in a mysql DB.
    exim_auth_store_mysql: yes    
    
    # User/password lookup on local mysql server. Put definitions of __ variables in the vault.
    exim_mysql_hostname: localhost
    exim_mysql_database:  "{{ __exim_mysql_database }}"
    exim_mysql_table:  "{{ __exim_mysql_table }}"
    exim_mysql_user:  "{{ __exim_mysql_user }}"
    exim_mysql_password: "{{ __exim_mysql_password }}"
    
    # Enable/disable DKIM checking facility.
    # Enable if possible.
    exim_use_dkim: yes
    
    # Domains we expect to sign mail.
    exim_dkim_good_signers:
      - gmail.com
      - paypal.com
      - ebay.com
    
    # DKIM/DMARC config.
    exim_dkim_canon: 'relaxed'
    exim_dkim_selector: 'selector'
    

## Dependencies

No direct dependencies, but you may need to configure:

 - mysql or mariadb, for authentication
 - spamassassin - generally a good idea even on some smarthosts;
 - mailman, if using mailing lists;
 - ClamAV or equivalent, if virus scanning is enabled;
 - Dovecot or equivalent, if you have local IMAP users.


## License

MIT / BSD


## Author Information

This role was created 2018-2020 by [Ruth Ivimey-Cook](https://www.ivimey.org/) from an original
outline "ansible-exim" by Jeff Geerling.
