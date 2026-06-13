## AndrewGodGivens /ansible_ansible_nginx_exporter 
============

An Ansible role which installs and configures ansible nginx_exporter on Linux

============

## Requirements

Ansible 2.8+

============

## Role Variables

You can see all vars in defaults/main.yml vars file.

## Example Playbook

```yaml
- name: Ensure ansible_nginx_exporter
  hosts: ansible_nginx_exporters
  remote_user: root

  roles:
    - ansible_nginx_exporter
  
```
