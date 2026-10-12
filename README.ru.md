# Access Manager

**[English](https://github.com/Nord1cWarr1or/Access-Manager/blob/main/README.md)** | **Русский**

Access Manager — плагин AMX Mod X для Counter-Strike 1.6. Он загружает аккаунты администраторов из базы MySQL или локального INI-файла и пускает игроков по SteamID, IP или игровому нику. Пароли проверяются, флаги нескольких совпавших аккаунтов складываются в один набор, а просроченные аккаунты отбрасываются. Другие плагины читают данные доступа через нативы и форварды из таблиц ниже.

## Возможности

- Два источника аккаунтов: таблицы `amx_amxadmins` в MySQL или `am_users.ini` с `am_local_mode "1"`.
- Вход по SteamID, IP или нику, с проверкой пароля и опциональным киком при ошибке.
- Несколько совпавших аккаунтов сливаются в один набор флагов; у каждого свой срок действия.
- Повторная авторизация после смены ника (хук ReAPI).
- После каждой загрузки из базы пишется `am_users_sql_backup.ini`; бэкап читается при старте, так что аккаунты на месте ещё до ответа запроса.
- Подхватывает соединение с базой Fresh Bans или Lite Bans, если они установлены.
- Локализованные сообщения через `data/lang/access_manager.txt`.

## Требования

- AMX Mod X с модулем ReAPI — хук смены ника идёт через `RG_CBasePlayer_SetClientUserInfoName`.
- Модуль `sqlx` — нужен только в режиме базы; локальный режим работает без него.

### Необязательные зависимости

- Fresh Bans или Lite Bans — без них плагин открывает своё соединение по `am_host` / `am_user` / `am_password` / `am_database`.
- AmxBans RBS (SKAJIbnEJIb) и Admin Load (Fant0M) — плагин отдаёт им `amxbans_get_expired()` и `adminload_get_expired()`, панели работают без изменений.

## Установка

1. Убедитесь, что на сервере стоит ReAPI.
2. Скомпилируйте `scripting/AccessManager.sma`:
   ```bash
   amxxpc AccessManager.sma -oAccessManager.amxx
   ```
3. Скопируйте `AccessManager.amxx` в `addons/amxmodx/plugins/`.
4. Добавьте `AccessManager.amxx` в `addons/amxmodx/configs/plugins.ini`.
5. Скопируйте `configs/AccessManager.cfg` и `configs/am_users.ini` в `addons/amxmodx/configs/`, а `data/lang/access_manager.txt` — в `addons/amxmodx/data/lang/`.
6. Заполните CVar'ы базы в `AccessManager.cfg` либо выставьте `am_local_mode "1"` и работайте только через `am_users.ini`.
7. Смените карту или перезапустите сервер.

## Команды

| Команда | Доступ | Описание |
|---------|--------|----------|
| `amx_reloadadmins` | `ADMIN_CFG` | Перезагрузить аккаунты из базы или INI-файла. Игроки без флага получают доступ при `am_allow_reload_to_anyone "1"`, с кулдауном. |

## CVars

| CVar | По умолчанию | Описание |
|------|--------------|----------|
| `am_local_mode` | `0` | `1` — читать аккаунты только из `am_users.ini`, без базы. |
| `am_allow_reload_to_anyone` | `0` | Разрешить `amx_reloadadmins` игрокам без флага доступа. |
| `am_reloadadmins_cooldown` | `30` | Кулдаун в секундах для перезагрузки без флага. |
| `am_long_query_time` | `10` | Запросы дольше этого значения (в секундах) пишутся в лог как долгие. |
| `am_host` | `""` | Хост MySQL. |
| `am_user` | `""` | Пользователь MySQL. |
| `am_password` | `""` | Пароль MySQL. |
| `am_database` | `""` | Имя базы данных. |
| `am_server_ip` | `127.0.0.1:27015` | Адрес сервера в виде `IP:PORT`, сверяется с `amx_serverinfo.address`. |
| `am_prune_expired` | `0` | Удалять просроченные аккаунты, созданные через `AccessManager_AddAccess()`. Нужен define `ADD_ACCESS_FEATURE` и колонка `ingame` (int) в `amx_amxadmins`. |
| `amx_password_field` | `_pw` | Ключ инфо клиента, в котором передаётся пароль (стандартный CVar AMX Mod X, привязан плагином). |
| `amx_default_access` | `z` | Флаги для игроков без совпадения (стандартный CVar AMX Mod X, привязан плагином). |

## Нативы и форварды

Подключите API и вызывайте:

```pawn
#include <access_manager>

public client_admin(pPlayer, bitAccessFlags) {
    // срабатывает после авторизации игрока
}

new szNick[MAX_ACCOUNT_NAME_LENGTH];
if (AccessManager_GetNickName(pPlayer, szNick, charsmax(szNick))) {
    // ник аккаунта из базы или am_users.ini
}
```

| Натив | Описание |
|-------|----------|
| `AccessManager_GetNickName(pPlayer, szBuffer[], iMaxLen)` | Ник аккаунта, с которым игрок авторизован. Возвращает длину строки или `0` — игрок не в сети либо аккаунта нет. |
| `AccessManager_GetNickNameByAuthID(const szAuthID[], szBuffer[], iMaxLen)` | Ник по authid (SteamID, IP или имя — как записано в аккаунте). Возвращает длину или `0`, если совпадений нет. |
| `AccessManager_ReAuthPlayer(pPlayer)` | Запустить авторизацию заново для подключённого игрока. `1` — успех, `0` — игрок не в сети. |
| `admin_expired(pPlayer)` | Время истечения аккаунта игрока, `0` — бессрочный. |
| `amxbans_get_expired(pPlayer)` | То же значение, псевдоним для AmxBans RBS. |
| `adminload_get_expired(pPlayer)` | То же значение, псевдоним для Admin Load. |
| `AccessManager_AddAccess(pPlayer, const szAuthID[], const szFlags[], iMinutes, iCaller)` | Выдать временный доступ. Собирается только с define `ADD_ACCESS_FEATURE`; возвращает id запроса, `-1` — источник аккаунтов недоступен. |

| Форвард | Описание |
|---------|----------|
| `client_admin(pPlayer, bitAccessFlags)` | Игрок авторизован; отдаёт слитый набор флагов. |
| `amxx_admin_access(pPlayer, bitAccessFlags, iExpireTimeStamp)` | То же событие плюс время истечения. |
| `AccessManager_AccessAdded(iUserID, iCallerID)` | `AccessManager_AddAccess()` закончил работу; собирается только с `ADD_ACCESS_FEATURE`. |

## Технические заметки

- Флаги авторизации в `am_users.ini`: `a` — кик при неверном пароле, `c` — ключ аккаунта SteamID, `d` — IP, `e` — без проверки пароля.
- Поле ника — идентификация аккаунта, на авторизацию оно не влияет.
- Сначала плагин берёт коннект Fresh Bans (`fbans_sql_connected`) или Lite Bans (`lite_bans_sql_init`), и только потом свои CVar'ы.
- `AccessManager_AddAccess()` и `am_prune_expired` — опция на этапе сборки: раскомментируйте `ADD_ACCESS_FEATURE` в исходнике, добавьте колонку `ingame` (int) в `amx_amxadmins` и пересоберите плагин.

## Авторы

- **mx?!** — автор.
- **Nordic Warrior** — добавил нативы `AccessManager_GetNickName()` и `AccessManager_GetNickNameByAuthID()`, написал этот README.
- **Neygomon** — «Admin Loader», оттуда взята часть механики авторизации.

## Лицензия

TODO: в репозитории пока нет файла `LICENSE`.