Ansible Firewall Role
=========

I tried several firewall roles out there but none satisfied the requirements I had:

- Support virtually all iptables rules from the start
- Allow granular rules addition/overriding for specific hosts
- Easily inject variables in the rules
- Allow rules ordering
- Simplicity (not having to learn how role variables would generate the rules)
- Persistence (reload the rules at boot)

This role is an attempt to solve these requirements.

It supports **ipv4** and **ipv6** on Debian distributions. ipv6 rules are not configured by default. If you which to use them, don't forget to set `firewall_v6_configure` to `true`.

Requirements
------------

* Ansible 2.4.0.0
* `iptables` (installed by default on all official Debian distributions except Debian 11 "Bullseye")

Role Variables
--------------

`defaults/main.yml`:

```yaml
---
firewall_v4_configure: true
firewall_v6_configure: false
# ...

firewall_v4_flush_rules:
# ...
firewall_v4_default_rules:
# ...
firewall_v4_group_rules: {}
firewall_v4_host_rules: {}

firewall_v6_flush_rules:
# ...
firewall_v6_default_rules:
# ...
firewall_v6_group_rules: {}
firewall_v6_host_rules: {}
```

The keys to the `*_rules` dictionaries, except the flush rules, can be anything. They are only used for rules **ordering** and **overriding**. On rules generation, the keys are sorted alphabetically. That's why I chose here the 001s and 999s.

Those defaults will generate the following script to be executed on the host. Script paths are stored in variables:
- `iptables_v4_config` for IPv4 rules
- `iptables_v6_config` for IPv6 rules
- `iptables_custom_config` for rules that are defined without ansible. If you want to allow them, don't forget to set `firewall_custom_configure` to `true`.

As you can see, you have complete control over the rules syntax.

`$ iptables -L -n` on the host then shows...

```
Chain INPUT (policy DROP)
target     prot opt source               destination
ACCEPT     all  --  127.0.0.0/8          127.0.0.0/8
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 8
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:22 state NEW,RELATED,ESTABLISHED

Chain FORWARD (policy DROP)
target     prot opt source               destination

Chain OUTPUT (policy DROP)
target     prot opt source               destination
ACCEPT     all  --  127.0.0.0/8          127.0.0.0/8
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 0
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            state NEW,RELATED,ESTABLISHED
```

Now that takes care of the default rules. What about overriding?

You can change the rules for specific hosts and groups instead of re-defining everything. Rules in `firewall_v4_host_rules` will be merged with `firewall_v4_group_rules`, and then the result will be merged back with the defaults. Same thing for ipv6.

This allows 3 levels of rules definition and overriding. I simply chose the names to match how the variable precedence works in Ansible (`all` -> `group` -> `host`). See the example playbook below to see rules overriding in action.

Example Playbook (ipv4)
----------------

```yaml
---
- hosts: all
  become: true

  roles:
    - role: 'firewall'
```

in `group_vars/all.yml` you could define the default rules for all your hosts:

```yaml
firewall_v4_default_rules:
  001 default policies:
    - -P INPUT DROP
    - -P OUTPUT DROP
    - -P FORWARD DROP
  002 allow loopback:
    - -A INPUT -i lo -j ACCEPT
  003 allow ping replies:
    - -A INPUT -p icmp --icmp-type echo-request -j ACCEPT
    - -A OUTPUT -p icmp --icmp-type echo-reply -j ACCEPT
  100 allow established related:
    - -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
  200 allow ssh limiting brute force:
    - -I INPUT -p tcp -d {{ ansible_default_ipv4.address }} --dport 22 -m state --state NEW -m recent --set
    - -I INPUT -p tcp -d {{ ansible_default_ipv4.address }} --dport 22 -m state --state NEW -m recent --update --seconds 60 --hitcount 4 -j DROP
  999 allow output:
    - -A OUTPUT --match state --state NEW,ESTABLISHED,RELATED -j ACCEPT
```

in `group_vars/webservers.yml` you would open up port 80:

```yaml
firewall_v4_group_rules:
  400 allow web traffic:
    - -A INPUT -p tcp --dport 80 -j ACCEPT
```

in `host_vars/secureweb.yml` you would want to open https as well and remove ssh logins:

```yaml
firewall_v4_host_rules:
  400 allow web traffic:
    - -A INPUT -p tcp --dport 80 -j ACCEPT    # need to redefine this one as well because the whole key is overwritten
    - -A INPUT -p tcp --dport 443 -j ACCEPT
  200 allow ssh limiting brute force: []
```

To "delete" rules, you just assign an empty list to an existing dictionary key.

To summarize, rules in `firewall_v4_host_rules` will overwrite rules in `firewall_v4_group_rules`, and then rules in `firewall_v4_group_rules` will overwrite rules in `firewall_v4_default_rules`.

You can play with the rules and see the generated script on the host at the following location: `firewall.sh` and `firewall_v6.sh`.

Dependencies
------------

none
