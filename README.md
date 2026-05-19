# verge-watch

Live download monitor for Clash Verge. Shows PROXY vs DIRECT throughput plus the per-host culprits eating bandwidth in real time. Single-file Python, tkinter only.

Clash Verge 实时下载监视器：分别看 PROXY/DIRECT 下行速度与累计，再列出当下 >1 MB/s 的 host。单文件 Python，只依赖 tkinter。

## Display / 界面

```
┌──────────────────────────────────────┐
│ PROXY                    5.2 G       │
│ DIRECT   45.3 M/s        12.4 G      │
│ mirrors.ustc.edu.cn   30.1 M/s 800 M │  ← DIRECT color
│ steamcdn-a.akamaihd…  15.2 M/s 240 M │  ← DIRECT color
│ api.anthropic.com      2.3 M/s  45 M │  ← PROXY  color
└──────────────────────────────────────┘
```

A host row appears when its download rate crosses 1 MB/s and vanishes when it drops back. Row color is inherited from the matching summary row: proxy host → PROXY current color (grey / orange / red by speed), direct host → DIRECT green. The PROXY rate cell goes blank when the value is below 10 KB/s; cumulative keeps ticking either way.

host 明细行在下行 >1 MB/s 时出现，掉回阈值以下立即消失。颜色继承对应汇总行：走代理 host → PROXY 当前色（灰/橙/红），直连 host → DIRECT 绿。速度 <10 KB/s 时 PROXY 速率列留空，累计仍在统计。

Window auto-sizes both axes — 22 px per row at 14pt bold; width fits the longest visible hostname.

窗口宽高自适应：每行 22 px（14pt 粗体），宽度跟当前最长主机名走。

## How / 实现

- `GET /connections` via unix socket `/tmp/verge/verge-mihomo.sock`
- Aggregate by `metadata.host`; per-tick rate = Δdownload / Δt
- `chains` contains `DIRECT` → direct, otherwise → proxy
- Totals accumulate from script start, not from Clash boot

走 unix socket 调 mihomo 内核的 `/connections`；按 `metadata.host` 聚合算速率；`chains` 含 `DIRECT` 视为直连，否则代理；累计从脚本启动时刻算起。

## Install / 安装

```bash
install -m 0755 verge-watch ~/.local/bin/verge-watch
python3 -c "import tkinter"   # required; ships with most distros
```

Python 3 + tkinter. No pip dependencies.

依赖：Python 3 + tkinter。无 pip 包。

## Run / 运行

```bash
verge-watch &
```

GNOME autostart — drop the following at `~/.config/autostart/verge-watch.desktop`:

GNOME 开机自启 — 写入 `~/.config/autostart/verge-watch.desktop`：

```ini
[Desktop Entry]
Type=Application
Name=Verge Watch
Exec=/home/neo/.local/bin/verge-watch
Terminal=false
X-GNOME-Autostart-Delay=8
```

The 8-second delay lets `clash-verge-service` bring the unix socket up before the GUI connects.

延迟 8s 是为了等 `clash-verge-service` 把 socket 建好。

## Controls / 控制

| Input | Action |
|---|---|
| Left drag | Move window / 拖动 |
| Right click | Quit / 关闭 |

## Tunables / 配置

Constants live at the top of the script. 所有常量都在脚本顶部。

| Name | Default | Meaning / 含义 |
|---|---|---|
| `RATE_HIDE` | 10 KB/s | summary rate cell blanks below this / 汇总速率列留空阈值 |
| `ACTIVE_RATE` | 1 MB/s | per-host row appears above this / 明细出现阈值 |
| `HOST_TRUNC` | 24 | hostname max chars / 主机名最长字符数 |
| `MAX_ROWS` | 10 | max detail rows / 明细最多行 |
| `POLL_MS` | 1000 | tick interval (ms) / 刷新间隔 |
| `SOCK` | `/tmp/verge/verge-mihomo.sock` | controller socket / 控制器 socket |

PROXY row color thresholds (also applied to proxy host detail rows):

PROXY 配色阈值（同样作用于走代理的明细行）：

| Rate | Color |
|---|---|
| `> 500 KB/s` | red  `#ff5555` |
| `> 50 KB/s`  | orange `#ffaa55` |
| else         | grey `#888888` |

## Caveats / 已知限制

- Per-host cumulative under-reports when a TCP connection closes mid-download — the closed connection's bytes leave the snapshot.
- If your Clash Verge exposes a TCP external-controller instead of the unix socket, change `SOCK` and rework `http_get` to use `AF_INET`.

单 host 累计在 TCP 连接关闭后会漏掉部分字节（关闭的连接不再出现在快照里）；如果你的 Clash Verge 走 TCP 控制器而非 unix socket，改 `SOCK` 并把 `http_get` 换成 `AF_INET`。
