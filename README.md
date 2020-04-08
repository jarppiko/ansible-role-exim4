# Ansible Role: Exim

Installs Exim 4 ESMTP server on Debian/Ubuntu.

There are three major operational modes this supports:

 - as a smarthost server, forwarding all mail to another 'smarter' server;
 - as an SMTP server on the internet, using MX records and DKIM;
 - as a local user server, not on the internet;
 
For both the latter modes you can enable local users via IMAP and LMTP, or using files in a 
filesystem (old style).

The role includes support for:

 - virtual email domains, based on MySQL user authentication;
 - integrated spamassassin spam filtering (install SA itself sepatately);
 - integrated GNU 'mailman' routing and delivery;
 - callouts to externally configured virus checking tools such as ClamAV;
 - blacklists and whitelists of both host addresses and sender email addresses;
 - lots more!

## Requirements

You may need to open TCP port 25 (and/or 465, 587, dep on services) in your server firewall.

The role is preconfigured to support spamassassin scanning of incoming mail. Some builds of Exim
may not include 'spamd' support, or you may not want this, so there is a flag to disable it.

The 

## Role Variables

Some of the main variables are listed below. See `defaults/main.yml` for the complete list:

### Smarthost mode

This configuration is about as simple as it gets: a basic SMTP-only smarthost server, suitable
for email relaying duties e.g. on a web server or print server. The default listen address is
the ipv4 localhost, so this server won't get mail from anywhere else.

    exim_server_name: example.org
    
    # TCP/IP ports on which exim should listen for mail. Keep things simple on port 25.
    exim_listen_ports: [ 25 ]
    exim_tls_ports: []
    exim_local_domains: []
    
    # These define the basic operation: here, a non-local smarthost.
    exim_deliver_direct: no
    exim_deliver_smarthost: yes
    exim_deliver_localuser: no
    
    # Where is the smarthost?
    exim_smarthost_authenticated: no
    exim_smarthost_address: post.example.org
    
    # No spamd support.
    exim_use_spamassassin: no
    exim_use_dkim: no

You can add spamassassin and TLS encryption to this if you need. smarthosts don't support local 
users, so most other facilities don't make sense.

### Single domain Email Server

This is a simple server for one domain, listening on both SMTP and TLS ports. You will need
to set up a mysql database to provide credentials: put details in the __exim* variables.

    exim_server_name: example.org
    
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
        
    # Pass the message on to SpamAssassin for spam checking.
    exim_use_spamassassin: yes
    
    # Aliases added to all domains.
    exim_all_domain_aliases: 
    - postmaster
    - abuse
    
    exim_all_sender_blacklist:
      warez.me:        [ noreply ]
    
    exim_all_sender_whitelist: 
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
          - example.org
          
        rewrites:
          - from: "*@*.example.org"
            to: "${local_part}@example.org"
            opts: q
    
    # Which domain names are condsidered to be local (i.e. can deliver to / send from)
    exim_local_domains:
    - example.org
    
    # TLS
    exim_tls_certificate: "/etc/letsencrypt/live/{{ exim_server_name }}/fullchain.pem"
    exim_tls_privatekey: "/etc/letsencrypt/live/{{ exim_server_name }}/privkey.pem"
    
    # Store credentials in a mysql DB.
    exim_auth_store_mysql: yes    
    
    # User/password lookup on local mysql server:
    exim_mysql_hostname: localhost
    exim_mysql_database:  "{{ __exim_mysql_database }}"
    exim_mysql_table:  "{{ __exim_mysql_table }}"
    exim_mysql_user:  "{{ __exim_mysql_user }}"
    exim_mysql_password: "{{ __exim_mysql_password }}"
    
    # Enable/disable DKIM checking facility.
    # Enable if possible.
    exim_use_dkim: yes
    
    # Domains we expect to sign mail.
    exim_dkim_known_signers:
      - gmail.com
      - paypal.com
      - ebay.com
    
    # DKIM/DMARC config.
    exim_dkim_canon: 'relaxed'
    exim_dkim_selector: 'selector'
    

## Dependencies

No direct dependencies, but you may need to configure:

 - mysql or mariadb, for authentication
 - spamassassin
 - ClamAV or equivalent
 - Dovecot or equivalent


## License

MIT / BSD


## Author Information

This role was created 2018-2020 by [Ruth Ivimey-Cook](https://www.ivimey.org/) from an original outline "ansible-exim" by Jeff Geerling.
