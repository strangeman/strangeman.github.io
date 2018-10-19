---
layout: post
categories: python
title: "Анонимизация данных в PostgreSQL базе с помощью Python"
description: "Анонимизация данных в PostgreSQL базе с помощью Python"
keywords: "python, anonimize"
---
# Table Of Contents

* TOC
{:toc}

# Постановка задачи

Есть дампы баз микросервисов с прода. Перед заливкой их на тест (чтобы гонять тестовую среду на приближенных к боевым объемах данных) хочется заменить там логины, имена, фамилии и адреса почты. Рандомно генерировать не подойдет, тестировщикам будет неудобно работать с Аваолртпщр Пватващитвап. Соответственно, надо взять какое-то решение для генерации фейковых данных.

Помимо этого, после замены базы должны остаться консистентными, во всех базах один и тот же id должен соответствовать одному и тому же пользователю. Также должен быть whitelist пользователей, которых не надо обновлять. Ну и плюс при добавлении баз это все должно довольно легко расширяться.

Писать буду на Python, ибо быстро и куча библиотек на все случаи жизни.

# Популярные библиотеки

В ходе подготовки нашлись две популярные библиотеки для генерации фейков:

* [faker](https://github.com/joke2k/faker)
* [mimesis](https://github.com/lk-geimfari/mimesis)

При тестировании выяснилось, что mimesis работает в несколько раз быстрее чем faker при одинаковом, в общем-то, функционале.

# Реализация

Обе библиотеки обладают относительно ограниченным набором данных. В таблицах стояло ограничение на уникальность логина и емейла и в итоге через некоторое время записи начинали повторяться и все падало к хренам. Пришлось добавлять рандомный постфикс к логинам и емейлам. В итоге получился примерно такой класс для генерации фейкового аккаунта:

```python
class MimesisPerson:
  def __init__(self):
    self.first_name = Person('ru').name()
    self.last_name = Person('ru').last_name()
    self.username = Person('ru').username() + "_" + ''.join(random.choice(ascii_lowercase + digits) for _ in range(4))
    self.email = self.username + "_" + Person('ru').email()
```

Целиком скрипт выглядит примерно так:

```python
import psycopg2
import psycopg2.extras
from mimesis import Person

import random
from string import Template, ascii_lowercase, digits

import config

class MimesisPerson:
  def __init__(self):
    self.first_name = Person('ru').name()
    self.last_name = Person('ru').last_name()
    self.username = Person('ru').username() + "_" + ''.join(random.choice(ascii_lowercase + digits) for _ in range(4))
    self.email = self.username + "_" + Person('ru').email()

class Connection:
  def __init__(self, conn_string, sql):
    self.conn_string = conn_string
    self.sql = sql
    self.connector = psycopg2.connect(self.conn_string)
    self.cursor = self.connector.cursor()

  def update_data(self, person, user_id):
    for query in self.sql:
      self.cursor.execute(query.substitute(user_login=person.username, email=person.email,
                                                     first_name=person.first_name, last_name=person.last_name, user_id=user_id))

# update user info in base1 and base2
base1 = Connection(conn_string=config.base1_conn_string, sql=[
       Template("UPDATE users SET login = '$user_login', email = '$email', first_name = '$first_name', last_name = '$last_name' WHERE id = '$user_id'")
     ])
base2 = Connection(conn_string=config.base2_conn_string, sql=[
       Template("UPDATE users SET username = '$user_login', mail = '$email', first_name = '$first_name', last_name = '$last_name' WHERE id = '$user_id'")
     ])

base_list = [base1, base2]

id_cur = base1.connector.cursor(cursor_factory=psycopg2.extras.DictCursor)
id_cur.execute('SELECT id FROM users')

bulk = 0
entry = 0

for row in id_cur:
  person = MimesisPerson()

  if row["id"] in config.id_whitelist:
    print("Skipping user_id %s"%(row["id"]))
    continue

  try:
    for con in base_list:
      con.update_data(person, row["id"])
  except Exception as e:
    print("Error on %i entry in batch, exiting..."%(bulk))
    print(e)
    break

  bulk+=1
  entry+=1

  if bulk > 100:
    for con in base_list:
      con.connector.commit()
    bulk=0

  if entry % 1000 == 0:
    print("Processed %i entries..."%(entry))

for con in base_list:
  con.connector.commit()
  con.connector.close()
```

Новые базы при надобности просто добавляются в массив и по ним тоже начинает бегать скрипт.