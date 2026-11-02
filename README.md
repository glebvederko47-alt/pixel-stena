# Пиксельная стена

Общий холст 512×512 для всех игроков Telegram: каждый ставит по пикселю раз в 20 секунд (запас до 10), видно, кто что нарисовал, есть рейтинг недели, приглашения друзей и покупки за Telegram Stars.

- `index.html` — игра (Telegram Mini App). Без адреса сервера работает в демо-режиме с имитацией других игроков.
- `server/` — сервер на Cloudflare Workers + D1: холст, заряды, оплата Stars, рейтинг, приглашения.
  - `config.js` — размер холста, заряды, пакеты. Пакеты продублированы в `PACKS` в `index.html`.
  - `test_worker.mjs` — 42 проверки без Cloudflare: `node server/test_worker.mjs`.

## Запуск

1. **Бот.** В @BotFather: `/newbot`, токен сохранить в `server/token.txt` (файл не попадает в git).
2. **Игра на GitHub Pages.** Репозиторий `pixel-stena`, в нём `index.html` и этот README. Адрес: `https://glebvederko47-alt.github.io/pixel-stena/`.
3. **Cloudflare.** В `server/`:
   ```
   npx wrangler login
   npx wrangler d1 create pixel-stena          # database_id вписать в wrangler.toml
   npx wrangler d1 execute pixel-stena --remote --file=schema.sql
   npx wrangler secret put BOT_TOKEN
   npx wrangler secret put ADMIN_ID            # ваш Telegram id — для /refund
   python setup_bot.py https://pixel-stena.<аккаунт>.workers.dev   # создаёт webhook_secret.txt
   npx wrangler secret put WEBHOOK_SECRET      # значение из webhook_secret.txt
   npx wrangler deploy
   ```
   В `wrangler.toml` поменять `SUPPORT_CONTACT` на свой ник.
4. **Подключить игру к серверу:** в `index.html` вписать `const API = 'https://pixel-stena.<аккаунт>.workers.dev'` и `const BOT = 'имя_бота'`, обновить Pages.
5. **Ссылки на место холста** (`t.me/<бот>?startapp=p_X_Y`): в @BotFather → бот → Bot Settings → Configure Mini App → включить Main Mini App с адресом игры.

## Как устроено

- Холст хранится снимком (байт-цвет на пиксель, base64) и журналом изменений. Игра загружает снимок один раз, потом раз в 2 секунды забирает только новые изменения. Каждые 1000 изменений снимок пересобирается, журнал старше 20 000 изменений удаляется — отставшие игроки получают `reset` и перезагружают холст.
- Заряды считаются на сервере. Двойное списание при одновременных запросах исключено условным `UPDATE` по счётчику `placed`.
- Возврат Stars: `/refund <user_id> <charge_id>` от владельца бота.
