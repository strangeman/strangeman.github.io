---
layout: post
categories: ansible
title: "Обратная совместимость модулей между Ansible 2.1 и Ansible 2.0"
description: "Обратная совместимость модулей между Ansible 2.1 и Ansible 2.0"
keywords: "ansible, python"
---

## Вводная (TL;DR)

Столкнулся тут с неожиданно неприятным поведением Ansible, надо бы задокументировать опыт. В 2.1 изменился алгоритм обработки передаваемых модулю аргументов без явного указания типа. Если в 2.0 они передавались "как есть", передали вы ему list или dict, так он таким и остался, то в 2.1 такие аргументы начали конвертироваться в строку. Вот [коммит](https://github.com/ansible/ansible/commit/62765858825173dd532827fe4c1dfb246eb8db47#diff-90085fdcec6ed8b273ba885eaee60328). Для передачи аргументов "как есть" ввели новый тип `raw`.

## Проблема

Ничего не предвещало беды, я выкатывал новую версию приложения (хорошо хоть на тестовое окружение), Ansible отработал без ошибок. Но приложение не поднялось. Полезли с разработчиками на сервер, визуально все лежит на своих местах, в логах малоинформативные ошибки, дескать не могу загрузить такие-то модули (хотя они лежат на своих местах).

Наконец, через некоторое время, нашли проблему. Кусок json-конфига вместо 

```yaml
{
  "existing_key": ["first", "second"]
}
```

выглядел вот так (кавычки, обрамляющие список):

```yaml
{
  "existing_key": "['first', 'second']"
}
```

Для работы с json у нас используется самописный [модуль](https://github.com/UnitedTraders/ansible-role-jsonpatch). Стало очевидно что дело в нем. Опытным путем установил, что под 2.0 он работал корректно, а под 2.1 подложил свинью. 

## Решение

Документация по написанию модулей для Ansible крайне бедна, поэтому почти сразу пришлось лезть в код класса `AnsibleModule`, что в `ansible/lib/ansible/module_utils/basic.py`. Выяснил изменения в преобразовании типов (что описано во вводной), добавил к нужному аргументу тип `raw`, под 2.1 все заработало корректно, зато сломалось в 2.0:

```
TASK [Patch JSON - add key] ****************************************************
fatal: [localhost]: FAILED! => {"changed": false, "failed": true, "msg": "implementation error: unknown type raw requested for value"}
        to retry, use: --limit @test.retry
```

Ломать совместимость не хотелось, поэтому в результате родился вот такой код:

```python
def main():

    module = AnsibleModule(
        argument_spec=dict(
            value=dict(required=False, default=None)
        )
    )

    if 'raw' in module._CHECK_ARGUMENT_TYPES_DISPATCHER:
        module = AnsibleModule(
            argument_spec=dict(
                value=dict(required=False, default=None, type='raw'),
            )
        )
```
Проверяем, есть ли тип ﻿⁠⁠⁠⁠`raw﻿⁠⁠⁠⁠` среди поддерживаемых, если есть, то переопределяем экземпляр класса ﻿⁠⁠⁠⁠`AnsibleModule﻿⁠⁠⁠⁠` с ним. 

Не знаю, насколько много людей затрагивает такая проблема, но в нашем случае это изменение в поведении оказалось критичным. Большинство, наверное, и так передают в модуль строки, в этом же случае нам надо передавать самые разные типы: строки, числа, массивы (мало ли что в json понадобится вставить) и преобразование в строку для всего ломает логику модуля.