## Отчет

Для начала собираем образ patroni:

<img width="1086" height="566" alt="{49C734C4-FE9C-4102-971B-9B21135BD056}" src="https://github.com/user-attachments/assets/b303c05b-6907-46f8-92ad-1c86cd1b3e20" />

Запускаем docker конейнеры:

<img width="1079" height="384" alt="{8D8EB667-7653-4A1A-87FB-E0D905B837DE}" src="https://github.com/user-attachments/assets/04c15782-d504-4ed1-b2f8-1a739e5771ff" />

Кластер состоит из трех узлов, две реплики patroni1 (IP адрес 172.18.0.9) и patroni2 (IP адрес 172.18.0.6) и лидер (мастер) (IP адрес 172.18.0.4) patroni3:

<img width="1081" height="203" alt="{599A71CB-D99F-4D9C-AF85-29217F0E3C82}" src="https://github.com/user-attachments/assets/03ea2bb0-97b9-44da-a852-465bac159505" />

Состояние HAProxy:

<img width="1920" height="729" alt="{BD7D3A68-B5D9-4E41-A1E6-E1D09631781E}" src="https://github.com/user-attachments/assets/0df4dbdd-8442-4b4d-9800-5d87a1be0348" />

Видно, что в разделе primary 2 узла, patroni1 и patroni2, выделены красненьким и имеют статус DOWN, а узел patroni3 выделен зелененьким и имеет статус UP. В разделе replicas узлы patroni1 и patroni2 зеленые со статусом UP, а patroni3 красный со статусом DOWN. Данная картина соответстует тому, что выводилось по команде patronictl list, то есть что узел patroni3 мастер, а остальные два узла - реплики. Соответственно, в postgreSQL можно изменять БД только с узла patroni3, а на узлах patroni1 и patroni2 можно только читать.

В общем, через psql у меня не получилось подключиться к мастер-ноде по порту 5001, поэтому использовала порт 5432:

<img width="884" height="250" alt="{C74BFAC4-4A47-4AD8-920C-71098957C220}" src="https://github.com/user-attachments/assets/d18a4021-a3c9-4ca4-a71f-5f925c8b9b28" />

С репликой аналогично:

<img width="1088" height="285" alt="{27ED72A9-AF94-428C-9344-C33308C3EE13}" src="https://github.com/user-attachments/assets/25bfd2f6-21c9-4e1b-aff4-dd1ffd30c64a" />

Заливаем скрипт в базу на мастере:

<img width="886" height="985" alt="{C435BABC-5AC7-4939-94F7-F894527AB05E}" src="https://github.com/user-attachments/assets/68be9a39-a393-4c13-8331-89484f60835d" />

Проверяем на реплике, что все получилось. Видим 2 таблицы owners и events, значит, все идет по плану.

<img width="665" height="284" alt="{70C703B3-5642-4514-8961-77209DAF7800}" src="https://github.com/user-attachments/assets/0146862d-8ba7-415a-bf09-7989b6042fa1" />

После этого момента был какой-то перерыв в несколько дней, поэтому теперь лидером стал узел patroni2, IP-адреса узлов тоже поменялись:

<img width="947" height="205" alt="{8FBFD472-14B8-472E-A718-239C6679F311}" src="https://github.com/user-attachments/assets/30392654-5a55-4454-921f-a3e3259ca894" />

Новое состояние HAProxy:

<img width="1920" height="718" alt="{260134A4-3091-4F2D-9265-490FAF35D400}" src="https://github.com/user-attachments/assets/64029914-7cca-412b-9d21-2ecd0546d51c" />

Опять заливаем приложенный в дз скрипт в бд на мастере, но уже на patroni2. Далее я запустила скрипт traffic-generator.py:

<img width="1071" height="560" alt="{1697E7D1-3C77-48F7-93D7-50A75967AA10}" src="https://github.com/user-attachments/assets/6a6e78aa-da7d-4919-889b-30918f50a555" />

Судя по логам, сначала идет подключение к PostrgreSQL по порту 5002, соответствующему репликам, но за счет HAProxy происходит перенаправление на мастер, что также видно в логах, в результате чего соединение устанавливается с мастером, на котором успешно происходят и запись (INSERT), и чтение (READ).

Теперь выключила узел-реплику patroni1, скрипт продолжает работать, лидер по-прежнему узел patroni2:

<img width="1919" height="721" alt="{B0A49A8F-3C88-489D-ADDC-AE3394B1F4C2}" src="https://github.com/user-attachments/assets/4ce40352-a991-4f7a-8272-d11fb61445da" />

Включила обратно, никаких изменений в выводимых логах traffic-generator.py нет.

Теперь выключаю узел-лидер patroni2, на время подключение было потеряно, но затем HAProxy перенаправил запросы на нового лидера, и скрипт вновь начал работать как раньше:

<img width="1285" height="379" alt="{98A38078-3304-4697-863D-4262D1226C2E}" src="https://github.com/user-attachments/assets/29e80d12-3686-47c2-84b3-f9564b1f2244" />

Новым лидером стал узел patroni1:

<img width="1918" height="687" alt="{2F44C4E5-8022-4D83-A7FE-DCE54CF2FA19}" src="https://github.com/user-attachments/assets/f35ddae4-e0f8-4ae8-b321-4b17829e594a" />

Если выключить и patroni1, скрипт спустя какое-то время продолжит работать, но лидером уже станет узел patroni3:

<img width="1919" height="678" alt="{B95C3017-A1E4-48C3-BDD1-725F7C45D1B5}" src="https://github.com/user-attachments/assets/d96efe73-cf38-4259-96c1-b9d02772431f" />

Если выключить и patroni3, то уже ничего работать не будет по очевидной причине:

<img width="1264" height="312" alt="{781DA606-519D-4A16-8683-BF2A4436540B}" src="https://github.com/user-attachments/assets/51435f6f-82b1-4c11-9747-3ae11265aa6d" />

<img width="1920" height="593" alt="image" src="https://github.com/user-attachments/assets/71ab7ba3-9326-44e6-bd56-719d5c1ee102" />

Таким образом, можно сделать вывод об отказоустойчивости системы, что при наличии n узлов patroni при падении n - 1 из них все еще будет возможно чтение и запись с БД.

Включаем обратно 3 контейнера patroni, patroni2 вновь лидер, теперь пробую выключить один etcd1, никаких изменений не происходит, в логах скрипта в том числе ничео не меняется:

<img width="1915" height="720" alt="{7367F405-DD52-41F5-A075-17879105E411}" src="https://github.com/user-attachments/assets/14381258-32ae-49d0-9b88-38a68bf729e0" />

А вот выключение двух контейнеров (etcd1 и etcd2) роняет работу скрипта:

<img width="1270" height="978" alt="{997EF3E2-6EC2-4C1E-BC0A-52200C2145C4}" src="https://github.com/user-attachments/assets/d6eb2ae5-ddd6-4ee2-87c5-c06b21c4af35" />

При этом все узлы становятся репликами:

<img width="1920" height="727" alt="{54AD2CB6-8DEA-4E4F-9B39-EE4059A9B8C4}" src="https://github.com/user-attachments/assets/5b011308-0c04-4589-bec8-a8a4d8e309ba" />

Включение одного из контейнеров etcd восстанавливает возможность приложения писать и читать с мастера:

<img width="422" height="265" alt="{BFEC4732-96B4-4993-A373-054B11F90D9F}" src="https://github.com/user-attachments/assets/1d7e3c52-fca2-4ba4-ad5c-0bcdbee16262" />

Это связано с тем, что etcd нужен кворум, поэтому при выключении 1 etcd контейнера он еще может работать, а при выключении большего числа уже нет, поэтому он становится не в состоянии хранить информацию о том, кто, например, является лидером, поэтому мы видим такую картину в HAProxy, где все трое узлов - реплики, а чтение и запись невозможны.

Теперь отключаем контейнер haproxy, сразу все падает, чтение и запись прекращаются, http://localhost:7001/ становится недоступен:

<img width="1049" height="443" alt="{F10F0959-0F52-4C00-8E94-8EEF9E6AEE1B}" src="https://github.com/user-attachments/assets/e30003d3-e736-4051-9aec-b416f235e439" />

Это связано с тем, что приложение теряет точку доступ к кластеру PostgreSQL, так как все подключения в скрипте к БД были через HAProxy.

Таким образом, отказоустойчивость кластера не является достаточной, так как пока хотя бы один узел patroni работает, то будет возможным доступ к БД, но все еще бутылочным горлышком является HAProxy, так как он один и при его отказе теряется целиком доступ к базе. Также при нарушении кворума сисема может рухнуть из-за etcd.

Наибольшей проблемой является единственность HAProxy, решением этого может быть использование нескольких haproxy, чтобы при падении одного сразу использовать другой.

В Grafana ничего не было:

<img width="1920" height="970" alt="{12918648-5B0B-49F7-AA9B-4CBD960AE0BE}" src="https://github.com/user-attachments/assets/2bd016b5-cb72-4926-8950-03ef8f0e8899" />

Поэтому я импортировала туда три дашборда из grafana_dashboards:

<img width="1912" height="871" alt="{20DADBE4-8C82-4C3D-A83E-636E5194D2D2}" src="https://github.com/user-attachments/assets/01e77a21-7016-41ae-80ec-e313b9ff03ac" />

Но по-моему, что-то пошло не так, потому что он ругался:

<img width="1908" height="891" alt="{319276E5-EB5A-44B5-A408-814EF268AC63}" src="https://github.com/user-attachments/assets/79f5e1f0-6575-4af6-8d80-e4629747297e" />
