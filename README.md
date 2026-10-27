# bromelo × rayan4ik — TikTok checker

Group TikTok checker inspired by the workflow of link-based checker/downloader bots, implemented independently.

## Flow
1. User sends a public TikTok link in the group, or `/tt <link>`.
2. Worker first tries the public TikTok video page and parses `__UNIVERSAL_DATA_FOR_REHYDRATION__` / `SIGI_STATE`.
3. If TikTok does not return usable page data, it falls back to TikWM.
4. If TikWM is unavailable/rate-limited, it tries the secondary fallback API.
5. Bot sends the public video when a usable media URL is available and shows fresh public stats.

## Features
- Auto-detect TikTok links in groups
- `/tt <url>` and reply `/tt`
- Public stats: views, likes, comments, shares, saves, ER
- Refresh button
- Open TikTok button
- Modern TikTok SSR parser (statsV2 preferred)
- Short `vm.tiktok.com` / `vt.tiktok.com` support
- Provider fallback chain

## Deploy
```bash
npm install
npx wrangler secret put BOT_TOKEN
npx wrangler secret put WEBHOOK_SECRET
npx wrangler deploy
```

Then open:

`https://YOUR-WORKER.workers.dev/setup`

Expected response contains `"ok": true`.

## Telegram group setup
For automatic processing of ordinary links in a group, disable Group Privacy for the bot in BotFather and re-add the bot to the group if necessary.

## Notes
- Only public TikTok links are handled.
- External platforms may change their public pages or block datacenter traffic; the bot therefore has multiple providers.
- The project does not use or reverse-engineer reLynth's private APIs/code/branding.


## Media proxy fix
Telegram receives a signed `/media` URL from the Worker instead of a raw TikTok CDN URL. The Worker fetches the video with browser-like headers and forwards Range requests, which avoids `Bad Request: failed to get HTTP URL content` for CDN links that Telegram cannot fetch directly.


## DIRECT UPLOAD FIX

Эта версия не передаёт TikTok/CDN URL в Telegram `sendVideo`.
Worker сам скачивает MP4 и загружает реальные байты в Telegram через `multipart/form-data`.
Это исправляет `Bad Request: failed to get HTTP URL content`, когда Telegram не может сам открыть внешний CDN URL.

После замены файлов достаточно:

```cmd
npm install
npx wrangler deploy
```

Если Worker URL не менялся, webhook повторно ставить не обязательно. Для проверки `/setup` можно открыть ещё раз.

## FIX 403
Эта сборка больше не зависит от одного TikTok CDN URL.
Порядок получения MP4:
1. Все playAddr/downloadAddr/bitrate URL из TikTok.
2. Несколько HTTP-профилей без Range в первом запросе.
3. @satorufx/mediadownloader как независимый extractor fallback.
4. depay.cloud как дополнительный fallback.

После распаковки обязательно снова выполните `npm install`, потому что в этой версии добавлена npm-зависимость.

## Premium Emoji

В этой версии отчёт поддерживает Telegram Custom Emoji через `PREMIUM_EMOJIS_JSON`.
Если secret не задан, бот использует обычные emoji.

1. Отправь в Telegram сообщение с нужными Premium Emoji.
2. Ответь на это сообщение командой `/emojiids`.
3. Бот покажет `custom_emoji_id` каждого Premium Emoji.
4. Собери JSON по шаблону `PREMIUM_EMOJIS_JSON.example.json`.
5. Добавь его в Cloudflare:

```cmd
npx wrangler secret put PREMIUM_EMOJIS_JSON
```

Вставь JSON одной строкой, затем:

```cmd
npx wrangler deploy
```

Для отображения custom emoji у бота владелец бота должен иметь Telegram Premium (либо бот должен соответствовать другим условиям Telegram для custom emoji).

## Теневой бан

Строка `Теневой бан` показывается всегда.
Обычный публичный TikTok-ответ не предоставляет надёжный флаг «shadowban», поэтому бот не делает ложный вывод по низким просмотрам.
Если источник явно отдаёт recommendation-eligibility, используется этот сигнал. Иначе выводится `Не определяется публично`.

## STYLE V2 — reLynth-like layout

Эта сборка оформляет отчёт плотнее и ближе к референсу: заголовок «ВИДЕО • АНАЛИТИКА», автор + дата, blockquote описания, музыка, статистика, информация, качество, play_addr-блок, категории и нижний бренд-блок.

21 уникальный Premium Emoji ID пользователя уже встроен в `src/index.js` как `DEFAULT_PREMIUM_EMOJIS`. `PREMIUM_EMOJIS_JSON` теперь необязателен и нужен только если захочешь переназначить иконки.

### Теневой бан

Бот показывает:
- `Нет` — только если TikTok сам отдаёт явный recommendation-eligibility=true;
- `Да` — только если TikTok сам отдаёт recommendation-eligibility=false;
- `Неизвестно` — если публичный ответ TikTok не содержит такого флага.

Это специально сделано без ложного «Нет» по одним лишь просмотрам/лайкам.

## ENTITY_TEXT_INVALID FIX
- Внутри `<tg-emoji>` теперь всегда используется валидный стандартный emoji `⭐`.
- Custom Emoji визуально всё равно берётся из `emoji-id`.
- Если Telegram отклонит Premium Emoji entity, отправка автоматически повторяется с обычными emoji, чтобы проверка ролика не падала.

## Exact analytics enrichment (2026-10-22)

This build enriches the normal TikTok SSR data with the public no-key TikTok analytics endpoints documented by Void Studio (`v2.voidaep.dev`):

- `isProhibited` -> direct restriction/shadow-ban signal when available.
- `statusMsg` -> restriction reason when present.
- `vqScore` -> TikTok VQ/compression score.
- `download_count` -> TikTok download counter.
- `browserHD` / `phoneSD` -> separate quality profiles.

If the direct restriction flag is unavailable, the bot falls back to a clearly-labelled probability estimate based on public view/engagement/age signals. The estimate is not presented as an official TikTok decision.

The enrichment service is called server-to-server and requires no extra secret/API key. Its public documentation currently states a 60 req/min/IP limit.

## DOWNLOAD COUNT MOBILE FIX
If web/provider responses omit `download_count`, the checker now tries TikTok mobile aweme detail directly by video ID across multiple regional hosts before falling back to ClipX.


## Reliable TikTok download_count

The public TikTok web/official APIs often omit the device-download counter.
This build supports Scrape Creators as a dedicated source for `download_count`.

Add the key once in Cloudflare:

```cmd
npx wrangler secret put SCRAPE_CREATORS_API_KEY
```

Then redeploy:

```cmd
npx wrangler deploy
```

If the key is not configured, the bot keeps using the free fallbacks, but `Скачивания` may remain `—`.

## VQ SCORE FIX

If `SCRAPE_CREATORS_API_KEY` is configured, this build also reads TikTok's real `vq_score`
from the same Scrape Creators response. TikTok commonly stores the score inside
`aweme_detail.video.meta` (or related `video_extra`) as nested JSON text, so the checker
now parses those embedded JSON strings instead of only looking for a top-level `VQScore`.

## VQ FINAL V2
- VQ parser scans the raw Scrape Creators response before JSON parsing.
- Handles normal, escaped and double-escaped `vq_score` inside TikTok `video.meta`.
- Also reads `video.meta` / bitrate `video_extra` directly from other TikTok responses.
- Verified with dedicated VQ parsing tests.
