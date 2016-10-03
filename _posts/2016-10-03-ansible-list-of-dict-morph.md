---
layout: post
categories: ansible
title: "Как преобразовать один массив хэшей в другой"
description: "Как преобразовать один массив хэшей в другой"
keywords: "ansible"
---

## Задача

Есть в Ansible массив хэшей с данными о сервисах:

```
apps:
  - app_name: "app1"
    db_name: "db1" 
    db_owner: "own1"
    another_key: "somevalue"
  - app_name: "app2"
    db_name: "db2" 
    db_owner: "own2"
    another_key2: "somevalue2"
```

Я хочу этот массив преобразовать в другой, для того чтобы передать плейбуку для создания баз:

```
databases:
  - name: "db1" 
    owner: "own1"
  - name: "db2" 
    owner: "own2"
```

На голом Python это было бы легко, а вот с Jinja2 пришлось повозиться немного, в основном из-за постоянных истерик YAML на не нравящиеся ему скобочки да кавычки.

## Итоговый код

```yaml
{% raw %}pre_tasks:
  - name: set dict with postgresql bases
    set_fact: "postgres_bases={{ postgres_bases|default([]) + [{ 'name': item.db_name, 'owner': item.db_owner }] }}"
    with_items:  "{{ apps }}"{% endraw %}
```

P.S. Дурацкий Jekyll сходит с ума от двойных фигурных скобок. :-/