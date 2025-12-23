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
