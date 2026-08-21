---
name: douban-export
description: >
  Export Douban (豆瓣) book/movie/music/game collections to Markdown, CSV or JSON.
  No login, no cookies — reads any public profile via the Frodo mobile API.
  Use when the user wants to export or update their Douban data, back up their
  书影音游 marks, or pull reading/watching history into notes for analysis.
  Triggers on: 豆瓣, douban, 导出豆瓣, 更新豆瓣数据, 观影记录, 读书记录, 书影音.
---

# Douban Export

Export Douban collections to local files. Douban shut down its official API in
2018 and there is no data export feature; this reads the Frodo API that the
Douban Android app itself uses.

## Usage

```bash
python3 scripts/douban-export.py [--user ID] [--output-dir DIR] [--format FMT] [--type TYPES]
```

| Flag | Default | Notes |
|---|---|---|
| `--user` / `-u` | config file | Douban ID or full profile URL |
| `--output-dir` / `-o` | **current directory** | Created if missing |
| `--format` / `-f` | **`md`** | `md`, `csv`, or `json` |
| `--type` / `-t` | all four | Comma-separated: `book,film-tv,music,game` (`movie`/`tv`/`film` alias to `film-tv`) |

**User ID resolution order:** `--user` > `$DOUBAN_USER` > `~/.config/douban-export/config.json`.
After a successful run the ID is saved to the config file, so later runs need no
argument. The config lives outside the repo on purpose — the default output
directory is the working directory, and a committed output must never carry the
user's ID with it.

**Finding the ID:** profile URL `douban.com/people/<ID>/` — the part after `/people/`.
A full URL is accepted and parsed.

## Output

One file per type, only for types that actually have items:

```
books.md    film-tv.md    music.md    games.md
```

Columns: `title, card_subtitle, url, date, rating, status, comment, tags` — plus `subtype` for film-tv.

- `card_subtitle` is Douban's own one-line metadata: `2019 / 美国 / 剧情 / 托德·菲利普斯 / 华金·菲尼克斯`
  for a film, `[美] Robert C. Martin / 2020 / 人民邮电出版社` for a book. It is the
  only metadata field all four types share.
- `subtype` (film-tv only) is `电影` or `剧集`. Douban files films, TV series and
  anime under one category; this is the only field separating them. The other three
  types just echo their own category name back, so the column is omitted there.
- `status` is the Chinese mark: 读过/在读/想读, 看过/在看/想看, 听过/在听/想听, 玩过/在玩/想玩
- `rating` is ★ to ★★★★★, empty when unrated
- Files are overwritten in full on every run. Safe to re-run; no incremental state.

## Workflow

1. Resolve the user ID (flag, env, or saved config — ask the user only if none resolve)
2. Run the script; a full export of ~500 items takes a little over a minute
3. Report per-file row counts from the console output

## Constraints

- **Never lower the rate limits** (1.5s per page, 2s per category). They are tuned,
  not arbitrary.
- **Do not attempt web scraping.** Douban serves Proof-of-Work challenges on web
  pages; cookies, `curl`, `browser_cookie3` and Jina Reader all fail. See
  [references/troubleshooting.md](references/troubleshooting.md) for the full log
  of seven tested approaches.
- Cannot export long reviews (长评), reading notes, or broadcasts (广播) — the
  `interests` endpoint does not return them.
- A private profile returns zero items silently rather than erroring.
