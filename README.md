Ansible Role: Bind9 Installation and Configuration
=========

This role install and configure [Bind9](https://www.isc.org/bind/) in a Ubuntu server.


Requirements
------------

None

Role Variables
--------------

Available variables are listed below along with default values (see `defaults\main.yaml`)

- Bind9 packages to install
  ```yaml
  bind9_packages:
  - bind9
  - dnsutils
  ```
- Bind9 server installation details

  bind UNIX user/group
  ```yml
  bind9_user: bind
  bind9_group: bind
  ```
  By default Ubuntu packages install bind configuration files in (`/etc/bind`). Additional directories is created to store Dynamic DNS (DDNS) keys(`bind9_keydir`)

  Ubuntu packate configure Apparmor permissions so `/var/lib/bind` directory is configured with proper permissions. That is the folder that must be used to store the zone files in case DDNS is in going to be used. If that directory is not uses DDNS updates are not working because of permission issues.

  Ansible variable (`bind9_zonedir`) contains that path and it should not be modified. See https://ubuntu.com/server/docs/domain-name-service-dns

  > Note:
  > If BIND will be receiving automatic updates to the file as with DDNS, then use /var/lib/bind/db.example.com rather than /etc/bind/db.example.com both here and in the copy command below.

  ```yml
  # Configuration directories
  bind9_zonedir: /var/lib/bind
  bind9_keydir: /etc/bind/keys
  ```

- Bind9 server Configuration

  IPv6 configuration
  Whether to enable IPv6 or not
  ```yml
  bind9_enable_ipv6: false
  ```
  
  Listening IP/IPv6 addresses:
  ```yml
  # IPv6 addresses to listen on
  bind9_listen_on_v6: []
  # IP addresses to listen on
  bind9_listen_on: []
  ```
  Leave them empty, default value, to enable listening in all IP addresses

  Configure DNS server as Resolver (enabling recursion)
  ```yaml
  bind9_recursor: true
  ```
  
  Configure DNS server as Forwarder (`bind9_forward`) and specify the list of forward DNS servers (`bind9_forward_servers`)
  ```yaml
  # run bind with forwarding?
  bind9_forward: true
  # forward servers
  bind9_forward_servers:
    - 1.1.1.1
    - 8.8.8.8
  ```

  Confifure DNS server as Authoritative DNS server
  ```yaml
  # Run bind as authoritative nameserver?
  bind9_authoritative: false
  ```

  Configure networks to include in ACL for querying.
  ```yaml
  # Permitted hosts/networks in ACL for querying
  bind9_acl_networks:
    - 10.10.0.0/16
    - 10.20.0.0/16
  ```
  Enable logging
  ```yaml
  # Enable logging
  bind9_named_logging: false
  bind9_log_path: /var/log/named
  ```

- Configure Zone files

  Zone files to be deployed in the Authoritative DNS server are defined in a dict `bind9_zones`
  The list of zones can include forward zones and reverse zone file definitions

  ```yaml
  bind9_zones:
    - name: homelab.example.com
      type: primary
      default_ttl: 600
      serial: 2024101102
      refresh: 1D
      retry: 2H
      expire: 1000H
      primary: ns.homelab.example.com
      admin: admin.example.com
      ns_records:
        - ns.homelab.example.com
      rrs:
        - label: ns
          type: A
          rdata: 192.168.2.11
        - label: node1
          type: A
          rdata: 192.168.2.11
        - label: node2
          type: A
          rdata: 192.168.2.12
        - label: node3
          type: A
          rdata: 192.168.2.13
    - name: 168.192.in-addr.arpa
      type: primary
      default_ttl: 600
      serial: 2024101102
      refresh: 1D
      retry: 2H
      expire: 1000H
      primary: ns.homelab.example.com
      admin: admin.homelab.example.com
      ns_records:
        - ns1.homelab.example.com.
      rrs:
        - label: 11.2
          type: PTR
          rdata: ns.homelab.example.com.
        - label: 12.2
          type: PTR
          rdata: node2.homelab.example.com.
        - label: 13.2
          type: PTR
          rdata: node3.homelab.example.com.
  ```

Zone files definition 

- Dynamic DNS (DNS)

  Whether the DDNS keys are generated or not
  ```yaml
  bind9_generate_ddns_key: true
  ```

  For each zone supporting dynamic updates the `ddns_update_key` and `ddns_update_policy_grant` need to be added. `ddns_update_key` contains file name of the key to be stored in `/etc/bind/keys` directory. `ddns_update_policy_grant` contain the grant policy description to be used. See details in [Bind9: Dynamic Update Policies](https://bind9.readthedocs.io/en/v9.16.20/reference.html#dynamic-update-policies)


  ```yaml
  bind9_zones:
    - name: homelab.example.com
      type: master
      ddns_update_key: ddnskey
      ddns_update_policy_grant: "zonesub any"
  ```

- Primary/Secondary configuration

  In primary server, zone definition has to be configured as `type` primary
  Also `secondary_servers` has to include list of IP addresses of secondary servers

  ```yaml
  bind9_zones:
    - name: homelab.example.com
      type: primary
      secondary_servers:
        - 10.11.0.12
  ```
  In secondary servers, zone definition has to be configured as `type` secondary
  Also `primary_servers` has to include IP addresses of the primary server

  ```yaml
  bind9_zones:
    - name: homelab.example.com
      type: secondary
      primary_servers:
        - 10.11.0.11
  ```
- Null forwarders configuration

 If you would like to be able delegate a subdomain from your authoritative server to another server, you may need to set `null_forwarders` to true. This enables the delegtion and proper name resolution when running both an authoritative server, and a general forwarder. For more details see https://kb.isc.org/docs/aa-00538.

  ```yaml
  bind9_zones:
    - name: homelab.example.com
      type: primary
      secondary_servers:
        - 10.11.0.12
      null_forwarders: true
  ```
  


Dependencies
------------

None

Example Playbooks
-----------------


The following playbook install and configure DNS server as authoritative server for domain `homelab.example.com`, it also configure it as Forwarder and Resolver.

```yml
---
- name: Install and configure Minio Server
  hosts: dns-server
  become: true
  gather_facts: true
  roles:
    - role: ricsanfre.bind9
      bind9_authoritative: true
      bind9_forward: true
      bind9_recursor: true
      bind9_forward_servers:
        - 8.8.8.8
        - 4.4.4.4
      bind9_generate_ddns_key: true
      bind9_zones:
        - name: homelab.example.com
          type: primary
          ddns_update_key: ddnskey
          ddns_update_policy_grant: "zonesub any"
          default_ttl: 600
          serial: 2024101102
          refresh: 1D
          retry: 2H
          expire: 1000H
          null_forwarders: false
          primary: ns.homelab.example.com
          admin: admin.example.com
          ns_records:
            - ns.homelab.example.com
          rrs:
            - label: ns
              type: A
              rdata: 192.168.2.11
            - label: node1
              type: A
              rdata: 192.168.2.11
            - label: node2
              type: A
              rdata: 192.168.2.12
            - label: node3
              type: A
              rdata: 192.168.2.13
        - name: 168.192.in-addr.arpa
          type: primary
          default_ttl: 600
          serial: 2024101102
          refresh: 1D
          retry: 2H
          expire: 1000H
          primary: ns.homelab.example.com
          admin: admin.homelab.example.com
          ns_records:
            - ns1.homelab.example.com.
          rrs:
            - label: 11.2
              type: PTR
              rdata: ns.homelab.example.com.
            - label: 12.2
              type: PTR
              rdata: node2.homelab.example.com.
            - label: 13.2
              type: PTR
              rdata: node3.homelab.example.com.
```

The following configure a primary (`ns1.homelab.example.com`) and secondary (`ns2.homelab.example.com`) DNS servers

```yaml
---
- name: DNS1
  hosts: ns1.homelab.example.com
  become: true
  gather_facts: true
  roles:
    - role: ricsanfre.bind9
      bind9_authoritative: true
      bind9_forward: true
      bind9_recursor: true
      bind9_forward_servers:
        - 8.8.8.8
        - 4.4.4.4
      bind9_generate_ddns_key: true
      bind9_zones:
        - name: homelab.example.com
          type: primary
          secondary_servers:
            - 10.11.0.12
          ddns_update_key: ddnskey
          ddns_update_policy_grant: "zonesub any"
          default_ttl: 600
          serial: 2024101102
          refresh: 1D
          retry: 2H
          expire: 1000H
          # NS and other pre-formatted records values must be given as full qualified domain names, with or without final dot, but not relative to the zone
          primary: ns.homelab.example.com         # Optional, if you don't define it, firs NS is taken
          admin: admin.example.com
          ns_records:
            - ns1.homelab.example.com
            - ns2.homelab.example.com
          # RR values are either relative to the zone, either with a final dot when outside.
          rrs:
            - label: ns1
              type: A
              rdata: 10.11.0.11
            - label: ns2
              type: A
              rdata: 10.11.0.12
            - label: node1
              type: A
              rdata: 10.11.0.21
            - label: node2
              type: A
              rdata: 10.11.0.22
        - name: 11.10.in-addr.arpa
          type: primary
          default_ttl: 600
          serial: 2024101102
          refresh: 1D
          retry: 2H
          expire: 1000H
          # NS and other pre-formatted records values must be given as full qualified domain names, with or without final dot, but not relative to the zone
          primary: ns1.homelab.example.com         # Optional, if you don't define it, firs NS is taken
          admin: admin.homelab.example.com
          ns_records:
            - ns1.homelab.example.com.
            - ns2.homelab.example.com.
          # RR values are either relative to the zone, either with a final dot when outside.
          rrs:
            - label: 0.1
              type: PTR
              rdata: ns1.homelab.example.com
            - label: 0.2
              type: PTR
              rdata: ns2.homelab.example.com.
            - label: 0.21
              type: PTR
              rdata: node1.homelab.example.com.
            - label: 0.22
              type: PTR
              rdata: node2.homelab.example.com.

- name: DNS1
  hosts: ns2.homelab.example.com
  become: true
  gather_facts: true
  roles:
    - role: ricsanfre.bind9
      bind9_authoritative: true
      bind9_forward: true
      bind9_recursor: true
      bind9_forward_servers:
        - 8.8.8.8
        - 4.4.4.4
      bind9_zones:
        - name: homelab.example.com
          type: secondary
          primary_servers:
            - 10.11.0.11
        - name: 11.10.in-addr.arpa
          type: secondary
          primary_servers:
            - 10.11.0.11
```


License
-------

MIT

Author Information
------------------

Created by Ricardo Sanchez (ricsanfre)

