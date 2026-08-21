# douban-export-skill

把豆瓣的**书 / 影 / 音 / 游**标记导出成 Markdown、CSV 或 JSON。不需要登录，不需要 Cookie，不需要浏览器。

形态是一个 [Claude Code](https://claude.com/claude-code) skill，但脚本本身是纯 Python 标准库，单独当命令行工具用也可以。

## 致谢与来源

**本项目衍生自 [daymade/claude-code-skills](https://github.com/daymade/claude-code-skills) 的 `douban-skill`（MIT）。**

下面这些关键内容全部出自该项目，本仓库只是在其基础上做通用化改造：

- 用 Frodo API（豆瓣安卓 App 后端）绕开网页端 PoW 反爬的整体思路
- HMAC-SHA1 签名算法
- 限速参数（每页 1.5s、类目间 2s）
- `references/troubleshooting.md`：七种方案的完整失败记录

原作者的 `LICENSE` 版权声明已保留，见 [LICENSE](LICENSE)。

## 为什么不用网页抓取

豆瓣 2018 年关停了官方 API，也没有数据导出功能。网页端上了 Proof-of-Work 挑战，**所有 HTTP 抓取一律被挡**——Cookie、`curl`、`browser_cookie3`、Jina Reader 全部失败。唯一可行的是 Frodo API，即豆瓣安卓 App 自己用的后端。

详见 [references/troubleshooting.md](references/troubleshooting.md)。

## 安装

```bash
git clone https://github.com/<you>/douban-export-skill.git
```

作为 Claude Code skill 使用，放到 `~/.claude/skills/douban-export/` 即可（user 级，任何目录下都能触发）。

依赖：**Python 3.6+ 标准库，无第三方包**。

## 用法

```bash
# 导出到当前目录，Markdown 格式
python3 scripts/douban-export.py --user <你的豆瓣ID>

# 只要影视和书，输出 JSON
python3 scripts/douban-export.py --format json --type film-tv,book

# 指定目录
python3 scripts/douban-export.py --output-dir ~/notes/media
```

| 参数 | 默认 | 说明 |
|---|---|---|
| `--user` / `-u` | 配置文件 | 豆瓣 ID，也接受完整 profile URL |
| `--output-dir` / `-o` | **当前目录** | 不存在会自动创建 |
| `--format` / `-f` | **`md`** | `md` / `csv` / `json` |
| `--type` / `-t` | 全部四类 | 逗号分隔：`book,film-tv,music,game`（`movie`/`tv`/`film` 是 `film-tv` 的别名） |

### 豆瓣 ID 怎么给

优先级：`--user` > 环境变量 `$DOUBAN_USER` > 配置文件 `~/.config/douban-export/config.json`。

**首次成功导出后 ID 会存进配置文件，之后不用再传。** 配置文件刻意放在仓库之外——默认输出目录是当前工作目录，万一被 commit，不该把用户 ID 一起带走。

ID 在 profile URL 里：`douban.com/people/<这一段>/`。

## 输出

每种类型一个文件，**没有数据的类型不会生成空文件**：

```
books.md    film-tv.md    music.md    games.md
```

> ⚠️ 默认输出到**当前目录**。如果你在自己的仓库里跑，记得把上面这些文件名加进 `.gitignore`（本仓库的 `.gitignore` 里已经列好，可以直接抄）。

列：`title, card_subtitle, url, date, rating, status, comment, tags`，影视多一列 `subtype`。

- **`card_subtitle`** 是豆瓣自己拼好的一行元数据，四类通用：
  - 电影：`2019 / 美国 加拿大 / 剧情 惊悚 犯罪 / 托德·菲利普斯 / 华金·菲尼克斯 罗伯特·德尼罗`
  - 书：`[美] Robert C. Martin / 2020 / 人民邮电出版社`
- `status`：读过/在读/想读、看过/在看/想看、听过/在听/想听、玩过/在玩/想玩
- `rating`：★ 到 ★★★★★，没打分就是空
- **`subtype`**（仅 `film-tv.md`）：`电影` 或 `剧集`。豆瓣把电影、电视剧、动画番剧全归在同一个类目下，这一列是唯一能把它们分开的字段。其余三类的 `subtype` 只是把类目名回显一遍，没有信息量，所以不输出这一列
- `url` 是条目唯一标识，也是去重和回链的依据

每次运行**全量覆盖**，没有增量状态，重复跑是安全的。约 500 条数据跑完一分多钟。

## 相对上游的改动

| | 上游 `douban-skill` | 本项目 |
|---|---|---|
| 输出格式 | 仅 CSV | **Markdown（默认）/ CSV / JSON** |
| 字段 | 6 列 | **8 列**（影视 9 列），增加 `card_subtitle`、`tags`，影视另加 `subtype` |
| 输出目录 | `~/Downloads/douban-sync/<id>/` | **当前目录**，`--output-dir` 可改 |
| 文件名 | `书.csv` / `影视.csv` … | **`books.md` / `film-tv.md` …**（英文名） |
| 类别筛选 | 不支持，四类一起导 | **`--type` 可选子集**，支持别名 |
| 用户 ID | 每次传环境变量 | **配置文件记住，免重复输入** |
| 参数 | 环境变量 | 标准 CLI 参数（`argparse`） |
| RSS 增量同步 | 有 | 去掉了，见下 |

**为什么加 `card_subtitle` 和 `tags`**：上游丢掉了类型、年代、导演、作者、出版社和用户自己打的标签，而这些字段本来就在同一个 API 响应里，不额外花一分钱。少了它们，导出的表只能回答「我看过什么」，回答不了「我是什么样的观众」。

**为什么叫 `film-tv` 而不是 `movies`**：豆瓣 API 这个类目的参数名是 `movie`，但里面装的是电影、电视剧、动画番剧——叫 `movies` 会让人以为只有电影。对外统一用 `film-tv`，`movie`/`tv`/`film` 保留为 `--type` 的别名。

**为什么用英文文件名**：默认输出到当前目录，文件名会直接出现在别人的项目里和 `git status` 里。表格内容仍然全中文——该本地化的是数据，不是文件名。

**为什么去掉 RSS 增量同步**：RSS 只返回最近约 10 条且不支持分页，而全量导出 500 条也就一分多钟。增量在这个量级属于过度设计，还多一份要维护的代码和一个会不同步的状态。

## 已知限制

- **拿不到长评（长评）、读书笔记、广播**——`interests` 接口不返回这些
- **私密 profile 静默返回 0 条**，不会报错
- **依赖逆向凭证**：脚本里的 `apiKey` 和 HMAC secret 是从豆瓣安卓 APK 提取的公共凭证，所有 App 用户共用、不标识个人身份。豆瓣哪天更换它们，本工具会立刻失效
- **不要调低限速**（每页 1.5s、类目间 2s），这些值是实测调出来的

## 安全与隐私

脚本只向 `frodo.douban.com` 发请求，不使用也不存储任何个人凭证。唯一落盘的个人信息是豆瓣用户 ID，存在 `~/.config/douban-export/config.json`——而它本来就公开写在 profile URL 里。

## License

MIT。见 [LICENSE](LICENSE)，其中保留了原作者 daymade 的版权声明。
