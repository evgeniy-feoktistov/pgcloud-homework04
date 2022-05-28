# pgcloud-homework04
## Домашнее задание
**Тюнинг Постгреса**

### Описание/Пошаговая инструкция выполнения домашнего задания:

Имеем ВМ
```
OS: Ubuntu 20.04.4 LTS x86_64
Host: Virtual Machine Hyper-V UEFI Release v4.1
Kernel: 5.4.0-113-generic
CPU: AMD Ryzen 7 PRO 3700 (4) @ 3.703GHz
Memory: 756MiB / 1917MiB

sdb                             8:16   0   10G  0 disk
└─vg_tmp_pgdata-lv_tmp_pgdata 253:1    0   10G  0 lvm  /var/lib/postgresql

psql (PostgreSQL) 14.3 (Ubuntu 14.3-1.pgdg20.04+1)
```
Протестируем PostgreSQL с дефолтным конфигом
```
postgres=# create database test;
CREATE DATABASE

# sudo -u postgres pgbench -i test
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.05 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.18 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.10 s, vacuum 0.04 s, primary keys 0.03 s).

# sudo -u postgres pgbench -c 50 -j 2 -P 10 -T 60 test
pgbench (14.3 (Ubuntu 14.3-1.pgdg20.04+1))
starting vacuum...end.
progress: 10.0 s, 582.2 tps, lat 82.362 ms stddev 202.987
progress: 20.0 s, 582.5 tps, lat 85.191 ms stddev 242.094
progress: 30.0 s, 587.2 tps, lat 85.307 ms stddev 229.346
progress: 40.0 s, 581.7 tps, lat 87.203 ms stddev 208.804
progress: 50.0 s, 585.0 tps, lat 84.432 ms stddev 183.888
progress: 60.0 s, 572.3 tps, lat 88.403 ms stddev 239.114
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
duration: 60 s
number of transactions actually processed: 34959
latency average = 85.816 ms
latency stddev = 218.953 ms
initial connection time = 43.051 ms
tps = 582.170001 (without initial connection time)
```
**582 tps** в среднем без какого либо вмешательства в настройки.

Выставить "оптимальные" настройки: воспользуемся для этого утилитой pgtune
```
# mkdir /etc/postgresql/14/main/conf.d
# chown postgres: /etc/postgresql/14/main/conf.d
# nano /etc/postgresql/14/main/conf.d/01-pgtune.conf
# cat /etc/postgresql/14/main/conf.d/01-pgtune.conf
# DB Version: 14
# OS Type: linux
# DB Type: mixed
# Total Memory (RAM): 2 GB
# CPUs num: 4
# Connections num: 50
# Data Storage: ssd

max_connections = 50
shared_buffers = 512MB
effective_cache_size = 1536MB
maintenance_work_mem = 128MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 2621kB
min_wal_size = 1GB
max_wal_size = 4GB
max_worker_processes = 4
max_parallel_workers_per_gather = 2
max_parallel_workers = 4
max_parallel_maintenance_workers = 2
```
И звапустим наш тест еще раз
```
# sudo -u postgres pgbench -c 50 -j 2 -P 10 -T 60 test
pgbench (14.3 (Ubuntu 14.3-1.pgdg20.04+1))
starting vacuum...end.
progress: 10.0 s, 580.5 tps, lat 84.136 ms stddev 180.793
progress: 20.0 s, 587.4 tps, lat 83.997 ms stddev 183.738
progress: 30.0 s, 580.6 tps, lat 86.844 ms stddev 181.813
progress: 40.0 s, 577.0 tps, lat 85.192 ms stddev 233.928
progress: 50.0 s, 574.8 tps, lat 86.865 ms stddev 245.547
progress: 60.0 s, 581.3 tps, lat 88.399 ms stddev 209.630
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
duration: 60 s
number of transactions actually processed: 34866
latency average = 86.051 ms
latency stddev = 207.381 ms
initial connection time = 40.182 ms
tps = 580.577925 (without initial connection time)
```
Как и ожидалась в случае синтетического теста результат примерно такой же.
З.Ы.
Как показывает atop узкое место диск.
```
LVM | v_tmp_pgdata  | busy    102%  | read       0  | write  11646  | KiB/w      4  | MBr/s    0.0  | MBw/s    4.8  | avio 0.86 ms  |
DSK |          sdb  | busy    102%  | read       0  | write  11642  | KiB/w      4  | MBr/s    0.0  | MBw/s    4.8  | avio 0.86 ms  |
```
Теперь подкрутим некоторые параметры.
Текущие значение **wal_level**
```
postgres=# show wal_level ;
 wal_level
-----------
 replica
(1 row)
```
Поставим **minimal** и проверим. max_wal_senders - ставим 0, иначе СУБД не запуститься с минимальным журналом:
```
# cat /etc/postgresql/14/main/conf.d/02-jf.conf
max_wal_senders = 0
wal_level = minimal
```
Перезапускаем PostgreSQL и запускаем тест:
```
# systemctl restart postgresql@14-main.service
# sudo -u postgres psql -c 'show wal_level;'
 wal_level
-----------
 minimal
(1 row)
# sudo -u postgres pgbench -c 50 -j 2 -P 10 -T 60 test
pgbench (14.3 (Ubuntu 14.3-1.pgdg20.04+1))
starting vacuum...end.
progress: 10.0 s, 581.6 tps, lat 82.624 ms stddev 189.727
progress: 20.0 s, 568.4 tps, lat 88.193 ms stddev 254.700
progress: 30.0 s, 582.6 tps, lat 85.540 ms stddev 199.538
progress: 40.0 s, 579.5 tps, lat 87.723 ms stddev 218.490
progress: 50.0 s, 581.4 tps, lat 85.609 ms stddev 211.781
progress: 60.0 s, 572.8 tps, lat 84.974 ms stddev 210.774
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
duration: 60 s
number of transactions actually processed: 34713
latency average = 86.443 ms
latency stddev = 216.627 ms
initial connection time = 39.827 ms
tps = 577.871771 (without initial connection time)
```
**577 tps** ,ну, с погрешностью, но данный параметр не влияет. Жаль. Надо будет сравнить физически на сколько меньше данных попадает в WAL файл при таком значение.

Теперь поочередно отключим параметры fsync и synchronous_commit.
В начале fsync
```
# cat /etc/postgresql/14/main/conf.d/02-jf.conf
max_wal_senders = 0
wal_level = minimal
fsync = off
synchronous_commit = on #by default

# sudo -u postgres pgbench -c 50 -j 2 -P 10 -T 60 test
pgbench (14.3 (Ubuntu 14.3-1.pgdg20.04+1))
starting vacuum...end.
progress: 10.0 s, 3493.9 tps, lat 14.232 ms stddev 15.046
progress: 20.0 s, 3511.4 tps, lat 14.237 ms stddev 15.037
progress: 30.0 s, 3507.4 tps, lat 14.250 ms stddev 15.170
progress: 40.0 s, 3556.0 tps, lat 14.061 ms stddev 15.249
progress: 50.0 s, 3526.3 tps, lat 14.183 ms stddev 15.314
progress: 60.0 s, 3550.0 tps, lat 14.066 ms stddev 15.011
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
duration: 60 s
number of transactions actually processed: 211500
latency average = 14.178 ms
latency stddev = 15.149 ms
initial connection time = 39.729 ms
tps = 3525.115859 (without initial connection time
```
**tps = 3525.115859** Пабам!  Один из самых опавсных параметров для отключения, но выйгрыш в скорости огромный. (риск повреждения данных при отказе)
Теперь проверим другой вариант, менее опасный - отключим synchronous_commit но fsync включим.
Как пишут в документации: "Во многих случаях отключение synchronous_commit для некритичных транзакций может дать больший выигрыш в скорости, чем отключение fsync, при этом не добавляя риски повреждения данных." Проверяем:
```
# nano /etc/postgresql/14/main/conf.d/02-jf.conf
root@ubuntu02:/home/ujack# cat /etc/postgresql/14/main/conf.d/02-jf.conf
max_wal_senders = 0
wal_level = minimal
fsync = on
synchronous_commit = off

root@ubuntu02:/home/ujack# systemctl restart postgresql@14-main.service
root@ubuntu02:/home/ujack# sudo -u postgres psql -c 'show fsync;'
 fsync
-------
 on
(1 row)

root@ubuntu02:/home/ujack# sudo -u postgres psql -c 'show synchronous_commit;'
 synchronous_commit
--------------------
 off
(1 row)

# sudo -u postgres pgbench -c 50 -j 2 -P 10 -T 60 test
pgbench (14.3 (Ubuntu 14.3-1.pgdg20.04+1))
starting vacuum...end.
progress: 10.0 s, 3423.2 tps, lat 14.519 ms stddev 14.960
progress: 20.0 s, 3433.2 tps, lat 14.566 ms stddev 15.146
progress: 30.0 s, 3422.3 tps, lat 14.610 ms stddev 15.190
progress: 40.0 s, 3466.3 tps, lat 14.420 ms stddev 15.045
progress: 50.0 s, 3467.4 tps, lat 14.412 ms stddev 15.524
progress: 60.0 s, 3453.6 tps, lat 14.480 ms stddev 15.390
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
duration: 60 s
number of transactions actually processed: 206709
latency average = 14.507 ms
latency stddev = 15.216 ms
initial connection time = 40.481 ms
tps = 3445.065194 (without initial connection time)

```
**tps = 3445.065194** Бум! "Чуть-Чуть" меньше результат, чем с fsyn=off. 3445 против 3525 от fsyn. Считаю очень хорошим результатом, и при этом при сбое мы не повредим данные, но учитываем нюанс, что можем потерять последние транзакции перед сбоем. Т.е. подтверждаем, что написано в документации:

*"В режиме off ожидание отсутствует, поэтому может образоваться окно от момента, когда клиент узнаёт об успешном завершении, до момента, когда транзакция действительно гарантированно защищена от сбоя. (Максимальный размер окна равен тройному значению wal_writer_delay.) В отличие от fsync, значение off этого параметра не угрожает целостности данных: сбой операционной системы или базы данных может привести к потере последних транзакций, считавшихся зафиксированными, но состояние базы данных будет точно таким же, как и в случае штатного прерывания этих транзакций. Поэтому выключение режима synchronous_commit может быть полезной альтернативой отключению fsync, когда производительность важнее, чем надёжная гарантия сохранности каждой транзакции."*

Проверим влияние параметра **full_page_writes** (предыдущие параметры вернем пока в дефолт)
```
# cat /etc/postgresql/14/main/conf.d/02-jf.conf
max_wal_senders = 0
wal_level = minimal
fsync = on
synchronous_commit = on
full_page_writes = off

# systemctl restart postgresql@14-main.service

# sudo -u postgres psql -c 'show full_page_writes;'
 full_page_writes
------------------
 off
(1 row)

# sudo -u postgres pgbench -c 50 -j 2 -P 10 -T 60 test
pgbench (14.3 (Ubuntu 14.3-1.pgdg20.04+1))
starting vacuum...end.
progress: 10.0 s, 584.2 tps, lat 81.228 ms stddev 202.219
progress: 20.0 s, 581.6 tps, lat 87.323 ms stddev 256.750
progress: 30.0 s, 576.6 tps, lat 85.728 ms stddev 192.322
progress: 40.0 s, 577.6 tps, lat 87.988 ms stddev 229.789
progress: 50.0 s, 581.1 tps, lat 86.330 ms stddev 209.374
progress: 60.0 s, 582.7 tps, lat 86.295 ms stddev 214.270
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
duration: 60 s
number of transactions actually processed: 34887
latency average = 85.991 ms
latency stddev = 218.476 ms
initial connection time = 39.721 ms
tps = 581.043366 (without initial connection time)
```
**tps = 581.043366** Отключение только этого парамета, в моем случае, не дает результата. По **atop** система опять уперлась в диск при 20% загрузке CPU.
А если учесть факт из документации:

*"Отключение этого параметра ускоряет обычные операции, но может привести к неисправимому повреждению или незаметной порче данных после сбоя системы. Так как при этом возникают практически те же риски, что и при отключении fsync, хотя и в меньшей степени, отключать его следует только при тех же обстоятельствах, которые перечислялись в рекомендациях для вышеописанного параметра."*

то отключать его, наверное, стоит только в строго определенных случаях и только если это даст эффект.

Других параметров, которые могли бы влиять явно не вдаваясь в специфику запросов и системы сейчас не вспомню.

Поэтому сделаем последний тест при отключенных всех параметров:

```
# cat /etc/postgresql/14/main/conf.d/02-jf.conf
max_wal_senders = 0
wal_level = minimal
fsync = off
synchronous_commit = off
full_page_writes = off

# systemctl restart postgresql@14-main.service
# sudo -u postgres pgbench -c 50 -j 2 -P 10 -T 60 test
pgbench (14.3 (Ubuntu 14.3-1.pgdg20.04+1))
starting vacuum...end.
progress: 10.0 s, 3388.3 tps, lat 14.677 ms stddev 15.271
progress: 20.0 s, 3406.7 tps, lat 14.668 ms stddev 15.528
progress: 30.0 s, 3446.9 tps, lat 14.507 ms stddev 15.222
progress: 40.0 s, 3453.0 tps, lat 14.476 ms stddev 14.961
progress: 50.0 s, 3432.4 tps, lat 14.567 ms stddev 15.067
progress: 60.0 s, 3438.2 tps, lat 14.543 ms stddev 15.339
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
duration: 60 s
number of transactions actually processed: 205704
latency average = 14.577 ms
latency stddev = 15.234 ms
initial connection time = 41.038 ms
tps = 3428.768936 (without initial connection time)
```

Результат в моем случае почти ожидаемый. Для систем, где потеря данных не критична (n-я реплика для аналитиков) воплне можно воспользоваться отключением synchronous_commit. Отключать fsync возможно смысла вообще не имеет, (если получаются аналогичные тесты на рассматриваемой системе) или на совсем DEV данных, где их можно поднять быстро из бэкапа/дампа.

Done.
