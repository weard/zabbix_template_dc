# Template Windows Domain Controller — краткий README

Шаблон мониторинга единственного контроллера домена (Windows Server, AD + DNS) для Zabbix 7.4.
Дополняет связанные шаблоны: **Windows by Zabbix agent active** (CPU/RAM/сеть) и **AD DS Monitoring and Attack Detection** (атаки/аудит AD). Свои CPU/RAM не дублирует.

## Установка

1. Data collection → Templates → Import — выбрать `template-dc.yaml`.
2. Хост DC → шаблоны → привязать «Template Windows Domain Controller».
3. Подождать ~5 минут, проверить Latest data (все порты должны быть = 1, службы = 0).

## Что мониторит (22 айтема)

### Службы AD-стека (agent, passive, 1 мин)

| Служба | Айтем | Триггер |
|---|---|---|
| NTDS (Directory Service) | `service.info[NTDS,state]` | `<>0` → DISASTER |
| DNS Server | `service.info[DNS,state]` | `<>0` → DISASTER |
| Netlogon | `service.info[Netlogon,state]` | `<>0` → HIGH |
| KDC (Kerberos) | `service.info[kdc,state]` | `<>0` → DISASTER |
| W32Time | `service.info[W32Time,state]` | `<>0` → WARNING |

0 = работает, любое другое значение = проблема.

### Порты с сервера Zabbix (simple checks, не зависят от агента)

| Порт | Айтем | Триггер |
|---|---|---|
| LDAP 389 | `net.tcp.service[tcp,,389]` | 0 три мин → HIGH |
| LDAPS 636 | `net.tcp.service[tcp,,636]` | 0 пять мин → WARNING |
| SMB 445 (SYSVOL/GPO) | `net.tcp.service[tcp,,445]` | 0 три мин → HIGH |
| DNS 53 UDP | `net.udp.service[udp,,53]` | 0 три мин → DISASTER |
| DNS 53 TCP | `net.tcp.service[tcp,,53]` | 0 три мин → HIGH |
| Kerberos 88 UDP | `net.udp.service[udp,,88]` | 0 три мин → HIGH (филиалы не получают тикеты) |

Показывают доступность КД так, как её видят филиалы через фаервол. Если агент упал, а порты отвечают — проблема в агенте, а не в КД.

### Журналы Windows (eventlog, дубликаты пропускаются)

| Айтем | События | Триггер |
|---|---|---|
| Directory Service | 2042 (USN rollback — откат из снапшота!) | сразу → DISASTER |
| System | 5719/5722 (Netlogon) | 3+ за 30 мин → AVERAGE |
| System | 5774 (регистрация DNS-записей) | 2+ за 30 мин → AVERAGE |
| Security | 4740 (блокировка учетки) | сразу → WARNING |
| Security | 4728/4732/4756 (добавление в доменные группы) | сразу → HIGH |
| Security | 4725/4722 (отключение/включение учетки) | сразу → INFO |
| DNS Server | все Error (зоны, загрузка, дин. обновления) | 2+ за 30 мин → AVERAGE |

История айтемов по Security — готовый журнал изменений групп и блокировок.

### Прочее

| Айтем | Триггер |
|---|---|
| `agent.ping` (active) | nodata 5 мин → HIGH (агент/хост недоступен) |
| `system.localtime` | `fuzzytime` > 60 сек → WARNING (дрейф времени, Kerberos ломается после 5 мин) |
| `vfs.fs.size[C:,pused]` | > 85% → WARNING, > 95% → HIGH (там же NTDS.dit) |
| `system.uptime` | > 45 дней → INFO (напоминание, не автоматика) |

## Важные оговорки

- **КД не входит в принудительную недельную перезагрузку станций.** Ребут — только вручную, в нерабочее время, с Maintenance в Zabbix: на время ребута филиалы полностью остаются без AD/DNS.
- События 5719/5722 и 5774 заведомо шумные на перегруженном/нестабильном канале — триггеры настроены на повторы (3+ / 2+ за 30 мин), а не на единичные срабатывания.
- `eventlog[Security]` читает агент от SYSTEM — дополнительных прав не нужно, аудит 4740/4728/4732/4756/4725/4722 на КД включен по умолчанию.

## Файлы

- `zabbix\template-dc.yaml` — экспорт шаблона (version 7.4, UUID в формате v4).
