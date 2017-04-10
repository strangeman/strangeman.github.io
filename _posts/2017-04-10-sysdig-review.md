---
layout: post
categories: monitoring
title: "Впечатления от системы мониторинга Sysdig"
description: "Впечатления от системы мониторинга Sysdig"
keywords: "sysdig, monitoring"
---

# Общая информация

Ребята из [Sysdig](http://www.sysdig.org/) делают два продукта: опенсорсную [утилиту](https://github.com/draios/sysdig) с htop-like UI для получения подробной информации о том, что вообще происходит в системе и очень продвинутую облачную [систему](https://sysdig.com/product/) мониторинга, которая из коробки умеет собирать кучу метрик, детектить установленные приложения, работать с контейнерами (как с одиночными контейнерами, так и с compose/swarm/k8s/mesos оркестрацией). Обе этих системы, насколько я понимаю, работают на одном ядре, которое собирает информацию о системе на низком уровне (в процессе установки подсовывается, в том числе, свой модуль ядра) и либо выплевыает это все в ncurses-UI, либо шлет на сервер.

# Опенсорсная утилита

По сути, это швейцарский нож, включающий в себя strace, htop, lsof, tcpdump, iftop, docker client и, наверное, что-нибудь еще. Можно смотреть статистику потребления ресурсов (cpu, ram, сеть, диск) по процессам/контейнерам, что и куда шлет по сети каждый процесс/контейнер, ошибки и логи на уровне системы/приложения, трейсы, открытые файловые дескрипторы и так далее и тому подобное. Очень полезной фишкой является возможность записи всего происходящего в системе в специальный файл для последующего анализа. Вот тут есть [видео](https://www.youtube.com/watch?v=UJ4wVrbP-Q8) с примерным обзором возможностей, рекомендую посмотреть.

Из минусов - эта вундервафля (при запущенном curses-клиенте) жрет немало CPU, особенно на серверах с большим количеством приложений (и ими вызываемых событий). Но тут, наверное, ничего не поделаешь.

# Облачный мониторинг

Мониторинг умеет все тоже самое, что консольная утилита, по части сбора общесистемной информации. Помимо этого, клиент мониторинга детектит установленные на сервере приложения (в том числе и в контейнерах) и собирает по ним метрики, например статистику запросов (и время их выполнения) MySQL/Postgres, время отклика Nginx/HAProxy, состояние JVM, ну и так далее. Для значительного числа интеграций не требуется никакого ручного вмешательства, все начинает собираться автоматически. Есть куча predefined дашбордов и алертов, которые можно подкрутить под себя, при желании.

Также, клиент умеет собирать информацию об архитектуре микросервисов, запущенных поверх оркестраторов контейнеров (и красиво ее визуализировать), интегрироваться с AWS ECS, алертить в Slack/PagerDuty/VictorOps/OpsGenie.

Есть полноценный (с ограничением в 15 хостов) триал на две недели. Вполне достаточно, чтобы покрутить его со всех сторон и принять решение, насколько оно необходимо.

Есть, увы, и минусы. Основных претензий три:

* Тяжелый клиент. В системных требованиях указано, что он может отъедать до 512MB RAM, 5-7% CPU, плюс генерирует весьма хороший сетевой трафик. Не знаю, насколько сильно это будет аффектить высоконагруженные сервера, но, думаю, это может быть заметно.
* Веб-интерфейс - панель космолета. На странице информации о хосте показываются абсолютно все возможные интеграции и сервисы, даже те, которых и в помине нет на этом сервере. В итоге, на поиск нужной информации уходит довольно много времени. Хотелось бы переключатель, которые будет скрывать интеграции, по которым не приходит никаких данных. 
* Цена. 20-30$ за хост в месяц, на мой взгляд, ограничивают использование такого мониторинга большими серверами. Мониторить какой-нибудь K8S/DCOS-кластер из 5-7 здоровых машин такой штукой очень круто, а вот для россыпи мелких серверов выйдет очень накладно.

# Выводы 

Опенсорсную часть я водружу на все сервера, думаю. Очень удобная утилита для траблшутинга. А вот мониторинг пока стоит отложить до миграции в какой-нибудь K8S/DCOS, в текущем виде получается дороговато. Метрики приложений/системы можно пособирать и Prometheus'ом, а сложная контейнеризация у нас пока не используется. Но когда начнется плотная работа с контейнерами - sysdig окажется очень полезен.