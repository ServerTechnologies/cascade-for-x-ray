# Скрипт создания каскада на голом ядре Xray

Bash-скрипт для автоматической настройки каскада на голом ядре Xray. Корректно работает при использовании совместно с моим скриптом установки голого ядра [Ссылка на скрипт](https://github.com/ServerTechnologies/simple-xray-core). 
Принимает VLESS-ссылку, парсит из неё все параметры и автоматически обновляет `config.json` — правила маршрутизации и outbound-подключение.

Поддерживаемые протоколы:
- **VLESS + XHTTP + REALITY** (`type=xhttp`)
- **VLESS + TCP + REALITY + xtls-rprx-vision** (`type=tcp`)

---

## VPS для установки ядра

Для установки ядра нам понадобится VPS-сервер. Его можно приобрести у [ishosting](https://ishosting.com/affiliate/Mzg3IzY=).
Сервис предоставляет более 36 локаций на выбор.

---

## Требования

Если вы использовали мой скрипт для установки ядра, то почти все необходимые компоненты уже установлены на сервер.
Этот скрипт использует **python3**. Проверить его наличие на сервере можно командой:
```sh
python3 --version
```
Если вы получили версию python3 - он установлен. Если вы получили ошибку, установите его командой
```sh
apt install python3
```

### Установка Xray-core

Можно использовать официальную документацию, либо установить ядро с помощью моего скрипта: \
[Ссылка на скрипт](https://github.com/ServerTechnologies/simple-xray-core) \
[Видео на YouTube](https://youtu.be/PHn5JE9rXgg)

---

## Использование скрипта
Скрипт запускается на входящем (российском) сервере

```sh
wget -qO- https://raw.githubusercontent.com/ServerTechnologies/cascade-for-x-ray/refs/heads/main/cascad | bash
```

Скрипт запросит ссылку для подключения с исходящего сервера:

```
Вставьте vless-ссылку: vless://...
```

---

## Что делает скрипт

### 1. Парсинг VLESS-ссылки

Из ссылки автоматически извлекаются все параметры подключения:

| Переменная | Параметр в ссылке | Описание |
|---|---|---|
| `address` | хост после `@` | IP-адрес или домен сервера |
| `port` | порт после `:` | Порт сервера |
| `id` | UUID до `@` | Идентификатор пользователя |
| `name` | фрагмент после `#` | Имя подключения (tag в конфиге) |
| `type` | `type=` | Тип транспорта: `xhttp` или `tcp` |
| `publickey` | `pbk=` | Публичный ключ REALITY |
| `servername` | `sni=` | Server Name Indication |
| `fingerprint` | `fp=` | TLS-отпечаток |
| `shortid` | `sid=` | Short ID для REALITY |

После парсинга скрипт выводит распознанные параметры:

```
=== Параметры подключения ===
  name:        MyServer
  address:     1.2.3.4
  port:        443
  type:        xhttp
  security:    reality
  servername:  example.com
```

### 2. Обновление гео-файлов

Скрипт удаляет китайские геофайлы и скачивает актуальные базы для RU-сегмента из репозитория [russia-v2ray-rules-dat](https://github.com/runetfreedom/russia-v2ray-rules-dat):

- `geoip.dat` — российские IP-адреса
- `geosite.dat` — российские домены

### 3. Резервная копия конфига

Перед изменением создаётся резервная копия:

```sh
/usr/local/etc/xray/config.json  →  /usr/local/etc/xray/config.backup
```

Если после запуска скрипта что-то пошло не так, можно восстановить конфигурацию из резервной копии:

```sh
cp /usr/local/etc/xray/config.backup /usr/local/etc/xray/config.json
systemctl restart xray
```
### 4. Обновление правил маршрутизации

В `config.json` перезаписывается секция `routing.rules`:

| Правило | Действие |
|---|---|
| Домены `.ru`, `.su`, `.xn--p1ai` и `ru-available-only-inside` | `direct` — трафик идет напрямую с входящего сервера |
| IP-адреса из `geoip.dat:ru` | `direct` — трафик идет напрямую с входящего сервера |
| Рекламные домены `category-ads-all` | `block` — трафик блокируется |
| Весь остальной трафик (TCP + UDP) | трафик идет дальше на исходящий сервер |

### 5. Настройка outbound-подключения

В зависимости от значения `type` в ссылке применяется соответствующий конфиг:

**`type=xhttp`** — VLESS поверх XHTTP + REALITY с мультиплексированием:
```
network:   xhttp
security:  reality
xmux:      maxConcurrency 16-32, hMaxRequestTimes 600-900
```

**`type=tcp`** — VLESS поверх TCP + REALITY с потоком xtls-rprx-vision:
```
network:   tcp
security:  reality
flow:      xtls-rprx-vision
```

### 6. Скрипт автообновления гео-файлов

Периодически стоит обновлять геофайлы. Это можно сделать вручную, либо использовать скрипт автоматического обновления геофайлов. По пути `/usr/local/share/xray/renewgeofiles` создаётся скрипт, который можно добавить в `cron` для периодического обновления баз геолокации без полной перенастройки.

Пример добавления в cron (обновление раз в неделю):

```sh
crontab -e
# Добавить строку:
0 3 * * 1 bash /usr/local/share/xray/renewgeofiles
```

### 7. Перезапуск Xray

```bash
systemctl restart xray
systemctl status xray
```

---


## Необходимые компоненты и проверка перед запуском

| Компонент | Назначение | Установка |
|---|---|---|
| **Xray-core** | Прокси-сервер | см. выше |
| **jq** | Редактирование JSON | `apt install jq` |
| **wget** | Загрузка гео-файлов | `apt install wget` |
| **python3** | URL-декодирование параметров ссылки | `apt install python3` |
| **systemd** | Управление службой xray | присутствует в Ubuntu по умолчанию |

```sh
xray version                                      # Xray установлен
jq --version                                      # jq установлен
wget --version                                    # wget установлен
python3 --version                                 # python3 установлен
systemctl status xray                             # служба существует
cat /usr/local/etc/xray/config.json | jq .        # конфиг валидный JSON
```

Если вдруг вы решили использовать этот скрипт без моего скрипта установки ядра, то обязательно проверьте наличие необходимых компонентов и измените в скрипте пути до файлов конфигурации и геофайлов, если они отличаются от путей по умолчанию.