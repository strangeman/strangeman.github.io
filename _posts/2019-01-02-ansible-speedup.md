---
layout: post
categories: ansible
title: "Yet another 'How to speed up Ansible' tips"
description: "Yet another 'How to speed up Ansible' tips"
keywords: "ansible"
---
## SSH settings

ansible.cfg

```cfg
[ssh_connection]
ssh_args = -o PreferredAuthentications=publickey -o ControlMaster=auto -o ControlPersist=60s
pipelining = True
```

`PreferredAuthentications` helps to initiate the connection faster (without trying rare `gssapi-with-mic,gssapi-keyex,hostbased` methods).

`ControlMaster` and `ControlPersist` allows multiplexing existing connections to the server.

Useful links: [https://serverfault.com/questions/929222/ansible-where-do-preferredauthentications-ssh-settings-come-from](https://serverfault.com/questions/929222/ansible-where-do-preferredauthentications-ssh-settings-come-from)

## Facts caching

ansible.cfg

```cfg
[defaults]
gathering = smart
fact_caching=yaml
fact_caching_timeout=86400
fact_caching_connection=./ansible-cache
```

These options enable fact caching for 1 day and tell Ansible not gather facts in they already presented in the cache.

Useful links:

* [https://docs.ansible.com/ansible/latest/plugins/cache.html#enabling-fact-cache-plugins](https://docs.ansible.com/ansible/latest/plugins/cache.html#enabling-fact-cache-plugins)
* [https://docs.ansible.com/ansible/latest/reference_appendices/config.html#default-gathering](https://docs.ansible.com/ansible/latest/reference_appendices/config.html#default-gathering)

## Profiling

The most important part, in my opinion. There is no universal way, but I have some recomemndations:

* Try to make tasks idempotent as possible. No need to do a useless job;
* Delete duplicating tasks. For example, you're may try to enable the firewall every time when you're installing something. These tasks may be idempotent, but they still consume execution time;
* Use something like Ara to visualize the playbook execution. Ara provides a good web interface and allows to sort tasks for execution time.

Useful links: [https://github.com/openstack/ara](https://github.com/openstack/ara)