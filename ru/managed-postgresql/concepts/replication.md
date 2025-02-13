# Репликация

В кластерах {{ mpg-name }} используется _кворумная репликация_ (quorum-based synchronous replication):

1. Среди хостов кластера выбирается мастер, а все остальные хосты становятся репликами.
1. Из реплик случайным образом формируется _кворум_. Транзакция считается успешной только в том случае, если ее подтвердили хост-мастер и все входящие в кворум (_кворумные_) реплики.

Реплики, для которых вручную задан источник репликации, не могут становиться мастером или участвовать в кворуме.

Для формирования кворума необходимо участие не менее половины реплик кластера. Если количество реплик нечетное, значение округляется вниз. Например, в кластере с 17 репликами для формирования кворума необходимы 8.

Кворум формируется заново при изменении топологии кластера: [добавлении](../operations/hosts.md#add) и [удалении](../operations/hosts.md#remove) хостов, выходе их из строя, выводе на техническое обслуживание, возврате в строй и т. д. При этом добавленный в кластер хост сначала синхронизируется с мастером и только потом может участвовать в кворуме.

{% include [non-replicating-hosts](../../_includes/mdb/non-replicating-hosts.md) %}

Подробнее о том, как организована репликация в {{ PG }}, читайте в [документации СУБД](https://www.postgresql.org/docs/current/static/warm-standby.html).

## Управление репликацией {#replication}

Для обеспечения сохранности данных в кластере используется потоковая репликация. Каждый хост-реплика получает поток репликации от другого хоста (обычно это хост-мастер). {{ mpg-name }} управляет потоками репликации в кластере автоматически, но при необходимости ими можно [управлять вручную](../operations/hosts.md#update).

В кластере допустимо комбинировать автоматическое и ручное управление потоками репликации.

### Автоматическое управление потоками репликации {#replication-auto}

После создания кластера {{ PG }} из нескольких хостов в нем находятся один хост-мастер и реплики. Реплики используют хост-мастер в качестве источника репликации.

Особенности автоматической репликации в {{ mpg-name }}:

* При выходе из строя хоста-мастера одна из реплик становится новым мастером.
* При смене мастера источник репликации для всех хостов-реплик автоматически переключается на новый хост-мастер.

Подробнее о процессе выбора мастера см. в подразделе [{#T}](#selecting-the-master).

### Ручное управление потоками репликации {#replication-manual}

При ручном управлении в качестве источника репликации для реплики могут выступать другие хосты в кластере.

При этом вы можете:

- Полностью управлять процессом репликации в кластере и не использовать автоматическую репликацию.
- Настроить репликацию для кластеров {{ PG }} со сложной топологией. При этом часть реплик будет управляться автоматически, а часть — вручную.

Например, таким образом можно настроить каскадную репликацию, когда часть реплик кластера использует другие хосты кластера в качестве источника потока репликации. При этом управление потоком репликации для таких хостов-источников может осуществляться как автоматически средствами {{ mpg-name }}, так и вручную.

Реплики, для которых вручную установлен источник репликации, не могут:

- Становиться мастером при автоматической или ручной смене хоста-мастера (вне зависимости от значения [приоритета](#selecting-the-master)).
- Автоматически переключаться на новый источник репликации при выходе из строя текущего источника репликации.
- Участвовать в кворумной репликации.

## Выбор мастера {#selecting-the-master}

Чтобы повлиять на процесс выбора мастера в кластере {{ PG }}, [установите нужные значения приоритета](../operations/hosts.md#update) для хостов кластера: либо мастером будет выбран хост с самым высоким заданным приоритетом, либо, если в кластере есть несколько реплик с одинаково высоким приоритетом, будут проведены выборы среди этих реплик.

Задать приоритет для хоста кластера можно:

- при [создании кластера](../operations/cluster-create.md) через CLI;
- при [изменении настроек](../operations/hosts.md#update) хоста в кластере.

Наименьший приоритет — `0` (по умолчанию), наивысший — `100`.

## Синхронность записи и консистентность чтения {#write-sync-and-read-consistency}

По умолчанию синхронность реплики и мастера обеспечивается синхронностью передачи WAL — [журнала опережающей записи](https://www.postgresql.org/docs/current/wal-intro.html) (выставлен [параметр](settings-list.md#setting-synchronous-commit) `synchronous_commit = on`). Между моментом доставки WAL и моментом его применения на кворумной реплике проходит некоторое время, в течение которого на реплике можно наблюдать данные, уже устаревшие на мастере.

Если вы хотите гарантировать постоянную консистентность чтения данных между мастером и кворумной репликой, [укажите в настройках кластера](../operations/update.md#change-postgresql-config) параметр `synchronous_commit = remote_apply`. С этим значением параметра запись не считается успешной, пока кворумная реплика не применит изменения из доставленного WAL. При таком уровне синхронности операция чтения, выполненная на мастере и на кворумной реплике после подтверждения транзакции, вернет один и тот же результат.

Недостаток этого решения — операции записи в кластер будут занимать больше времени: если мастер и кворумная реплика расположены в разных зонах доступности, то задержка подтверждения транзакции (latency) будет не меньше, чем время передачи данных туда и обратно (Round-Trip Time, RTT) между дата-центрами, которые размещены в этих зонах доступности. Это приводит к тому, что при записи в один поток и включенном режиме `AUTOCOMMIT` производительность такого кластера может существенно снижаться. Для достижения максимальной производительности рекомендуем, где это возможно, производить запись в несколько потоков, а также [отключить AUTOCOMMIT](https://www.postgresql.org/docs/current/ecpg-sql-set-autocommit.html) и группировать запросы в транзакции.
