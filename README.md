# geocaching-cli

离线优先的个人 Geocaching 命令行工具。先导入 Groundspeak GPX / Pocket Query，在本地 SQLite 里 `search` / `show` / `export`；需要访问 geocaching.com 时再用独立的 `auth` / `live` 子命令（基于 [pycaching](https://github.com/tomasbedrich/pycaching)）。

官方 Groundspeak Partner API 不对个人 CLI 开放。本工具**不会**复用 c:geo 的 Android 模块。没有网络时，离线路径必须能独立工作。

## 安装

需要 Python 3.11+。

```bash
git clone https://github.com/zzzzzyc/geocaching-cli.git
cd geocaching-cli
python3 -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -e .

# 开发 / 跑测试
pip install -e ".[dev]"
python -m pytest
```

安装后提供两个入口：`gc` 与 `geocaching-cli`。

```bash
gc --help
gc --version
```

## 配置与密钥

**不要**把账号密码写进仓库、README、测试或 CI 日志。

推荐环境变量：

```bash
export GEOCACHING_USERNAME='你的用户名'
export GEOCACHING_PASSWORD='你的密码'
```

也可以在登录成功后由 `gc auth login` 写入本机、权限 0600 的凭证文件（已在 `.gitignore` 中忽略）：

- `$GEOCACHING_HOME/credentials.json`（默认 `~/.geocaching-cli/credentials.json`）
- `$GEOCACHING_HOME/session.json`（会话 cookie 缓存）

常用路径覆盖：

| 变量 | 作用 |
| --- | --- |
| `GEOCACHING_HOME` | 数据目录（库、凭证、会话） |
| `GEOCACHING_DB` | SQLite 文件路径 |
| `GEOCACHING_USERNAME` / `GEOCACHING_PASSWORD` | 登录 |
| `GEOCACHING_COOKIE` | 浏览器 `gspkauth` cookie（CAPTCHA 后备） |
| `GEOCACHING_CREDENTIALS` | 自定义凭证文件路径 |

全局选项 `--db PATH` 可临时指定数据库。`gc config` 只打印路径与“是否已配置”，不会打印密钥。

## 离线工作流

仓库自带合成夹具，可立刻试用（不是真实线上数据）：

```bash
gc import examples/sample.gpx examples/sample-wpts.gpx
gc list
gc search --type mystery
gc search --near '39.9042,116.4074' --radius 5
gc show GC1A2B3
gc export /tmp/beijing.gpx --type traditional --active
gc plan --start '39.90,116.40' --limit 3
gc stats
```

Pocket Query zip（含主 GPX 与 `*-wpts.gpx`）同样支持：

```bash
gc import ~/Downloads/my-pocket-query.zip
```

数据命令都提供 `--json`，方便脚本使用：

```bash
gc list --json
gc show GC1A2B3 --json
gc coord parse 'N 40 41.352 W 074 02.670' --json
```

## 坐标工具

```bash
gc coord parse 'N 39° 54.252 E 116° 24.444'
gc coord convert '39.9042,116.4074' --to dmm
gc coord project --from '39.9042,116.4074' --bearing 45 --distance 500
gc coord midpoint '39.90,116.40' '39.88,116.41'
gc coord distance '39.9042,116.4074' '39.8822,116.4066'
gc coord checksum 'N 39 54.252 E 116 24.444'
```

`checksum` 计算数字和、数字根与字母数，给谜题缓存做辅助，不保证解谜正确。

## 在线命令（可选）

```bash
gc auth login          # 读环境变量，或交互输入；成功后缓存会话
gc auth status --check # 可选：真的访问一次网站
gc live search --near '47.644,-122.119' --limit 10
gc live search GC1PAR2 --limit 8          # 以该缓存为中心搜索
gc live show GC1PAR2
gc live show GC1PAR2 --save               # 写入本地库
gc live logs GC1PAR2 --limit 10
gc live my-finds --limit 10
gc auth logout
```

程序化密码登录若返回 `CAPTCHA is required to login to the site.`（本仓库在无浏览器环境实测即为此错误），请在已经登录的浏览器里复制 `gspkauth` cookie，**只放在本机环境或 gitignore 文件中**：

```bash
export GEOCACHING_COOKIE='cookie 值'
gc auth login --cookie "$GEOCACHING_COOKIE"
```

`live` 默认限制条数并在请求之间停顿；不要对网站做批量锤击。

### 简短合规说明

`live` 走的是非官方网页接口（pycaching），不是 Groundspeak 合作伙伴 API。这可能违反网站服务条款并导致账号受限。本工具只适合你自己的个人账号、小流量使用；不要拿它做公共服务、批量爬取或验证码破解。

## 命令一览

| 命令 | 说明 |
| --- | --- |
| `gc import <gpx\|zip...>` | 导入 GPX / Pocket Query |
| `gc list` / `gc search` | 本地列表与筛选 |
| `gc show GCXXXX` | 本地详情 |
| `gc export <file>` | 导出 GPX 或 JSON |
| `gc plan` | 最近邻路线 |
| `gc stats` / `gc config` | 统计与路径 |
| `gc coord …` | 解析 / 转换 / 投影 / 中点 / 距离 / 校验和 |
| `gc auth login\|status\|logout` | 在线登录 |
| `gc live search\|show\|logs\|my-finds` | 在线查询 |

所有数据命令支持 `--json`。每个子命令的字段说明见 `gc <cmd> --help`。

## 开发

```
src/geocaching_cli/   Python 包
examples/             合成 GPX 夹具
tests/                解析 / 导入 / CLI / 在线 mock
```

测试不包含真实账号，也不访问 geocaching.com。
