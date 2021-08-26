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

### Rules definition
A rule can be define in two way:
```
ferm_rabbitmq_rules:
  - name: "rabbitmq_server"
    content:
      - domains: ['ip']        # Can be omit, ferm_default_domains will be used
        chains: ['INPUT']      # Can be omit, ferm_default_table will be used
        rules:
          # the final ';' is optional, it will be add if it's not here
          - proto tcp dport 5672 mod comment comment "rabbitMQ client" ACCEPT
          - proto tcp dport 15672 mod comment comment "rabbitMQ management" ACCEPT
      - chains: ['INPUT', 'OUTPUT']
        rules:
          - proto tcp dport 25672 mod comment comment "rabbitMQ internode" ACCEPT
```

or you can define raw rules:
```
ferm_rabbitmq_rules:
  - name: "rabbitmq_server"
    raw_content: |
      domain (ip) table filter {
        chain (INPUT) {
          proto tcp dport 5672 mod comment comment "rabbitMQ client" ACCEPT;
          proto tcp dport 15672 mod comment comment "rabbitMQ management" ACCEPT;
        }
      }
      domain (ip ip6) table filter {
        chain (INPUT OUTPUT) {
          proto tcp dport 25672 mod comment comment "rabbitMQ internode" ACCEPT;
        }
      }
```


For example:
```
```

will create:
```
```

It allows you to define variables in multiple group_vars and cumulate them for
hosts in multiples groups without the need to rewrite the complete list.


Dependencies
------------

None

Example Playbook
----------------

License
----------------

BSD

Author Information
------------------

[BIMData.io](https://bimdata.io/)
