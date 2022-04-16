Install Python
=========

The role install python using raw module, so you can use it to prepare host for using Ansible.

Example Playbook
----------------

```yaml
---
- hosts: test_servers
  become: true

  gather_facts: false
  pre_tasks:
  - include_role:
      name: 'install-python'
  - name: Gather facts now
    setup:

  roles:
  - role: 'other-role'
```

Author Information
------------------

* [Vyacheslav Pospelov](https://gitlab.com/v.pospelov)
