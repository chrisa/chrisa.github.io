---
layout: post
title: "Deploying a PKI with Chef"
date: 2012-07-22
comments: false
categories: ["chef"]
---

Traditionally, rolling out an X.509-style PKI has been a lot of
work. You've got to create the CA, generate all the necessary private
keys and distribute them securely, create signed certificates and
distribute those, and then do it all again when renewal time comes
around. Often this is enough work that a simpler security model is
used instead, like a shared secret.

With a modern configuration management system we ought to be able to
do better than this. Our CM system can know what certificates are
required where, and which CA should issue them. We've got an
authenticated transport, and the ability for an operator to control
the certificate issuing process from a central location.

This is the solution the `ssl` cookbook tries to provide for
Chef-managed infrastructures: it allows recipes to specify the
certificates required alongside the services which will use them,
generates private keys right on the host, and manages the certificate
issuing process via the Chef server and a separate command-line tool.

## Using the ssl_certificate resource ##

To use the `ssl` cookbook, you'll start by declaring as resources the
certificates your infrastructure needs. Probably the most common will
be a certificate for an https server - the standard "SSL certificate"
that gives the resource its name:

```ruby
ssl_certificate "www.example.com" do
  ca "MyCA"
  key "/etc/ssl/www.example.com.key"
  certificate "/etc/ssl/www.example.com.cert"
  type "server"
  bits 1024
  days 365
end
```

Here, we're specifying a certificate whose Common Name is
"www.example.com". The other parts of the certificate's Distinguished
Name are set as node attributes, and needn't be specified here.

The CA whose signature is requested here is `MyCA`. This is a string
which identifies your CA. It might identify an internal CA, or it
might be an abbreviation which indicates an external CA you might use,
but you need to choose an appropriate "short name" for your CA to use
here. Later, we'll use this identifier to find the CSR to sign.

The key and certificate locations are given here, which you'd need to
also specify in your webserver configuration. The key is given
restrictive `0600` permissions, and ownership defaults to
`root:root`. The key itself is declared to be of type `server`, and of
length 1024 bits.

A validity period of 365 days is requested for the certificate. This
is a request, and need not be honoured when the CSR is signed, but it
will be respected if the automatic signing tool is used. 

The first time this resource is converged, the key will be generated
and installed, and a temporary certificate signed by a temporary CA is
also installed -- this allows a webserver configured to use the key
and certificate to start up successfully. Browsers connecting to the
server will see a warning, indicating the correct certificate is not
yet in place.

The Chef run also generates a CSR and PGP-encrypts the private key,
then saves them both into node attributes, in the "CSR Outbox". This
contains the CSR, and the requested CA and validity period ready for
the signing tool.

## Signing certificates ##

The `chef-ssl` tool is provided as a Ruby gem, called
`chef-ssl-client`, and is maintained in the cookbook's git
repository. This is a standalone command line tool, which handles the
certificate issuing parts of the process, but which also provides for
the creation of new CAs and the issuing of ad hoc certificates.

Often you'll be signing certificates with your own CA for internal
services, then deploying that CA to your clients. `chef-ssl` makes it
easy to sign CSRs for these internal CAs:

```
    $ chef-ssl autosign --ca-name MyCA --ca-path ./MyCA
```

You'll be prompted for the CA key's passphrase, and then a Chef search
will be performed, looking for data in individual nodes' outboxes. The
search is constrained by the CA identifier you specified.

Each matching CSR is then presented in turn, and if you're happy to
issue the signed certificate, answering "yes" will do a number of
things:

 * Create a certificate for the CSR
 * Sign the certificate with your CA
 * Upload the certificate as a Chef data bag item

The data bag item is named for the Common Name of the certificate, and
contains all the information relating to the individual key and
certificate. The PGP-encrypted key is stored for archive purposes,
along with the CSR, and the certificate itself, the date and the
signing CA's certificate. The data bag item may be archived for backup
of both the key and certificate.

## Installing Certificates ##

On the next run after the certificate is issued and uploaded, Chef
notes the existence of the relevant data bag item, downloads it and
installs the certificate and the signing CA's certificate.

Having installed the real certificate, Chef then removes the CSR from
the outbox -- it's important here that only the Chef node ever updates
the outbox, and only the `chef-ssl` script ever updates data bags. In
this way, clobbering of data is avoided, and it's clear which part of
the system is responsible for each piece of data.

Notifications are sent when the new certificate is installed, which
allows (for example) the relevant web server to be restarted and begin
using the certificate.

## Summary ##

This cookbook should allow you to concentrate on choosing the right
security architecture for your systems, rather than avoiding a genuine
PKI where it's the right solution only because of the work involved. 

The cookbook is available here:

https://github.com/VendaTech/chef-cookbook-ssl

where further documentation and examples are available. GitHub issues
and pull requests are most welcome.

