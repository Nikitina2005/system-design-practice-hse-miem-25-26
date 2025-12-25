## Отчет

### Проверка приложения

После docker-compose up -d --build делаем docker ps:

<img width="1871" height="263" alt="{BCD8885B-DD7F-4E81-81C9-4843C1D5F14D}" src="https://github.com/user-attachments/assets/800fb6b2-fedd-4349-8eef-128dc6f7009f" />

Видим, что в колонке STATUS у всех контейнеров есть состояние "Up".

Проверяем docker logs demo-app-1_nginx_1:

<img width="1897" height="316" alt="{EC212ACC-4E28-440A-98A2-17B6BADADE5B}" src="https://github.com/user-attachments/assets/f141265e-b02a-46d7-b8c9-acc70a0d9765" />

### Проверка баз данных

<img width="990" height="286" alt="{08923A48-569E-4A36-A4CF-3FD64A89D476}" src="https://github.com/user-attachments/assets/ff2e140b-4368-4428-8a8a-cc532409c9e2" />

Видим таблицу users, в ней видим мои записи:

<img width="718" height="172" alt="{52DE7068-9BD8-4E71-91FC-3F05A6CE01F5}" src="https://github.com/user-attachments/assets/ec695433-8d44-40dc-872a-14253f9cdc41" />

Дашборды пришлось импортировать, например, Node Exporter Full:

<img width="1914" height="885" alt="{984F4B00-3C8E-4D32-8837-73EED1E5CDBC}" src="https://github.com/user-attachments/assets/910aa411-0bb3-4ad4-9273-4fcd5784cb64" />

### Этап 1. Метрики

Для данной системы важно отслеживать следующие метрики:

а) RPS (http_reqs) - requests per second, нужна, чтобы понимать, какую нагрузку реально получает система. **ВАЖНАЯ**

б) Количество виртуальных пользователей (vus) - представляет собой текущее число активных виртуальных пользователей в k6, также важен vus_max - сконфигурированный максимум виртуальных пользователей, чтобы понять, является ли vus бутылочным горлышком или нет.

в) latency (http_req_duration, p90/p95/p99) - полное время жизни запроса, очень важная метрика, считает, меньше какого времени выполняются 90/95/99 процентов запросов из всех. **ВАЖНАЯ**

г) failed запросы HTTP (http_req_failed) - отражает, все ли запросы отработали корректно. Еще будет полезно посмотреть зависимость этой метрики от времени, чтобы можно было сопоставить ошибки с пиками нагрузки. **ВАЖНАЯ**

д) iteration_duration - время выполнения одной итерации скрипта, важно убедиться, что задержки вызваны системой, а не логикой скрипта.

е) data_received и data_sent - объем полученных и отправленных данных за время работы скрипта.

ё) CPU usage

ж) System Load

з) Memory usage

Метрики (ё, ж, з) важны, чтобы понимать, является ли бутылочным горлышком системы недостаток ресурсов процессора.

и) количество подключений к БД - нужно, чтобы выявить проблемы с connection pool, рост до лимита приводит к отказам и росту latency.

й) cache hit ratio - доля запросов, обслуженных из памяти, без обращения к диску, нужна, чтобы понять, насколько эффективно используется RAM.

### Этап 2. Разные сценарии

1) Тот, что был изначально в load-script.js

Это смешанный пользовательский сценарий, где в 80% запросах создаются заказы (POST), в 20% - просматриваются заказы (GET).

Результаты:

<img width="1824" height="247" alt="image" src="https://github.com/user-attachments/assets/89138cdc-795b-460f-9559-f51289ce384a" />

<img width="884" height="324" alt="{58BD60C4-5838-4E9E-B6D4-CB5F317E658C}" src="https://github.com/user-attachments/assets/42622195-14aa-4f0b-8a0b-847fbe4f4897" />

<img width="1051" height="552" alt="{B78126EF-7B77-4ECC-9A36-8E65D0165AA6}" src="https://github.com/user-attachments/assets/1a1d95d8-ef70-4d34-85b4-686dae8abc92" />

2) Шторм

Сам сценарий с использованием k6:

```
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  stages: [
    { duration: '10s', target: 1000 }
  ],
};

const GET_BASE = __ENV.GET_BASE || 'http://host.docker.internal:8080';
const POST_BASE = __ENV.POST_BASE || 'http://host.docker.internal:8081';

function randomInt(min, max){
  return Math.floor(Math.random() * (max - min + 1)) + min;
}

export default function() {
  if (Math.random() < 0.8) {
    const payload = JSON.stringify({ user_id: randomInt(1,2), amount: Math.random()*100, description: 'k6 load' });
    const res = http.post(`${POST_BASE}/api/orders`, payload, { headers: { 'Content-Type': 'application/json' } });
    check(res, { 'created': (r) => r.status === 200 || r.status === 201 });
  } else {
    const res = http.get(`${GET_BASE}/api/orders`);
    check(res, { 'list ok': (r) => r.status === 200 });
  }
  sleep(0.01);
}
```

Результаты:

<img width="1811" height="254" alt="image" src="https://github.com/user-attachments/assets/beee982d-7cb7-42d2-821f-5fc3c0837b90" />

<img width="880" height="311" alt="{DC41A686-FF57-46E0-82F7-9F6C007B6DFD}" src="https://github.com/user-attachments/assets/689ec5f8-0c0e-48e5-8fea-950391bdc212" />

На диаграмме подключений к БД нужен самый правый пик.

<img width="959" height="543" alt="{35E0917C-BFB4-48D2-BB6F-73ADE6D4A6F2}" src="https://github.com/user-attachments/assets/020ef34f-71b5-4aa8-a662-59994df74417" />

3) Волна

Сам сценарий с использованием k6:

```
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  stages: [
    { duration: '2m', target: 500 }
  ],
};

const GET_BASE = __ENV.GET_BASE || 'http://host.docker.internal:8080';
const POST_BASE = __ENV.POST_BASE || 'http://host.docker.internal:8081';

function randomInt(min, max){
  return Math.floor(Math.random() * (max - min + 1)) + min;
}

export default function() {
  if (Math.random() < 0.8) {
    const payload = JSON.stringify({ user_id: randomInt(1,2), amount: Math.random()*100, description: 'k6 load' });
    const res = http.post(`${POST_BASE}/api/orders`, payload, { headers: { 'Content-Type': 'application/json' } });
    check(res, { 'created': (r) => r.status === 200 || r.status === 201 });
  } else {
    const res = http.get(`${GET_BASE}/api/orders`);
    check(res, { 'list ok': (r) => r.status === 200 });
  }
  sleep(0.05);
}
```

Результат:

<img width="1817" height="254" alt="image" src="https://github.com/user-attachments/assets/7db729d8-e998-4137-89b1-5f2ca5db4d62" />

<img width="894" height="322" alt="{159659EC-AEE2-44E3-A7E0-098FC78199F9}" src="https://github.com/user-attachments/assets/7f80b320-0c6a-4628-a10f-39fef2f51439" />

На диаграмме подключений к БД нужен самый правый пик.

<img width="1034" height="539" alt="{DBDEDCD9-42D0-45B0-AA2A-8E7F744A1982}" src="https://github.com/user-attachments/assets/6fbd7ca3-a745-4207-9749-b6c5a4ab391f" />

4) Мой сценарий

Суть: периодические всплески POST, когда пользователи только заказывают, например, во время распродажи, нужно проверить, как система переживает короткие пики нагрузки на фоне стабильного трафика, восстанавливается ли система после таких пиков.

Сам сценарий с использованием k6:

```
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '1m', target: 200 },
    { duration: '4m', target: 200 },
    { duration: '30s', target: 0 },
  ],
};

const GET_BASE = __ENV.GET_BASE || 'http://host.docker.internal:8080';
const POST_BASE = __ENV.POST_BASE || 'http://host.docker.internal:8081';

function randomInt(min, max) {
  return Math.floor(Math.random() * (max - min + 1)) + min;
}

export default function () {
  const now = Date.now();
  const cycleMs = 50_000;
  const burstMs = 10_000;
  const inBurst = (now % cycleMs) < burstMs;

  const postProb = inBurst ? 0.99 : 0.8;

  if (Math.random() < postProb) {
    const payload = JSON.stringify({
      user_id: randomInt(1, 2),
      amount: Math.random() * 100,
      description: inBurst ? 'k6 burst write' : 'k6 steady',
    });

    const res = http.post(`${POST_BASE}/api/orders`, payload, {
      headers: { 'Content-Type': 'application/json' },
      tags: { name: inBurst ? 'POST /api/orders (burst)' : 'POST /api/orders' },
    });

    check(res, {
      'created (200/201)': (r) => r.status === 200 || r.status === 201,
    });
  } else {
    const res = http.get(`${GET_BASE}/api/orders`, {
      tags: { name: 'GET /api/orders' },
    });

    check(res, {
      'list ok (200)': (r) => r.status === 200,
    });
  }

  sleep(0.05);
}
```

<img width="1833" height="253" alt="image" src="https://github.com/user-attachments/assets/f9fff3c3-d471-457e-957b-962fb10b21de" />

<img width="888" height="333" alt="{7B3E083B-A084-4F72-AE0E-1CC1B1B2CA8E}" src="https://github.com/user-attachments/assets/86094f08-0198-4615-8de7-afdac352ad07" />

<img width="1081" height="525" alt="{36F045FA-4BB6-4556-A887-BAB08A245D68}" src="https://github.com/user-attachments/assets/be0f5939-3390-44d1-8ca5-2d249e545c82" />

### Этапы 3 и 4. Анализ, выводы и как это исправить.

Сценарий 1. Исходный

Метрики:

Фактический RPS: 490 req/s

Количество пользователей доходило до 995 - значит, генератор нагрузки не был бутылочным горлышком

HTTP error rate: 42%

Success rate: 58%, при этом:

POST (created) - 48% успешных

GET (list ok) - 96% успешных

Latency

http_req_duration p95: 1.7 s

http_req_duration med: 200 ms, видно, что большой разрыв между медианой и p95.

Количество активных подключений к БД резко растёт во время нагрузки, видны пики подключений, после которых происходит спад.

CPU Busy 37%.

System Load > 500% - явный признак очередей и ожидания ресурсов.

По памяти все хорошо.

Выводы

Система не выдерживает нагрузку на запись в БД: POST-запросы массово приводят к ошибкам.

Большой разрыв между median и p95 latency указывает на очереди и насыщение БД.

Основной bottleneck - PostgreSQL (connections и latency).

Сценарий 2. "Шторм" (резкий пик до 1000 пользователей за 10 секунд)

Метрики:

Фактический RPS: 454 req/s

Количество пользователей доходило до 987 - генератор нагрузки не является бутылочным горлышком.

HTTP error rate: 47%

Success rate: 53%, при этом:
POST (created) - 44% успешных
GET (list ok) - 87% успешных

Latency

http_req_duration p95: 2.9 s

http_req_duration med: 800 ms

Разрыв между медианой и p95 значительно больше, чем в базовом сценарии, что указывает на резкое накопление очередей при пике нагрузки.

Количество активных подключений к БД резко возрастает в момент шторма.

Виден максимальный пик подключений, после которого происходит спад.

Восстановление числа подключений происходит с задержкой.

Ресурсы

CPU Busy: 37%

System Load: > 130%

По памяти OK.

Выводы

Система не справляется с резким скачком нагрузки.

POST-запросы массово приводят к ошибкам сразу после начала пика.

Latency резко растёт из-за очередей и перегрузки БД.

Основной bottleneck - PostgreSQL.

Сценарий 3. "Волна" (плавный рост до 500 VU за 2 минуты)

Метрики:

Фактический RPS: 417 req/s

Количество пользователей:
VU плавно растут до 499 - генератор нагрузки не является бутылочным горлышком.

HTTP error rate: 43%

Success rate: 57%, при этом:

POST (created) - 47% успешных

GET (list ok) - 94% успешных

Latency

http_req_duration p95: 1.6 s

http_req_duration med: 400 ms

Количество активных подключений к БД увеличивается во время нагрузки.

Ресурсы

CPU Busy: 25%

System Load: 390%

RAM: стабильна

Выводы

Плавный рост нагрузки дает системе время адаптироваться, но не решает проблему write-нагрузки.

Ошибки и рост latency появляются даже без резких скачков нагрузкки.

Максимальная устойчивая нагрузка системы ниже 500 VU.

Основной bottleneck остаётся PostgreSQL.

Сценарий 4. Собственный (периодические всплески, во время которых идет практически только запись)

Метрики:

Фактический RPS: 434 req/s

Количество пользователей постоянно 200 VU - генератор нагрузки стабилен.

HTTP error rate: 42%

Success rate: 58%,
при этом:

POST (created) — 50% успешных

GET (list ok) — 100% успешных

Latency

http_req_duration p95: 900 ms

http_req_duration med: 320 ms

Latency растет именно во время периодов, когда идет только POST, затем частично восстанавливается.

Количество активных подключений к БД растет волнообразно, то возрастает, то спадает.

QPS 367.

Каждый всплеск числа POST сопровождается всплеском подключений.

После всплесков происходит частичное восстановление.

Ресурсы

CPU Busy: 67%

System Load: >700%

RAM: стабильна

Выводы

Кратковременные всплески записи существенно нагружают БД, но не приводят к полному отказу.

Система демонстрирует частичную устойчивость, но работает на грани возможностей БД.

Bottleneck по-прежнему - PostgreSQL (connections + latency).

#### Восстанавливается ли система после пиковых нагрузок?

Что говорит о том, что восстановление происходит:

1) снижение нагрузки после пика;

2) после пиковых нагрузок активные подключения к БД снижаются;

3) CPU снижается;
  
4) deadlock отсутствуют;

5) нет постоянного роста числа активных подключений к БД, после пиков уменьшаются.

Что говорит о неполном восстановлении:

1) error rate остается высоким;

2) ошибки не исчезают даже при снижении нагрузки;

3) большая разница между p95 и медианой даже в сценарии "волна".

#### Вывод

Latency (p95/p99) растет при увеличении нагрузки, особенно при POST-запросах.

Error rate стабильно высокий во всех сценариях.

DB connections соотносятся с ростом latency и ошибок.

Высокое значение System Load указывает на накопление очередей и ожидание ресурсов, что связано с перегрузкой базы данных. При этом CPU и память не являются узким местом системы.

Главный bottleneck системы - база данных PostgreSQL, особенно обработка POST-запросов.

#### Предложения по улучшению

Ограничить и настроить connection pool к PostgreSQL, чтобы избежать перегрузки по подключениям.

Снизить POST-нагрузку на БД: асинхронная обработка создания заказов или batching.

Добавить кеширование GET /api/orders, чтобы чтение не конкурировало с записью.

Ввести rate limiting или backpressure на уровне nginx и backend для защиты от пиков нагрузки.
