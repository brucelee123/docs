debops.pki
##########



This role will bootstrap and manage fully-fledged Public Key Infrastructure
entirely within Ansible. You can create Certificate Authorities
(stand-alone as well as chained together) which in turn can automatically
sign incoming Certificate Requests and provide you with fully-functional
OpenSSL or GnuTLS certificates for your entire infrastructure, for free.

`debops.pki` can also be used to easily distribute private keys and
certificates signed by an external CA to your hosts in an easy and
convenient way. Role will automatically create a set of symlinks to make
use of the certificates within your applications easy and intuitive.

.. contents:: Table of Contents
   :local:
   :depth: 2
   :backlinks: top

Installation
~~~~~~~~~~~~

This role requires at least Ansible ``v1.7.0``. To install it, run::

    ansible-galaxy install debops.pki


Role dependencies
~~~~~~~~~~~~~~~~~

- ``debops.secret``


Role variables
~~~~~~~~~~~~~~

List of default variables available in the inventory::

    ---
    
    # Enable or disable PKI support
    pki: True
    
    
    # ---- Default DN for Certificate Requests ----
    
    pki_country:             'QS'
    pki_state:               'Q-Space'
    pki_locality:            'Global'
    pki_organization:        '{{ ansible_domain.split(".")[0] | capitalize }}'
    pki_organizational_unit: 'Data Center Operations'
    pki_common_name:         '{{ ansible_fqdn }}'
    pki_email:               'root@{{ ansible_domain }}'
    
    
    # ---- PKI main options ----
    
    # List of additional packages to install ('haveged' might be useful for faster
    # randomness in testing environment)
    pki_packages: []
    
    # Base PKI directory on remote hosts
    pki_base_path: '/etc/pki'
    
    # Base PKI directory on Ansible Controller
    # See debops.secret role for more information
    pki_base_src: '{{ secret + "/pki/" + ansible_domain }}'
    
    # Directory and file permissions for public and private data
    pki_owner: 'root'
    pki_public_group: 'root'
    pki_private_group: 'ssl-cert'
    pki_public_dir_mode: '0755'
    pki_private_dir_mode: '2750'
    pki_public_mode: '0644'
    pki_private_mode: '0640'
    
    # Make sure these private system groups exist
    pki_private_groups_present: []
    
    
    # ---- Certificate defaults ----
    
    # Default digest engine to use for signatures
    pki_digest: 'sha256'
    
    # Default key size
    pki_private_key_size: '2048'
    
    # Base sign period for "normal" certificates
    pki_sign_days: '365'
    
    # Base multiplier for Root CA - 10 years
    pki_sign_rootca_multiplier: '10'
    
    # Base multiplier for intermediate CA - 5 years
    pki_sign_ca_multiplier: '5'
    
    # Base multiplier for certificate - 1 year
    pki_sign_cert_multiplier: '1'
    
    
    # ---- Root Certificate Authority configuration ----
    
    pki_rootca: 'RootCA'
    pki_rootca_filename: '{{ pki_rootca + "-" + ansible_domain }}'
    pki_rootca_private_key_size: '4096'
    pki_rootca_o: '{{ pki_organization + " Certificate Authority" }}'
    pki_rootca_cn: '{{ pki_organization + " Root Certificate" }}'
    
    
    # ---- PKI snapshot configuration ----
    
    pki_snapshot: True
    pki_snapshot_path: '/var/backups'
    pki_snapshot_file: '{{ "pki-snapshot-" + ansible_fqdn + ".tar" }}'
    pki_snapshot_owner: 'root'
    pki_snapshot_group: 'root'
    
    
    # ---- Other configuration ----
    
    # Default library used to manage the certificates (openssl or gnutls)
    # Currently only OpenSSL is fully supported
    pki_library: 'openssl'
    
    # Certificate bundle configured as 'CA.crt' if no CA has been specified
    pki_default_ca: '/etc/ssl/certs/ca-certificates.crt'
    
    # Name of the certificates to symlink as 'default.*' if no default has been
    # specified
    pki_default_certificate: '{{ ansible_fqdn }}'
    
    # PKI realm to set as the default (it will be written in Ansible local facts,
    # as well as symlinked to '/etc/pki/system/')
    pki_default_realm: 'host'
    
    # By default files from all realms are sent to all remote hosts. To prevent
    # access to a realm for a particular host, add the realm name to this list to
    # prevent it being sent to the server
    pki_realm_blacklist: []
    
    # Certificate name to symlink as 'default.*' in PKI 'host' realm
    pki_default_host_certificate: '{{ ansible_fqdn }}'
    
    # Certificate name to symlink as 'default.*' in PKI 'domain' realm
    pki_default_domain_certificate: '{{ "wildcard.domain." + ansible_fqdn }}'
    
    # Subdomain reserved for CA server (certificate revocation lists, source for
    # Root certificate, etc.)
    pki_ca_domain: 'pki.{{ ansible_domain }}'
    
    # This string is used to uniquely bind a certificate to the requesting host
    pki_default_certificate_uri: '{{ "http://" + pki_ca_domain + "/cert/" + (ansible_default_ipv4.macaddress | sha1) }}'
    
    
    # ---- PKI realms ----
    
    # PKI realm is defined as a "channel" through which certificate requests are
    # sent to the Ansible controller and certificates, as well as other files, are
    # sent to remote hosts. It's defined by a "source directory" (on Ansible
    # Controller) and "destination directory" (on a remote host). Multiple sources
    # can be connected to one destination.
    #
    # Each realm can have an optional Certificate Authority bound to it, which is
    # used to sign certificates requested in that realm. Since each realm generates
    # a Makefile in its destination directory, this can be disabled to not
    # interfere if multiple source directories are connected to 1 destination.
    # You can also specify a certificate name which will be symlinked as
    # 'default.*' in main directory of the PKI realm. You can also specify which CA
    # certificates should be installed in a particular realm 'CA/' directory.
    #
    # To provide your own certificates and keys signed by an external CA, put them
    # in 'secret/pki/realms/' directory in a desired realm.
    pki_realms:
    
        # This realm is used to distribute certificates to all hosts in a domain. It
        # does not have its own CA, and additionally distributes the main Root
        # Certificate Authority to all hosts. If you manage hosts on which an
        # external entity might have access to private keys, and you want to prevent
        # them access to your wildcard certificates, you might want to disable this
        # realm on a particular host.
      - name: 'domain'
        source: 'domain'
        destination: 'domain'
        ca: [ 'root/RootCA' ]
        makefile: False
    
        # This realm can be used to manage wildcard certificates per host, instead of
        # globally. It by default provides a wildcard certificate for your domain.
      - name: 'host-domain'
        source: 'hosts/{{ ansible_fqdn }}/domain'
        destination: 'domain'
        authority: 'ca/domain'
        default: '{{ pki_default_domain_certificate }}'
    
        # This realm can be used to manage host-based certificates, a certificate
        # for your host will be automatically generated.
      - name: 'host-internal'
        source: 'hosts/{{ ansible_fqdn }}/host'
        destination: 'host'
        authority: 'ca/internal'
        default: '{{ pki_default_host_certificate }}'
    
    
    # ---- Certificate Authoriries ----
    
    # This list defines a chain of Certificate Authorities, from Root CA, through
    # Intermediate CA, ending on the "endpoint" CA which issue client and server
    # certificates. Root and Intermediate CA after signing the CSR of sibling CA
    # will automatically lock themselves, which allows you to move their private
    # keys offline to a secure storage.
    #
    # CA will automatically sign all incoming Certificate Signing Requests and
    # create chained certificates (with intermediate CA certificates included).
    # Signed certificates will be stored in a central location and distributed to
    # proper realms using route scripts (see below).
    pki_authorities:
    
      - name: 'root/RootCA'
        grants: 'ca'
        private_key_size: '{{ pki_rootca_private_key_size }}'
        filename: '{{ pki_rootca_filename }}'
        default_dn: False
        o: '{{ pki_rootca_o }}'
        cn: '{{ pki_rootca_cn }}'
    
      - name: 'intermediate/DomainCA'
        grants: 'ca'
        parent: 'root/RootCA'
        o: '{{ pki_rootca_o }}'
        ou: '{{ pki_organization + " CA" }}'
        cn: '{{ "ca." + ansible_domain }}'
    
      - name: 'ca/internal'
        parent: 'intermediate/DomainCA'
        ou: '{{ pki_organization + " Data Center" }}'
        cn: '{{ "dc." + ansible_domain }}'
    
      - name: 'ca/domain'
        grants: 'server'
        parent: 'intermediate/DomainCA'
        ou: '{{ pki_organizational_unit }}'
        cn: '{{ "dco." + ansible_domain }}'
    
    
    # ---- CA - realm route scripts ----
    
    # Route scripts provide a "glue" between Ansible facts and filesystem
    # directories. Because at the time of the Makefile execution system does not
    # have a knowledge about where to copy each file from Certificate Authorities
    # directories to PKI realms, small shell scripts are generated beforehand with
    # proper copy commands.
    pki_routes:
    
        # Copy signed host certificate to 'host' PKI realm
      - name: 'host_{{ ansible_fqdn }}'
        authority: 'ca/internal/certs'
        realm: 'hosts/{{ ansible_fqdn }}/host/certs'
        file: '{{ ansible_fqdn }}.crt'
    
        # Copy signed domain certificate to 'domain' PKI realm
      - name: 'domain_{{ ansible_fqdn }}'
        authority: 'ca/domain/certs'
        realm: 'hosts/{{ ansible_fqdn }}/domain/certs'
        file: 'wildcard.domain.{{ ansible_fqdn }}.crt'
    
        # Copy Root CA certificate to 'domain' realm for all hosts
      - name: 'root_ca'
        authority: 'root/RootCA'
        realm: 'domain/CA'
        readlink: 'CA.crt'
    
        # Copy internal CA CRL file to 'host' PKI realm
      - name: 'host_crl_{{ ansible_fqdn }}'
        authority: 'ca/internal'
        realm: 'hosts/{{ ansible_fqdn }}/host/revoked'
        readlink: 'default.crl'
    
        # Copy domain CA CRL file to 'domain' PKI realm
      - name: 'domain_crl_{{ ansible_fqdn }}'
        authority: 'ca/domain'
        realm: 'hosts/{{ ansible_fqdn }}/domain/revoked'
        readlink: 'default.crl'
    
    
    # ---- Certificates ----
    
    # This is a list of certificates to manage on a host. Each host sends
    # a Certificate Signing Request to Ansible Controller, where it's signed by
    # designated Certificate Authority and send back to the host.
    pki_certificates:
    
      - source: '{{ "hosts/" + ansible_fqdn + "/host" }}'
        destination: 'host'
        ou: '{{ pki_organization + " Data Center" }}'
        cn: '{{ ansible_fqdn }}'
        dns: [ '{{ "*." + ansible_domain }}' ]
        uri: [ '{{ pki_default_certificate_uri }}' ]
    
      - source: '{{ "hosts/" + ansible_fqdn + "/domain" }}'
        destination: 'domain'
        ou: '{{ pki_organizational_unit }}'
        cn: '{{ ansible_domain }}'
        dns: [ '{{ "*." + ansible_domain }}' ]
        uri: [ '{{ pki_default_certificate_uri }}' ]
        filename: 'wildcard.domain.{{ ansible_fqdn }}'
    
    
    # Example list of certificate options
    #  - realm: 'host'
    #    cn:    'www.example.com'
    #    mail:  [ 'root@example.com' ]
    #    dns:   [ 'www.example.com', 'mail.example.com', '*.mail.example.com' ]
    #    uri:   [ 'http://example.com/' ]
    #    ip:    [ '192.0.2.1' ]
    #
    #  - realm: 'host'
    #    cn:    'subdomain.{{ ansible_domain }}'
    #
    #  - realm: 'host'
    #    cn:    '{{ "other." + ansible_domain }}'
    #    ou:    'Other Department'
    #    e:     '{{ "root@other." + ansible_domain }}'
    #    mail:  [ '{{ "others@other." + ansible_domain }}', '{{ "root@" + ansible_domain }}' ]
    #    dns:   [ '{{ "*.other." + ansible_domain }}' ]




Authors and license
~~~~~~~~~~~~~~~~~~~

``debops.pki`` role was written by:

- Maciej Delmanowski | `e-mail <mailto:drybjed@gmail.com>`__ | `Twitter <https://twitter.com/drybjed>`__ | `GitHub <https://github.com/drybjed>`__

License: `GPLv3 <https://tldrlegal.com/license/gnu-general-public-license-v3-%28gpl-3%29>`_

