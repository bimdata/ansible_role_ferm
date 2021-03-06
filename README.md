Ansible role Ferm
=========

This role installs and configures Ferm.

Requirements
------------

* This role is only tested with Ansible >= 2.9.

Role Variables
--------------

You can find more information about each option in the [Ferm documentation](
http://ferm.foo-projects.org/download/2.6/ferm.html).

Variables used for the installation:

| Variables     | Default value | Description                                     |
|---------------|---------------|-------------------------------------------------|
| ferm_pkg_name | ferm          | The ferm APT package name.                      |
| ferm_svc_name | ferm          | The ferm service name to start/stop the daemon. |

Variables used for the general configuration:

| Variables           | Default value       | Description                   |
|---------------------|---------------------|-------------------------------|
| ferm_main_conf_file | /etc/ferm/ferm.conf | Ferm main configuration path. |
| ferm_rules_path     | /etc/ferm/ferm.d    | Ferm rules directory path.    |

The default rules will be store in the main configuration file, that will then
load all rules define in the configuration directory. This enable you to
define other rules not manage by this role (if `ferm_delete_unknown_rules` is set
to `false`).

Variable used for rules configuration:

| Variables                     | Default value | Description                                                  |
|-------------------------------|---------------|--------------------------------------------------------------|
| ferm_delete_unknown_rulefiles | true          | Delete the rules in ferm_rules_path not manage by this role. |
| ferm_default_domains          | ['ip', 'ip6'] | When a rule do not specify a domain, this one is used.       |
| ferm_default_table            | filter        | When a rule do not specify a table, this on is used.         |

Variable used for rules definition:

| Variables                 | Default value                                                                     | Description                                       |
|---------------------------|-----------------------------------------------------------------------------------|---------------------------------------------------|
| ferm_input_policy         | DROP                                                                              | Ferm default input policy.                        |
| ferm_output_policy        | ACCEPT                                                                            | Ferm default output policy.                       |
| ferm_forward_policy       | DROP                                                                              | Ferm default forward policy.                      |
| _ferm_rules               | "{{ lookup('template', 'get_vars.j2', template_vars=dict(vartype='rule')) }}"     | List of dictionaries defining all ferm rules.     |
| _ferm_vars                | "{{ lookup('template', 'get_vars.j2', template_vars=dict(vartype='var')) }}"      | List of dictionaries defining all ferm variables. |
| _ferm_functions           | "{{ lookup('template', 'get_vars.j2', template_vars=dict(vartype='function')) }}" | List of dictionaries defining all ferm functions. |
| _ferm_hooks               | "{{ lookup('template', 'get_vars.j2', template_vars=dict(vartype='hook')) }}"     | List of dictionaries defining all ferm hooks.     |

**In most cases, you should not modify the variables that start with an '_'.**
Templating is used to build these lists with other variables.
* `_ferm_rules` will aggregate all the variable whose name matches this regex: `^ferm_.+_rule(s)?$`
* `_ferm_vars` will aggregate all the variable whose name matches this regex: `^ferm_.+_var(s)?$`
* `_ferm_functions` will aggregate all the variable whose name matches this regex: `^ferm_.+_function(s)?$`
* `_ferm_hooks` will aggregate all the variable whose name matches this regex: `^ferm_.+_hook(s)?$`

Each variables matching these regexes must be:
  - a dictionary defining one rule/variable/function/hook **or**
  - a list of dictionaries defining one or more rules/variables/functions/hooks.

It allows you to define variables in multiple group_vars and cumulate them for
hosts in multiples groups without the need to rewrite the complete list.

Variables used to define the default ruleset:
- this one configure the default ruleset for the INPUT table:
  ```
  ferm_default_inputs:
    - "policy {{ ferm_input_policy }};"
    - interface lo ACCEPT;
    - "mod conntrack ctstate (RELATED ESTABLISHED) ACCEPT;"
  ```
- this one configure the default ruleset for the OUPUT table:
  ```
  ferm_default_outputs:
    - "policy {{ ferm_output_policy }};"
    - outerface lo ACCEPT;
    - "mod conntrack ctstate (RELATED ESTABLISHED) ACCEPT;"
  ```
- this one configure the default ruleset for the FORWARD table:
  ```
  ferm_default_forwards: []
  ```

**Debian 11 use `iptables-nft` by default and it's not supported by ferm.**
Since Debian 11, ferm ignore alternative setting and force the use of
iptables-legacy (https://github.com/MaxKellermann/ferm/issues/47)

| Variables           | Default value              | Description                                       |
|---------------------|----------------------------|---------------------------------------------------|
| ferm_iptables_path  | /usr/sbin/iptables-legacy  | Path of iptables-legacy.                          |
| ferm_ip6tables_path | /usr/sbin/ip6tables-legacy | Path of ip6tables-legacy.                         |
| ferm_arptables_path | /usr/sbin/arptables-legacy | Path of arptables-legacy.                         |
| ferm_ebtables_path  | /usr/sbin/ebtables-legacy  | Path of ebtables-legacy.                          |

**These variables are only used on Debian host with version > 10.**
It will configure the OS with the `alternative` system to set the value of `ferm_iptables_path`,
`ferm_ip6tables_path`, `ferm_arptables_path` and `ferm_ebtables_path` as the default iptables command.

### Variable definitions
A ferm variable can be define like this:
```
ferm_webports_var:
  name: web_ports
  content:
    - 80
    - 443

ferm_hosts_vars
  - name: web_front_addr
    content:
      - 172.29.10.100
      - 2a01:baaf::100

```

### Hook definitions
A ferm hook can be define like this:
```
ferm_fail2ban_hooks:
  - comment: Fail2ban hooks
    content: post "type fail2ban-server > /dev/null && (fail2ban-client ping > /dev/null && fail2ban-client reload > /dev/null || true) || true";
  - content: flush "type fail2ban-server > /dev/null && (fail2ban-client ping > /dev/null && fail2ban-client reload > /dev/null || true) || true";
```

### Rule definitions
A rule can be define in two way:
```
ferm_web_rules:
  - name: "web_server"
    content:
      - domains: ['ip']        # Can be omit, ferm_default_domains will be used
        chains: ['INPUT']      # Can be omit, ferm_default_table will be used
        rules:
          - proto tcp dport ($web_ports) mod comment comment "web server" ACCEPT

```

or you can define raw rules:
```
ferm_web_rules:
  - name: "web_server"
    raw_content: |
      domain (ip) table filter {
        chain (INPUT) {
          proto tcp dport ($web_ports) mod comment comment "web server" ACCEPT;
        }
      }
```

### Function definitions
A ferm function can be define like this:
```
ferm_dnat_function:
  comment: "Easy DNAT (DNAT+filter rules)"
  content: |
    @def &EASY_DNAT($wan_ip, $proto, $port, $dest) = {
      domain ip table nat chain PREROUTING daddr $wan_ip proto $proto dport $port DNAT to @ipfilter($dest);
      domain (ip ip6) table filter chain FORWARD outerface $dmz_iface daddr $dest proto $proto dport $port ACCEPT;
    }
```

Then you need to use a raw rule to use it, something like:
```
ferm_dnat_rules:
  - name: "80-dnat-rules"
    raw_content: |
      # HTTP(S) web_front
      &EASY_DNAT($main_public_ip, tcp, (80 443), $web_front_addr);
```

### Docker example
Docker and other software may want to manage their own iptables rules. This is a
possible, with some limitations. Here an example for Docker:

```
# No rule can't be defined in FORWARD before the rule use to preserve all
# rules configured by docker
ferm_default_forwards: []

# Preserve docker rules
ferm_docker_preserve_rules:
  - name: 99-docker-users.ferm
    content:
      - domains: ['ip']
        chains: ['DOCKER-USER']
        rules:
          - "RETURN;"
  - name: 00-docker-preserve.ferm
    content:
      - domains: ['ip']
        chains:
          - DOCKER
          - DOCKER-INGRESS
          - DOCKER-ISOLATION-STAGE-1
          - DOCKER-ISOLATION-STAGE-2
          - FORWARD
        rules:
          - "@preserve;"
      - domains: ['ip']
        table: nat
        chains:
          - DOCKER
          - DOCKER-INGRESS
          - PREROUTING
          - OUTPUT
          - POSTROUTING
        rules:
          - "@preserve;"
```

[@preserve](http://ferm.foo-projects.org/download/2.5/ferm.html#BASIC-KEYWORDS)
is a special word use by `ferm`. It will save the previous rules with `iptables-save`
and then extracts all rules for the preserved chains ans insert them into the new rules.

Dependencies
------------

None

Example Playbook
----------------

in `group_vars/all.yml`:
```
ferm_webports_var:
  name: web_ports
  content:
    - 80
    - 443
```

in `group_vars/web.yml`:
```
ferm_web_rules:
  - name: "web_server"
    content:
      - chains: ['INPUT']
        rules:
          - proto tcp dport ($web_ports) mod comment comment "web server" ACCEPT
```

in `group_vars/router.yml`:
```
ferm_interface_vars:
  - name: wan_iface
    content: ['eth0']
  - name: dmz_iface
    content: ['eth1']
  - name: lan_iface
    content:
      - eth2
      - eth3

ferm_ips_vars:
  - name: main_public_ip
    content: ['1.2.3.4']
  - name: web_front_addr
    content:
      - 10.0.0.100
      - 2a01:baaf::100

ferm_dnat_function:
  comment: "Easy DNAT (DNAT+filter rules)"
  content: |
    @def &EASY_DNAT($wan_ip, $proto, $port, $dest) = {
      domain ip table nat chain PREROUTING daddr $wan_ip proto $proto dport $port DNAT to @ipfilter($dest);
      domain (ip ip6) table filter chain FORWARD outerface $dmz_iface daddr $dest proto $proto dport $port ACCEPT;
    }

ferm_dnat_rules:
  - name: "80-dnat-rules"
    raw_content: |
      # HTTP(S) web_front
      &EASY_DNAT($main_public_ip, tcp, $web_ports, $web_front_addr);

```

in `playbook.yml`:
```
- hosts: all
  gather_facts: True
  become: yes
  roles:
    - bimdata.ferm
```

License
----------------

BSD

Author Information
------------------

[BIMData.io](https://bimdata.io/)
