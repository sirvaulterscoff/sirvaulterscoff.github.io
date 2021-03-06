
#articles #sql #programming #patterns

  

# О синхронизации потоков через СУБД

Есть целый класс задач, когда необходимо выполнить какую-то обработку данных на нескольких узлах (экземплярах микросервиса) при этом необходимо обеспечить синхронизацию этих независимых узлов так, чтобы только один обрабатывал порцию данных в один момент времени. Kafka из коробки предлагает механизм партицирования, но к сожалению такое решение не всегда применимо

  

## Формулировка задачи

  

В таблицу Tasks помещаются строки для обработки, каждая содержит ID, статус, дата создания. Необходимо сделать так, чтобы каждый из экземпляров микросервиса обрабатывал только одну запись, и каждая уникальная запись обрабатывалась только одним микросервисом.

Задачи выполняются по мере их поступления в порядке FIFO (т.е. сортирует по дате создания).

  

Первое требование выполняется элементарно: достаточно лишь  обработку строк таблицы выполнять в одном потоке. А вот второе требование чуть сложнее. Рассмотрим варианты:

  

  

### Пессимистчная блокировка

Тут все очень просто: многие БД предоставляют доступ к инструментам блокировки строк и таблиц, например [postgres предоставляет  семантику select for update](https://www.postgresql.org/docs/9.1/explicit-locking.html)

Этот способ вполне ок, до тех пор пока вы можете обеспечить минимальный период блокировки.

Наш пример может выглядеть так:

  

```sql

select *

from Tasks

where  status='READY'

order by created_date asc

limit  1

FOR  UPDATE  SKIP  LOCKED;

```

Тут два ключевых момента. С помощью `FOR UPDATE` мы указываем, что необходимо использовать блокировку строки в EXCLUSIVE режиме. С помощью `SKIP LOCKED` мы указываем на необходимость пропустить заблокированные строки.

  

Таким образом, полная обработка в нашем случае будет выглядеть как-то так:

```sql

select ... FOR  UPDATE  SKIP  LOCKED;

  

-- some row processing

--

-- end processing

  

update Tasks set  status='COMPLETED'  where  id=?;

commit;

```

Несложно понять, что граница блокировки здесь охватывает весь блок, в т.ч. обработку записей

  

### Оптимистичная блокировка

В отличие от пессимистичной (которая исходит из предположения, что эту строку точно кто-то кроме нас попытается обработать) оптимистичная блокировка рассчитана на более редкие коллизии. На то она и оптимистичная. Вариантов реализации оптимистичной блокировки масса, покажу лишь общий принцип.

Суть заключается в том, что мы не используем эксклюзивные блокировки, предоставляя БД решать какая транзакция может накатиться, а какая нет. Самый простой и, пожалуй, распространненый способ реализации оптимистичной блокировки - колонка с версией записи. В качестве версии может выступать как число-счетчик изменений, так и timestamp последнего изменения записи.

```

select  id, version

from Tasks

where  status='READY'

order by created_date asc

limit  1;

  

-- some row processing

--

-- end processing

  

update Tasks set  status='COMPLETED', version=version+1

where  id=? and  version=?;

commit;

```

  

На что тут обратить внимание:

- мы выбираем не только идентификатор строки, но и ее версию

- после обработки строки мы в условие дописываем критерий version=? куда передаем значение версии строки, вычитанное на первом этапе

Таким образом, если какая-то транзакция обновила строку, пока мы ее обрабатывали update не выполнит никаких изменений, т.к. строка просто не подойдет по условию

  

Тут важный момент: все обновления таблицы должны использовать версионированную колонку.

Как несложно заметить это решение не создает экслюзивной блокировки, но и не помогает в достижении нашей цели. Более того, в этом примере результат обработки, фактически, оказывается потеряным, если параллельная транзакция обновила строку.

  

Но, вернемся к нашей цели. Напомню, нам нужно позволить двум экземплярам сервиса одновременно обрабатывать разные строки.

Тут возможен следующий алгоритм действий

1. Каждый из микросервисов отбирает набор строк для обработки:

```

select  id, version

from Tasks

where  status='READY'

order by created_date asc

limit N;

```

Значение N должно определяться количеством потоков, которые могут одновременно обрабатывать строки, т.к. при использовании этого подхода часть отобранных строк может успеть попасть в обработку и нам придется их пропустить

2. По каждой строке пытаемся обновить статус

```

update Tasks

set version=X, status='PROCESSING'

where id=? and version=X

commit;

```

Если в результате апдейта не была изменена ни одна строка (это можно выяснить, например, использую семантику [returning](https://postgrespro.ru/docs/postgresql/9.5/dml-returning)), то необходимо перейти к следующей строке

  

#### Что важно понимать

В этом примере нет долгоживущих эксклюзивных блокировок и долгих транзакций. Вместо этого тут короткие и частые транзакции. Кроме того, транзакция перестает содержать `unit of work` а используется для присвоения промежуточного статуса. Тут важно понимать, что нужен какой-то механизм recovery на тот случай, если микросервис не закончит обработку строки по какой-то причине (например, если процесс будет убит через kill -9).

Таким образом, этот вариант подходит для случая высокой конкуренции за ресурсы БД (быстрее освобождаем коннекты, более короткие транзакции, нет эксклюзивной блокировки).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjM5NDU3MzhdfQ==
-->