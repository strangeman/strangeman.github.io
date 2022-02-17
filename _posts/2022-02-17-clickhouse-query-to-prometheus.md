---
layout: post
categories: devops monitoring clickhouse
title: "Как превратить кастомный запрос ClickHouse в метрики для Prometheus"
description: "Как превратить кастомный запрос ClickHouse в метрики для Prometheus"
keywords: "clickhouse, prometheus, metrics"
---
Потребовалось тут положить в Prometheus информацию о наличии таблиц на репликах, плюс о движке этих таблиц, чтобы иметь возможность отлавливать ситуации, когда создается таблица без репликации, либо она создается не на всех репликах.

Стандартными метриками эта информация не собирается. Но для решения этой проблемы в ClickHouse есть механизм
[`predefined_query_handler`](https://clickhouse.com/docs/en/interfaces/http/#predefined_http_interface), который позволяет отдавать по заданному URL результаты запроса в необходимом формате.

Как это выглядит в конфигах. В первую очередь, нам надо определиться с запросом. Для нашего кейса он будет выглядеть просто:
`SELECT database,name,engine FROM system.tables`

Теперь этот запрос надо обернуть в корректное форматирование. Для этого используются параметры запроса `FORMAT Template` и `SETTINGS format_template_*` (`format_template_row`, `format_template_resultset`, `format_template_rows_between_delimiter`). Эти параметры описаны в [документации](https://clickhouse.com/docs/ru/interfaces/formats/#format-template).

Нам надо задать `format_template_row` для форматирования каждой строки результата запроса под требования Prometheus. Для этого создаем файл `/var/lib/clickhouse/format_schemas/prometheus_table_engine.format`:

```
# TYPE clickhouse_db_engines counter
# HELP show table engines
clickhouse_db_engines{db_name=${database:Quoted}, table=${name:Quoted},engine=${engine:Quoted}} 1
```

Также нам надо переопределить дефолтное значение `format_template_rows_between_delimiter` на пустую строку (по дефолту стоит `\n` и это будет приводить к пустым строкам в списке метрик, что для Prometheus не подойдет).

Теперь нам надо прикрутить этот запрос к какому-нибудь URL в ClickHouse. Это делается через конфиг, через секцию `<http_handlers>`:

```
    <http_handlers>
        <rule>
            <url>/table_engines</url>
            <methods>GET</methods>
            <handler>
                <type>predefined_query_handler</type>
                <query>SELECT database,name,engine FROM system.tables FORMAT Template SETTINGS format_template_row = 'prometheus_table_engine.format', format_template_rows_between_delimiter = '' </query>
            </handler>
        </rule>
        <defaults/>
    </http_handlers>
```


Важно: внутри секции обязательно должно быть поле `<defaults/>`, иначе отломаются все стандартные эндпойнты.

Итого, при запросе на `/table_engines` нам вывалится набор метрик по таблицам и их движкам:

```
curl http://localhost:8123/table_engines

# TYPE counter
# HELP show table engines
clickhouse_db_engines{db_name='system', table='aggregate_function_combinators',engine='SystemAggregateFunctionCombinators'} 1
# TYPE counter
# HELP show table engines
clickhouse_db_engines{db_name='system', table='asynchronous_metrics',engine='SystemAsynchronousMetrics'} 1
# TYPE counter
# HELP show table engines
clickhouse_db_engines{db_name='system', table='build_options',engine='SystemBuildOptions'} 1
# TYPE counter
# HELP show table engines
clickhouse_db_engines{db_name='system', table='clusters',engine='SystemClusters'} 1
...
```

## Источник
* <https://clickhouse.com/docs/en/interfaces/http/#predefined_http_interface>
* <https://clickhouse.com/docs/en/interfaces/formats/#format-template>
