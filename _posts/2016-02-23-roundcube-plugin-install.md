---
layout: post
title: "Как установить плагин в Roundcube"
categories: php
description: "Как установить плагин в Roundcube"
keywords: "roundcube, composer, php"
---

Задача нетривиальная для пользователей, как ни странно.

Плагины ставятся с помощью composer – менеджера зависимостей для PHP. То есть, фактически, мы просто добавляем к текущей инсталляции Roundcube новую зависимость.

Для начала, перейдем в директорию Roundcube и установим сам composer:

```
cd /var/www/roundcube
curl -s https://getcomposer.org/installer | php
```

Скопируем шаблон конфига композера для Roundcube и откроем его для редактирования:

```
cp composer.json-dist composer.json
vim composer.json
```

Идем в [https://plugins.roundcube.net/](https://plugins.roundcube.net/), ищем там нужный нам плагин, например [https://plugins.roundcube.net/packages/roundcube/filters](https://plugins.roundcube.net/packages/roundcube/filters), добавляем строчку из описания плагина, следующую после require (например `require: "roundcube/filters": "dev-master"`) в секцию `require` нашего `composer.json`. В итоге она начинает выглядеть так (не забываем про запятые):

```
    "require": {
        "php": ">=5.3.7",
        "roundcube/plugin-installer": "~0.1.6",
        "pear-pear.php.net/auth_sasl": "~1.0.6",
        "pear-pear.php.net/net_idna2": "~0.1.1",
        "pear-pear.php.net/mail_mime": "~1.10.0",
        "pear-pear.php.net/net_smtp": "~1.7.1",
        "pear-pear.php.net/crypt_gpg": "~1.4.0",
        "roundcube/net_sieve": "~1.5.0",
        "roundcube/filters": "dev-master"
    },
```

Сохраняем файл, закрываем редактор, запускаем установку зависимостей: `php composer.phar install`

Композер выкачивает их, собирает, устанавливает, и т.д. По окончанию процесса он предложит добавить плагины в config/config.inc.php автоматически. Если не хотите, то можно сделать это вручную, добавив название плагина в параметр `$config['plugins']` в `config/config.inc.php`:

```
$config['plugins'] = array(
        'acl',
        'jqueryui',
        'filters',
);
```

**Неприятный момент**. Если вы используете PHP 5.3 (который как бы нормально поддерживается Roundcube), то ничерта у вас не соберется со стандартным composer.phar. Причина – net_smtp последней версии (1.7.1 на данный момент) требует PHP не ниже 5.4.0. Выход: заменить строчку

`"pear-pear.php.net/net_smtp": "~1.7.1",`

на 

`"pear-pear.php.net/net_smtp": "~1.6.3",`

Ну или обновиться до PHP 5.4+, т.к. 5.3 уже не поддерживается уже почти два года (5.4, впрочем, тоже).

## Полезные ссылки:

* [https://getcomposer.org/](https://getcomposer.org/)
* [https://plugins.roundcube.net/](https://plugins.roundcube.net/)