# DNS

# DNS

---

## 📋 Содержание

1. [Блок 1 — Теория и основы](about:blank#%D0%B1%D0%BB%D0%BE%D0%BA-1--%D1%82%D0%B5%D0%BE%D1%80%D0%B8%D1%8F-%D0%B8-%D0%BE%D1%81%D0%BD%D0%BE%D0%B2%D1%8B)
2. [Блок 2 — Типы DNS-записей](about:blank#%D0%B1%D0%BB%D0%BE%D0%BA-2--%D1%82%D0%B8%D0%BF%D1%8B-dns-%D0%B7%D0%B0%D0%BF%D0%B8%D1%81%D0%B5%D0%B9)
3. [Блок 3 — Инструменты диагностики](about:blank#%D0%B1%D0%BB%D0%BE%D0%BA-3--%D0%B8%D0%BD%D1%81%D1%82%D1%80%D1%83%D0%BC%D0%B5%D0%BD%D1%82%D1%8B-%D0%B4%D0%B8%D0%B0%D0%B3%D0%BD%D0%BE%D1%81%D1%82%D0%B8%D0%BA%D0%B8)
4. [Блок 4 — DNS на Linux-хосте](about:blank#%D0%B1%D0%BB%D0%BE%D0%BA-4--dns-%D0%BD%D0%B0-linux-%D1%85%D0%BE%D1%81%D1%82%D0%B5)
5. [Блок 5 — Split-horizon и безопасность](about:blank#%D0%B1%D0%BB%D0%BE%D0%BA-5--split-horizon-%D0%B8-%D0%B1%D0%B5%D0%B7%D0%BE%D0%BF%D0%B0%D1%81%D0%BD%D0%BE%D1%81%D1%82%D1%8C)
6. [Глоссарий терминов](about:blank#%D0%B3%D0%BB%D0%BE%D1%81%D1%81%D0%B0%D1%80%D0%B8%D0%B9-%D1%82%D0%B5%D1%80%D0%BC%D0%B8%D0%BD%D0%BE%D0%B2)
7. [Шпаргалка команд](about:blank#%D1%88%D0%BF%D0%B0%D1%80%D0%B3%D0%B0%D0%BB%D0%BA%D0%B0-%D0%BA%D0%BE%D0%BC%D0%B0%D0%BD%D0%B4)

---

## Блок 1 — Теория и основы

### 1.1 Что такое DNS и зачем DevOps-инженеру

DNS (Domain Name System) — глобальная распределённая база данных, переводящая доменные имена в IP-адреса.

**Почему DNS важен для DevOps:**
- Деплой сервисов (управление TTL)
- Service Discovery в Kubernetes
- Балансировка нагрузки (Round Robin DNS)
- TLS-сертификаты
- Мониторинг и аудит

---

### 1.2 Иерархия DNS

```
. (root)
└── .net (TLD — Top Level Domain)
    └── corp.internal (авторитативный NS)
        └── db01.dc.corp.internal (конкретная запись)
```

| Уровень | Кто | Что делает |
| --- | --- | --- |
| Root servers | 13 кластеров (ICANN и партнёры) | Знают адреса всех TLD-серверов |
| TLD servers | `.com`, `.ru`, `.org`, `.net` | Знают NS-серверы конкретных доменов |
| Authoritative NS | Ваш хостинг / Route53 / Cloudflare / корп. DNS | Хранят реальные записи (`A`, `CNAME` и т.д.) |
| Recursive Resolver | `8.8.8.8`, `10.10.0.1`, корпоративный DNS | Обходит цепочку от имени клиента |

---

### 1.3 Полный цикл DNS-запроса

```
App → Stub Resolver → OS Cache → Recursive Resolver
                                        ↓
                                 Root Server (.)
                                        ↓
                                 TLD Server (.internal)
                                        ↓
                             Authoritative NS (corp.internal)
                                        ↓
                              Ответ: 10.20.10.1
```

**Пошагово:**
1. Браузер/приложение спрашивает локальный резолвер
2. Резолвер проверяет `/etc/hosts` и кэш ОС
3. Если не найдено — идёт к **recursive resolver** (например, `10.10.0.1`)
4. Recursive resolver спрашивает **root server**: «Кто отвечает за `.internal`?»
5. Root server отвечает адресами TLD-серверов
6. Recursive resolver спрашивает **TLD server**: «Кто отвечает за `corp.internal`?»
7. TLD отвечает адресом **авторитативного NS**
8. Recursive resolver спрашивает авторитативный NS: «Что такое `db01.corp.internal`?»
9. Получает IP и кэширует на время TTL

---

### 1.4 Рекурсивный vs Итеративный запрос

| Тип | Кто работает | Как |
| --- | --- | --- |
| **Рекурсивный** | Recursive Resolver | Сам обходит всю цепочку за клиента, возвращает готовый ответ |
| **Итеративный** | DNS-серверы между собой | Каждый сервер возвращает «спроси вот там», клиент идёт дальше сам |
| **Non-recursive (кэш)** | Resolver / OS | Отвечает из кэша мгновенно |

> **Аналогия:** рекурсивный — попросить помощника найти книгу (он всё сделает сам). Итеративный — самому ходить от стойки к стойке.
> 

---

### 1.5 TTL — главный параметр для DevOps

TTL (Time To Live) — сколько секунд запись кэшируется.

- **Высокий TTL** (3600–86400 сек) → меньше нагрузки, но **медленные изменения** при деплое
- **Низкий TTL** (60–300 сек) → быстрые изменения, но **больше запросов**

> ⚠️ **Best practice перед миграцией:** за 24–48 часов до деплоя снизьте TTL до 60–300 сек. После успешного переключения верните высокий TTL.
> 

---

## Блок 2 — Типы DNS-записей

| Запись | Назначение | Пример |
| --- | --- | --- |
| `A` | Имя → IPv4 | `db01.corp.internal. 3600 IN A 10.10.5.11` |
| `AAAA` | Имя → IPv6 | `db01.corp.internal. IN AAAA 2001:db8::1` |
| `CNAME` | Алиас имени → другое имя | `api.corp.internal. CNAME api-v2.corp.internal.` |
| `PTR` | IP → имя (обратный DNS) | `11.5.10.10.in-addr.arpa. → srv01.dc.corp.internal.` |
| `MX` | Почтовый сервер домена | `corp.internal. IN MX 10 mail.corp.internal.` |
| `TXT` | Произвольный текст (SPF, DKIM, верификация) | `v=spf1 ip4:10.10.0.0/16 -all` |
| `NS` | Авторитативный DNS-сервер зоны | `corp.internal. IN NS ns1.dc.corp.internal.` |
| `SOA` | Start of Authority — метаданные зоны | `dig SOA corp.internal` |
| `SRV` | Сервис + порт + приоритет | `_kerberos._tcp.corp.internal. SRV 0 100 88 dc01.corp.internal.` |
| `CAA` | Какие CA могут выдавать сертификаты | `corp.internal. CAA 0 issue "letsencrypt.org"` |

> **Важно:** `CNAME` нельзя ставить на apex-домен (`corp.internal.`) — только на поддомены!
> 

---

## Блок 3 — Инструменты диагностики

### dig — основной инструмент

```bash
# A-запись к конкретному DNS-серверу
dig @10.10.0.1 corp.internal A

# Краткий вывод — только IP
dig +short db01.dc.corp.internal

# Трейс всей цепочки от root
dig +trace srv01.dc.corp.internal

# Обратный PTR-запрос
dig -x 10.10.5.11

# TTL записи
dig +ttlunits corp.internal A

# SOA зоны
dig SOA corp.internal

# Зонный трансфер (в корп. окружении обычно заблокирован)
dig @10.10.0.1 corp.internal AXFR
```

### Разбор вывода `dig`

```
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 45215
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 4, AUTHORITY: 0, ADDITIONAL: 1
```

| Флаг | Значение |
| --- | --- |
| `aa` | Authoritative Answer — ответ от владельца зоны, не из кэша |
| `rd` | Recursion Desired — клиент просит рекурсивный запрос |
| `ra` | Recursion Available — сервер поддерживает рекурсию |
| `qr` | Query Response — это ответ, а не запрос |

| Статус | Значение |
| --- | --- |
| `NOERROR` | Запрос выполнен успешно |
| `NXDOMAIN` | Домен/запись не существует |
| `SERVFAIL` | DNS-сервер сам не смог получить ответ |

### Другие инструменты

```bash
# Резолвинг через NSS (как делает приложение)
getent hosts db01.corp.internal

# Простой запрос (обходит /etc/hosts)
nslookup db01.corp.internal

# Управление systemd-resolved
resolvectl status
resolvectl flush-caches

# Проверка TCP/UDP порта
nc -zvw 3 10.10.0.1 53          # UDP/53
nc -zvw 3 db01.corp.internal 5432  # PostgreSQL порт
```

---

## Блок 4 — DNS на Linux-хосте

### Порядок резолвинга (NSS)

```bash
cat /etc/nsswitch.conf | grep hosts
# hosts: files dns myhostname
#        ↑     ↑    ↑
#   /etc/hosts DNS  hostname
```

> На хостах с SSSD: `hosts: files sss dns myhostname`
> 
> 
> **Не редактировать `nsswitch.conf` вручную** — управляется через `authselect`!
> 

### `/etc/resolv.conf`

```bash
# Пример resolv.conf в корпоративной инфраструктуре
nameserver 10.10.0.1      # Primary DNS
nameserver 10.10.0.2      # Secondary DNS
search dc.corp.internal corp.internal remote.corp.internal
options ndots:5
```

| Директива | Назначение |
| --- | --- |
| `nameserver` | IP DNS-сервера для запросов |
| `search` | Суффиксы для коротких имён (без точек) |
| `ndots` | Сколько точек нужно для «полного» имени |

> ⚠️ На хостах с **NetworkManager** — не редактировать `resolv.conf` вручную! Он перезапишется.
> 

### Настройка DNS через NetworkManager

```bash
# Посмотреть активные соединения
nmcli connection show

# Посмотреть DNS соединения
nmcli connection show "ens3" | grep DNS

# Прописать корпоративный DNS
sudo nmcli connection modify "ens3" \
    ipv4.dns "10.10.0.1 10.10.0.2" \
    ipv4.dns-search "dc.corp.internal corp.internal"

sudo nmcli connection up "ens3"
```

### Round Robin DNS

Пример — `corp.internal` имеет 4 A-записи (балансировка нагрузки):

```
corp.internal. 600 IN A 10.20.10.1
corp.internal. 600 IN A 10.20.10.2
corp.internal. 600 IN A 10.20.10.3
corp.internal. 600 IN A 10.20.10.4
```

Каждый запрос получает IP в разном порядке → нагрузка распределяется.

---

## Блок 5 — Split-horizon и безопасность

### Split-horizon DNS

Домен отдаёт **разные** ответы внутри и снаружи корпоративной сети.

```
Снаружи:   corp.internal → NXDOMAIN (домен не виден из интернета)
Внутри:    corp.internal → 10.20.10.1 (внутренний сервер)
```

### Форвардинг DNS

```
Pod в k8s → CoreDNS → корп. DNS (10.10.0.1) → ответ
```

Зоны типа `*.corp.internal`, `*.dc.corp.internal` форвардятся на внутренние DNS-серверы.

### DNS-атаки

| Атака | Суть | Защита |
| --- | --- | --- |
| DNS Spoofing | Подмена ответа | DNSSEC |
| Cache Poisoning | Отравление кэша | DNSSEC, TSIG |
| DNS Hijacking | Перехват трафика | DoT, DoH |
| AXFR abuse | Кража всех записей зоны | Блокировка TCP/53, TSIG |

### Зонные трансферы и TSIG

```bash
# AXFR (полный трансфер зоны) — в корп. среде обычно заблокирован
dig @10.10.0.1 corp.internal AXFR
# → Transfer failed (защита корп. DNS)

# TSIG — криптоподпись между Primary и Secondary DNS
# Без ключа — трансфер запрещён
```

---

## Глоссарий терминов

| Термин | Расшифровка | Пояснение |
| --- | --- | --- |
| DNS | Domain Name System | Система доменных имён |
| FQDN | Fully Qualified Domain Name | Полное имя: `db01.corp.internal.` (с точкой) |
| TTL | Time To Live | Время жизни записи в кэше (секунды) |
| NS | Name Server | Сервер, авторитативный для зоны |
| SOA | Start of Authority | Метаданные зоны DNS |
| RR | Resource Record | Любая запись в DNS |
| AXFR | Authoritative Zone Transfer | Полная передача зоны между NS |
| TSIG | Transaction SIGnature | Криптоподпись DNS-транзакций |
| Split-horizon | — | Разные ответы DNS внутри/снаружи сети |
| NDots | — | Порог точек для прямого запроса без search |
| Stub Resolver | — | Минималистичный резолвер в ОС |
| Recursive Resolver | — | Сервер, обходящий цепочку DNS за клиента |
| Forwarder | — | DNS, перенаправляющий запросы другому NS |
| Round Robin | — | Балансировка через несколько A-записей |
| DNSSEC | DNS Security Extensions | Криптоподпись зон DNS |

---

## Шпаргалка команд

```bash
# ===== DIG =====
dig @<dns-server> <hostname> <type>     # Запрос к конкретному DNS
dig +short <hostname>                    # Только IP
dig +trace <hostname>                    # Трейс от root
dig -x <ip>                             # Обратный PTR-запрос
dig +ttlunits <hostname> A              # Показать TTL
dig SOA <zone>                          # Метаданные зоны
dig <zone> AXFR                         # Зонный трансфер (если разрешён)

# ===== NSS / HOSTS =====
getent hosts <hostname>                 # Резолвинг через NSS (как app)
cat /etc/hosts                          # Статические записи
cat /etc/nsswitch.conf | grep hosts     # Порядок резолвинга
cat /etc/resolv.conf                    # DNS-серверы и search-домены

# ===== NETWORK MANAGER =====
nmcli connection show                   # Список соединений
nmcli connection show "<conn>" | grep DNS
sudo nmcli connection modify "<conn>" ipv4.dns "<ip1> <ip2>"
sudo nmcli connection modify "<conn>" ipv4.dns-search "<domain1> <domain2>"
sudo nmcli connection up "<conn>"

# ===== SYSTEMD-RESOLVED =====
resolvectl status                       # Статус резолвера
resolvectl flush-caches                 # Сбросить DNS-кэш
resolvectl query <hostname>             # Запрос через resolved

# ===== KUBERNETES / COREDNS =====
kubectl get configmap coredns -n kube-system -o yaml
kubectl rollout restart deployment coredns -n kube-system
kubectl run dns-debug --image=busybox:1.35 --rm -it -- sh

# ===== CONNECTIVITY =====
nc -zvw 3 <host> 53                     # Проверить порт DNS (UDP/TCP)
nc -zvw 3 <host> <port>                 # Проверить любой порт
```