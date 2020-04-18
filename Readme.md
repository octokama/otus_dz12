# OTUS ДЗ Selinux (Centos 7)
-----------------------------------------------------------------------
### Домашнее задание

Практика с SELinux
Цель: Тренируем умение работать с SELinux: диагностировать проблемы и модифицировать политики SELinux для корректной работы приложений, если это требуется.
1. Запустить nginx на нестандартном порту 3-мя разными способами:
- переключатели setsebool;
- добавление нестандартного порта в имеющийся тип;
- формирование и установка модуля SELinux.
К сдаче:
- README с описанием каждого решения (скриншоты и демонстрация приветствуются).

2. Обеспечить работоспособность приложения при включенном selinux.
- Развернуть приложенный стенд
https://github.com/mbfx/otus-linux-adm/blob/master/selinux_dns_problems/
- Выяснить причину неработоспособности механизма обновления зоны (см. README);
- Предложить решение (или решения) для данной проблемы;
- Выбрать одно из решений для реализации, предварительно обосновав выбор;
- Реализовать выбранное решение и продемонстрировать его работоспособность.
К сдаче:
- README с анализом причины неработоспособности, возможными способами решения и обоснованием выбора одного из них;
- Исправленный стенд или демонстрация работоспособной системы скриншотами и описанием.
Критерии оценки:
Обязательно для выполнения:
- 1 балл: для задания 1 описаны, реализованы и продемонстрированы все 3 способа решения;
- 1 балл: для задания 2 описана причина неработоспособности механизма обновления зоны;
- 1 балл: для задания 2 реализован и продемонстрирован один из способов решения;
Опционально для выполнения:
- 1 балл: для задания 2 предложено более одного способа решения;
- 1 балл: для задания 2 обоснованно(!) выбран один из способов решения. 


#### Вступление

Для того чтобы работать с политиками сразу установим ```semanage```:
```yum -y install policycoreutils-python```

Просмотреть контекст безопасности на файлах, директориях и процессах можем командами ```ps -Z``` и ```ls -Z```.

Узнать режим работы можно командой ```sestatus``` и ```getenforce```

Файлы политик размещаются по пути - ```/etc/selinux/targeted/contexts/files```

Лог аудита храниться в файле ```/var/log/audit/audit.log```

Просмотреть текущие правила - ```sesearch -A -s httpd_t```

Правильно задать контекст ветвей файловой системы необходимо через semanage (прописывается в политику) - ```semanage fcontext -a -t httpd_sys_content_t "/html(/.*)?"```

Восстановить контекст (а также необходимо если надо окончательно применить новые метки файлов в файловой системе, чтобы заработало после перезагрузки) - ```restorecon /html```

Проверка по портам - ```semanage port -l | grep ssh```. Назначить доп. порты в тип правил (ssh) - ```semanage port -a -t ssh_port_t -p -tcp 5022```. Удалить прописанный порт - ```semanage port -d -p tcp 5022```. Порты по-умолчанию описанные в политике удалить нельзя.

#### Как анализировать логи selinux

- audit2why < /var/log/audit/audit.log - парсит файл логов и показывает нам как решить проблему

#### Создание политики на основе лог файла 

- Для создания политики на основе лог файла где содержиться информация о том что было заблокировано, можно использовать утилиту ``` audit2allow```, а именно команду - ```audit2allow -M httpd_add --debug < /var/log/audit/audit.log```. При создании модуля создается файл .te (Type Enforcement) и файл .pp (скомпилированный пакет политики).

- Включение созданного модуля - ```semodule -i httpd_add.pp```. Убедиться что модуль подключен - ```semodule -l | grep httpd_add```

- Удаление модуля - ```semodule -r httpd_add```

Дополнительная информация:

- https://docs.fedoraproject.org/ru-RU/Fedora/13/html/Security-Enhanced_Linux/sect-Security-Enhanced_Linux-Fixing_Problems-Allowing_Access_audit2allow.html

- https://www.server-world.info/en/note?os=CentOS_7&p=selinux&f=9 

- https://max-ko.ru/14-upravlenie-selinux.html


#### Режимы работы:

- Enforcing (Включен, разрешает только то что явно разрешено. Переключиться на этот режим можно командой ```setenforce 1```)

- Permissive (Только журналирование, переключиться на этот режим можно командой ```setenforce 0``` (команда работает до первой перезагрузки и чтобы выставить это значение навсегда надо править файл ```/etc/selinux/config```  и выставить значение в переменной SELINUX на ```SELINUX=Permissive```))

- Disabled (Выключен, для активации/деактивации требуется перезагрузка. Для включения этого режима необходимо править файл конфигурации selinux - ```/etc/selinux/config``` и выставить значение в переменной SELINUX на ```SELINUX=disabled```, после этого перезагрузить компьютер.)



### Решение домашнего задания

1. Запустить nginx на нестандартном порту 3-мя разными способами.

а) переключатели setsebool;

- Для начала установить веб-сервер nginx командой ```yum install -y nginx```

- Заходим в файл конфигурации nginx и меняем порт на 12345 - ```nano /etc/nginx/nginx.conf```. Запустить на таком порту веб-сервер не получается.

- Далее выполняем команду ```audit2why < /var/log/audit/audit.log```, для того чтобы понять какой логический тип включать

- Утилита ```audit2why``` рекомендует выполнить команду ```setsebool -P nis_enabled 1```, ключ -P сохранит правило и после перезагрузки. Выполняем команду без ключа -P.

- После этого веб-сервер nginx успешно запускается

б) добавление нестандартного порта в имеющийся тип;

- выполняем команду для того чтобы посмотреть какие порты могут работать по протоколу http - ```semanage port -l | grep http```. Видим что нашего порта 12345 там нет

- добавляем наш нестандартный порт в правило политики, командой - ```semanage port -a -t http_port_t -p tcp 12345```. Удалить ```semanage port -d -t http_port_t -p tcp 12345```

Листинг:
```
[root@host1 vagrant]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      12345, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```

- Теперь веб-сервер nginx запускается на нашем нестандартном порту

в) формирование и установка модуля SELinux

Для этого нам необходимо будет скомпилировать модуль на основе лог файла аудита, в котором есть информация о запретах.

- выполним команду ```audit2allow -M httpd_add --debug < /var/log/audit/audit.log```:
```
[root@host1 vagrant]# audit2allow -M httpd_add --debug < /var/log/audit/audit.log
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i httpd_add.pp

[root@host1 vagrant]# ls -l 
total 8
-rw-r--r--. 1 root root 964 Apr 13 02:45 httpd_add.pp
-rw-r--r--. 1 root root 261 Apr 13 02:45 httpd_add.te
```
- Далее происталлируем наш созданный модуль - ```semodule -i httpd_add.pp```

- проверяем загрузился ли наш модуль:
```
[root@host1 vagrant]# semodule -l | grep http
httpd_add	1.0
```
- Наш веб-сервер теперь снова работает на нашем порту 12345

- Чтобы удалить модуль, надо выполнить команду ```semodule -r httpd_add```. Чтобы выключить модуль ```semodule -d -v httpd_add```. Включить модуль ```semodule -e -v httpd_add```


2. Обеспечить работоспособность приложения при включенном selinux

- Скачиваем данные из репозитория ```git clone https://github.com/mbfx/otus-linux-adm.git```

- Запускаем наши виртуальные машины командой ```vagrant up```

- Подключаемся к клиентской машине ```vagrant ssh client``` и пробуем выполнить команды:
```
nsupdate -k /etc/named.zonetransfer.key
server 192.168.50.10
zone ddns.lab 
update add www.ddns.lab. 60 A 192.168.50.15
send
```
Получаем ошибку ```update failed: SERVFAIL```

Для того чтобы решить эту задачу необходимо создать некоторое количество модулей по конкретным ошибкам (потому как решаем одну ошибку SELINUX то появляется другая) на DNS сервере, которые описываются в файлах ```/var/log/audit/audit.log``` и ```/var/log/messages```, а также для отлавнивания ошибок я использовал ```systemctl status named```, после перезапуска процесса BIND сервера, там также отображаются ошибки.

Какой алгоритм решил проблему:
1. Нам нужно убрать все исключения и ошибке по линии SELINUX, чтобы система безопасности перестала ругаться
- выполняем команду ```audit2why < /var/log/audit/audit.log``` и видим:
```
[root@ns01 vagrant]# audit2why < /var/log/audit/audit.log
type=AVC msg=audit(1587231618.482:1955): avc:  denied  { search } for  pid=7268 comm="isc-worker0000" name="net" dev="proc" ino=33134 scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:sysctl_net_t:s0 tclass=dir permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.

type=AVC msg=audit(1587231618.482:1956): avc:  denied  { search } for  pid=7268 comm="isc-worker0000" name="net" dev="proc" ino=33134 scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:sysctl_net_t:s0 tclass=dir permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.

```
- Далее выполняем команду ```audit2allow -M named-selinux --debug < /var/log/audit/audit.log``` и ```semodule -i named-selinux.pp```
- Это не решает проблему, у нас начинают появляется новые ошибки:
```
[root@ns01 vagrant]# cat /var/log/messages | grep ausearch
Apr 18 17:40:20 localhost python: SELinux is preventing /usr/sbin/named from search access on the directory net.#012#012*****  Plugin catchall (100. confidence) suggests   **************************#012#012If you believe that named should be allowed search access on the net directory by default.#012Then you should report this as a bug.#012You can generate a local policy module to allow this access.#012Do#012allow this access for now by executing:#012# ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000#012# semodule -i my-iscworker0000.pp#012
```
Здесь нам подсказывают что надо сделать чтобы SELinux перестал блокировать доступ к доступ то нужно выполнить команду: ```ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000 | semodule -i my-iscworker0000.pp```

- Так как это тоже не решает проблему смотрим что надо дальше пишет лог ```/var/log/messages```:
```
Apr 18 17:45:18 localhost python: SELinux is preventing /usr/sbin/named from read access on the file ip_local_port_range.#012#012*****  Plugin catchall (100. confidence) suggests   **************************#012#012If you believe that named should be allowed read access on the ip_local_port_range file by default.#012Then you should report this as a bug.#012You can generate a local policy module to allow this access.#012Do#012allow this access for now by executing:#012# ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000#012# semodule -i my-iscworker0000.pp#012
```
Здесь у файла DNS сервера нет доступа прочитать файл ```ip_local_port_range```
Здесь нам подсказывают что надо сделать чтобы SELinux перестал блокировать доступ к доступ то нужно выполнить команду: ```ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0001 | semodule -i my-iscworker0001.pp```

- Это не помогает идем дальше:
```
Apr 18 17:46:51 localhost python: SELinux is preventing /usr/sbin/named from open access on the file /proc/sys/net/ipv4/ip_local_port_range.#012#012*****  Plugin catchall (100. confidence) suggests   **************************#012#012If you believe that named should be allowed open access on the ip_local_port_range file by default.#012Then you should report this as a bug.#012You can generate a local policy module to allow this access.#012Do#012allow this access for now by executing:#012# ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000#012# semodule -i my-iscworker0000.pp#012
```
Здесь нам подсказывают что надо сделать чтобы SELinux перестал блокировать доступ к доступ то нужно выполнить команду: ```ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0002 | semodule -i my-iscworker0002.pp```

- Это нам не помогает идем дальше:
```
Apr 18 17:49:29 localhost python: SELinux is preventing /usr/sbin/named from getattr access on the file /proc/sys/net/ipv4/ip_local_port_range.#012#012*****  Plugin catchall (100. confidence) suggests   **************************#012#012If you believe that named should be allowed getattr access on the ip_local_port_range file by default.#012Then you should report this as a bug.#012You can generate a local policy module to allow this access.#012Do#012allow this access for now by executing:#012# ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000#012# semodule -i my-iscworker0000.pp#012
```
Здесь нам подсказывают что надо сделать чтобы SELinux перестал блокировать доступ к доступ то нужно выполнить команду: ```ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0003 | semodule -i my-iscworker0003.pp```

- И это нам не помогает, идем дальше:
```
Apr 18 17:54:02 localhost python: SELinux is preventing isc-worker0000 from create access on the file named.ddns.lab.view1.jnl.#012#012*****  Plugin catchall_labels (83.8 confidence) suggests   *******************#012#012If you want to allow isc-worker0000 to have create access on the named.ddns.lab.view1.jnl file#012Then you need to change the label on named.ddns.lab.view1.jnl#012Do#012# semanage fcontext -a -t FILE_TYPE 'named.ddns.lab.view1.jnl'#012where FILE_TYPE is one of the following: dnssec_trigger_var_run_t, ipa_var_lib_t, krb5_host_rcache_t, krb5_keytab_t, named_cache_t, named_log_t, named_tmp_t, named_var_run_t, named_zone_t.#012Then execute:#012restorecon -v 'named.ddns.lab.view1.jnl'#012#012#012*****  Plugin catchall (17.1 confidence) suggests   **************************#012#012If you believe that isc-worker0000 should be allowed create access on the named.ddns.lab.view1.jnl file by default.#012Then you should report this as a bug.#012You can generate a local policy module to allow this access.#012Do#012allow this access for now by executing:#012# ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000#012# semodule -i my-iscworker0000.pp#012
```
Здесь нам подсказывают что надо сделать чтобы SELinux перестал блокировать доступ к доступ то нужно выполнить команду: ```ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0004 | semodule -i my-iscworker0004.pp```

- После чего в файле ```/var/log/messages``` перестают появляться, но ошибки есть в выводе команды ```systemctl status named```:
```
[root@ns01 vagrant]# systemctl status named
● named.service - Berkeley Internet Name Domain (DNS)
   Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; vendor preset: disabled)
   Active: active (running) since Sat 2020-04-18 18:02:46 UTC; 4s ago
  Process: 8015 ExecStop=/bin/sh -c /usr/sbin/rndc stop > /dev/null 2>&1 || /bin/kill -TERM $MAINPID (code=exited, status=0/SUCCESS)
  Process: 8028 ExecStart=/usr/sbin/named -u named -c ${NAMEDCONF} $OPTIONS (code=exited, status=0/SUCCESS)
  Process: 8026 ExecStartPre=/bin/bash -c if [ ! "$DISABLE_ZONE_CHECKING" == "yes" ]; then /usr/sbin/named-checkconf -z "$NAMEDCONF"; else echo "Checking of zone files is disabled"; fi (code=exited, status=0/SUCCESS)
 Main PID: 8031 (named)
   CGroup: /system.slice/named.service
           └─8031 /usr/sbin/named -u named -c /etc/named.conf

Apr 18 18:02:46 ns01 named[8031]: automatic empty zone: view default: HOME.ARPA
Apr 18 18:02:46 ns01 named[8031]: none:104: 'max-cache-size 90%' - setting to 211MB (out of 235MB)
Apr 18 18:02:46 ns01 named[8031]: command channel listening on 192.168.50.10#953
Apr 18 18:02:46 ns01 named[8031]: managed-keys-zone/view1: journal file is out of date: removing journal file
Apr 18 18:02:46 ns01 named[8031]: managed-keys-zone/view1: loaded serial 10
Apr 18 18:02:46 ns01 named[8031]: managed-keys-zone/default: journal file is out of date: removing journal file
Apr 18 18:02:46 ns01 named[8031]: managed-keys-zone/default: loaded serial 10
Apr 18 18:02:46 ns01 named[8031]: zone 0.in-addr.arpa/IN/view1: loaded serial 0
Apr 18 18:02:46 ns01 named[8031]: zone ddns.lab/IN/view1: journal rollforward failed: no more
Apr 18 18:02:46 ns01 named[8031]: zone ddns.lab/IN/view1: not loaded due to errors.
```
2. Удаляем файл ```/etc/named/dynamic/named.ddns.lab.view1.jnl```, перезапускаем сервис DNS сервера ```systemctl restart named``` и больше не видим ошибок, теперь динамическое обновление выполняется успешно.
```
[root@ns01 vagrant]# systemctl status named
● named.service - Berkeley Internet Name Domain (DNS)
   Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; vendor preset: disabled)
   Active: active (running) since Sat 2020-04-18 18:03:44 UTC; 3min 38s ago
  Process: 8057 ExecStop=/bin/sh -c /usr/sbin/rndc stop > /dev/null 2>&1 || /bin/kill -TERM $MAINPID (code=exited, status=0/SUCCESS)
  Process: 8070 ExecStart=/usr/sbin/named -u named -c ${NAMEDCONF} $OPTIONS (code=exited, status=0/SUCCESS)
  Process: 8068 ExecStartPre=/bin/bash -c if [ ! "$DISABLE_ZONE_CHECKING" == "yes" ]; then /usr/sbin/named-checkconf -z "$NAMEDCONF"; else echo "Checking of zone files is disabled"; fi (code=exited, status=0/SUCCESS)
 Main PID: 8072 (named)
   CGroup: /system.slice/named.service
           └─8072 /usr/sbin/named -u named -c /etc/named.conf

Apr 18 18:03:44 ns01 named[8072]: automatic empty zone: view default: 8.B.D.0.1.0.0.2.IP6.ARPA
Apr 18 18:03:44 ns01 named[8072]: automatic empty zone: view default: EMPTY.AS112.ARPA
Apr 18 18:03:44 ns01 named[8072]: automatic empty zone: view default: HOME.ARPA
Apr 18 18:03:44 ns01 named[8072]: none:104: 'max-cache-size 90%' - setting to 211MB (out of 235MB)
Apr 18 18:03:44 ns01 named[8072]: command channel listening on 192.168.50.10#953
Apr 18 18:03:44 ns01 named[8072]: managed-keys-zone/view1: journal file is out of date: removing journal file
Apr 18 18:03:44 ns01 named[8072]: managed-keys-zone/view1: loaded serial 11
Apr 18 18:04:30 ns01 named[8072]: client @0x7f61fc09df00 192.168.50.15#26071/key zonetransfer.key: view view1: signer "zonetransfer.key" approved
Apr 18 18:04:30 ns01 named[8072]: client @0x7f61fc09df00 192.168.50.15#26071/key zonetransfer.key: view view1: updating zone 'ddns.lab/IN': adding an RR at 'www.ddns.lab' A 192.168.50.15
Apr 18 18:04:36 ns01 named[8072]: client @0x7f61fc09df00 192.168.50.15#26071/key zonetransfer.key: view view1: signer "zonetransfer.key" approved
```

Причина неработоспособности механизма обновления заключается в том что Selinux блокировал доступ к обновлению файлов динамического обновления для DNS сервера, а также к некоторым файлам ОС, к которым DNS сервер (```/usr/sbin/named```) обращается во время своей работы (указаны выше, взяты из логов сервера). Кроме того рекомендуется удалить файл, куда записываются динамические обновления зоны, так как прежде чем данные попадают в этот файл, они сначала записываются во временный файл tmp, для которого может срабатывать блокировка (как в моем случае).
Временные файлы можно найти:
```
[root@ns01 vagrant]# ls -l /etc/named/dynamic/
total 32
-rw-rw-rw-. 1 named named 509 Apr 18 17:40 named.ddns.lab
-rw-rw-rw-. 1 named named 509 Apr 18 17:40 named.ddns.lab.view1
-rw-r--r--. 1 named named 700 Apr 18 18:04 named.ddns.lab.view1.jnl
-rw-r--r--. 1 named named 348 Apr 18 18:30 tmp-6OGP6YASy1
-rw-r--r--. 1 named named 348 Apr 18 18:44 tmp-HUsH1RRHBF
-rw-r--r--. 1 named named 348 Apr 18 18:17 tmp-OEmMkfw6J6
-rw-r--r--. 1 named named 348 Apr 18 19:09 tmp-R8cPmFCasl
-rw-r--r--. 1 named named 348 Apr 18 18:57 tmp-csgM4QDJR7
```

Считаю что можно использовать или компиляцию модулей или изменения контекста безопасности для файлов, оба эти способа специально предназначены разработчиками Selinux для того чтобы решать подобные проблемы.

Кроме данных способов существуют также способы:
- Выключить Selinux совсем (не рекомендуется)
- изменить контекст тех файлов, к которым DNS серверу затруднен доступ командам ```semanage fcontext -a -t FILE_TYPE named.ddns.lab.view1.jnl```, где назначить файлам один из следующих типов контекста безопасности ```dnssec_trigger_var_run_t, ipa_var_lib_t, krb5_host_rcache_t, krb5_keytab_t, named_cache_t, named_log_t, named_tmp_t, named_var_run_t, named_zone_t```, и затем выполнить запись контекста в ядро ```restorecon -v named.ddns.lab.view1.jnl```. В данном случае приведены примеры для файла ```named.ddns.lab.view1.jnl```, в данный файл DNS сервер записывает динамические обновления от DNS клиентов. 


