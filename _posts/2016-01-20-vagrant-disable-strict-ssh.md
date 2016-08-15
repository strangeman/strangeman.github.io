---
layout: post
title: "Как избавиться от WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED! при работе с Vagrant"
categories: hints
description: "Как избавиться от WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED! при работе с Vagrant"
keywords: "vagrant, ssh"
---
При разработке и тестировании часто приходится пересоздавать Vagrant-машину с нуля. Так как айпишник у нее не меняется, после пересоздания машины и при попытке к ней зацепиться мы получаем истерику от SSH:

```
$ ssh root@192.168.99.99
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@ WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED! @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
The fingerprint for the ECDSA key sent by the remote host is
9f:56:58:8c:08:bc:1a:b1:3f:fa:c8:2e:0a:97:6d:19.
Please contact your system administrator.
Add correct host key in /home/strangeman/.ssh/known_hosts to get rid of this message.
Offending ECDSA key in /home/strangeman/.ssh/known_hosts:340
 remove with: ssh-keygen -f "/home/strangeman/.ssh/known_hosts" -R 192.168.99.99
ECDSA host key for 192.168.99.99 has changed and you have requested strict checking.
Host key verification failed.
```

Решается эта проблема просто: мы начинаем использовать /dev/null для хранения KnownHosts и отключаем строгую проверку ключа. Естественно, в целях безопасности, стоит это делать только для IP ваших Vagrant-машин (я, например, использую 192.168.99.0/24). Добавляем в начало `~/.ssh/config` следующие строки:

```
Host 192.168.99.*
 StrictHostKeyChecking no
 UserKnownHostsFile=/dev/null
```