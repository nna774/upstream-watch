# upstream-watch

上流 (BGP ピア・トランジット・出口の数ホップ) の状態を定期的に測り、**前回から変化していたら通知する**。

**監視対象のルータではなく、その配下の Linux ホストで動かす**。ルータ側に置くのは
読み取りスクリプト 1 本だけで、判定・state・通知は全部配下に寄せる。

## 何を見ているか

probe ごとに「上流の同一性を表すシグネチャ」を出し、前回との diff を通知する。
時刻・経過時間・RTT のような毎回変わる値はシグネチャに含めない。

### `10-edgeos-bgp` — ルータの BGP (ssh 経由)

`vtysh` と `ip route` の出力から、ピアの state・default の属性・**default に付いてくる community** を拾う。

**community が最重要。** 今の quagga は RFC 8326 GRACEFUL_SHUTDOWN (`65535:0`) を honor しないので、
上流が drain に入ったことを知る手段が community を見る以外に無い。付いていても無視して使い続ける。

出力例 (アドレスは RFC 5737 / private ASN に置き換えてある):

```
summary4 peer 169.254.1.2 as=65001 pfx=1
summary4 peer 169.254.1.6 as=65001 pfx=1
default4 path 169.254.1.2 attr rid=192.0.2.1 lp=100 weight=300 best=no
default4 path 169.254.1.2 comm 65001:10031
default4 path 169.254.1.2 comm 65001:11013
default4 path 169.254.1.2 comm 65535:0
default4 path 169.254.1.6 attr rid=192.0.2.3 lp=100 weight=500 best=yes
default4 path 169.254.1.6 comm 65001:10100
default4 path 169.254.1.6 comm 65001:11013
route4 default via 169.254.1.6 dev v6tun1 proto zebra
```

上流 A (`169.254.1.2`) に `65535:0` が付いていて、best は上流 B 側になっている。
つまり上流 A は drain 中で、使ってはいけないと言われている。

community は 1 要素 1 行で出すので、増減がそのまま diff の 1 行になる。
主系の上流が drain に入ると以下が届く:

```
+default4 path 169.254.1.6 comm 65535:0
```

default を 1 本も受けていないときは `default4 (none)` を出す。これも通知対象。

確立していないピアの state は `Idle` / `Connect` / `Active` を数秒で巡回するので、
数値 (PfxRcd) は `pfx=N`、`Idle (Admin)` は `admin-shutdown`、それ以外は一律 `down` に畳む。
畳まないと、上がる予定の無いピアが 1 本あるだけで実行ごとに通知が飛ぶ
(上がる予定の無いピアを設定に置いてあるだけで実際に毎回飛んだ)。

**ルータ上には何も置かない。** ER-X はファームウェア更新で `/config` 外のローカルパッチが
消える機体なので、監視の実体は配下のホストに寄せる。

### `20-mtr-upstream` — 出口の先頭数ホップ

どの上流に出ているかを `mtr` で見る。**既定では先頭 4 ホップだけ**比較する。
全長で比べると自分の上流より外側の ECMP と負荷分散で揺れ続けて誤報に埋もれる。

出力例:

```
mtr -4 8.8.8.8 hop 1 192.0.2.254          # 自ルータ
mtr -4 8.8.8.8 hop 2 192.0.2.3            # 上流 B
mtr -4 8.8.8.8 hop 3 198.51.100.1         # そのトランジット
mtr -4 8.8.8.8 hop 4 ???
mtr -4 203.0.113.10 hop 1 192.0.2.254
mtr -4 203.0.113.10 hop 2 192.0.2.3       # 上流 B
mtr -4 203.0.113.10 hop 3 192.0.2.4       # 専用線を持つ境界
mtr -4 203.0.113.10 hop 4 192.0.2.99      # 専用線の対向
```

ホップ数は**自分の上流構成が判別できる深さ**に合わせる。上の例では 4 で
専用線の対向まで届くので、冗長な 2 本のどちらに乗っているかが見える。

v4 と v6 は別の上流に乗りうるので必ず両方をターゲットに入れる。

ホップ列は無応答ホップ (`???`) で揺れるため、この probe は既定で
**変化を検知したら測り直し、再現したときだけ通知する** (`CONFIRM_PROBES`)。

### `30-ripe-upstream` — 外から見たトランジット (既定では入れない)

**上流 AS 自身が統計を公開している場合は不要なので、既定では入れていない。**
probe は `PROBE_DIR` に置いたものだけが動くので、install しなければ無効。

RIPEstat から、対象 AS の上流 AS 集合を取る。家のインフラを一切使わない視点なので、
上流側に統計が無い AS を相手にするときや、対向を信用せず独立に確認したいときに install する。

```
ripe neighbour left AS65001 Example Transit One
ripe neighbour left AS65002 Example Transit Two
ripe transit 192.0.2.0/24 AS65001 Example Transit One
ripe transit 192.0.2.0/24 AS65002 Example Transit Two
```

`neighbour` は asn-neighbours (更新は 1 日 1 回)、`transit` は bgp-state で観測された
AS パス上の 1 つ上流。**障害検知には使えない**、構造変化の検知専用。
install するなら `MIN_INTERVAL_RIPE_UPSTREAM` で叩く間隔を絞ること。

AS の holder 名もシグネチャに含めている。RIPE DB 側の改名でも 1 回通知が飛ぶ。

ルータ配下のホストで動かすと「外からの視点」という取り柄が消える (家が落ちている間は
probe ごと黙る)。install するなら家の外のホストに置く。

## 通知先はメール

**`sendmail` はローカルキューに入れた時点で成功を返す。** だから v4 も v6 も落ちている
最中に変化を検知しても、postfix が滞留させて復帰後に配送する。**変化を取りこぼさない。**
秘密情報も要らない。

webhook 系 (Slack / ntfy / Discord) はこの性質を持たない。送信に失敗したらその変化は消える。
しかも `hooks.slack.com` と `discord.com` は AAAA を持たない v4 オンリーなので、
**v4 が落ちる事故を通知したい用途では通知経路ごと一緒に死ぬ**。

Slack に出したいなら **Slack の「チャンネルにメールを送る」アドレスを `MAIL_TO` に足す**。
キューの恩恵を受けたまま Slack に出る。

```sh
MAIL_TO='"#channel-name (Slack)" <xxxx@your-team.slack.com>, root@example.net'
```

`NTFY_URL` / `SLACK_WEBHOOK_URL` は**メールが投函できなかったときだけ**試される補助経路。
送信は v6 を先に試し、駄目なら v4 に落とす。

**どの経路にも渡せなかった変化は state を更新せず、次回に持ち越して再試行する。**

## 依存

```sh
# Debian/Ubuntu
sudo apt-get install -y mtr-tiny curl
```

- `10-edgeos-bgp`: `ssh`
- `20-mtr-upstream`: `mtr`
- 通知: `sendmail` (postfix 等の MTA)
- `30-ripe-upstream`: `python3` (入れる場合のみ)
- Slack webhook に直接投げる場合のみ `jq`

## インストール (deploy 先の Linux マシンで)

root では動かさない。`mtr` は非特権 ICMP datagram socket を使うので `CAP_NET_RAW` も要らない
(Debian 12 で、`uid/gid 65534` + group 全クリアでも打てることを確認済み)。

```sh
# 0. 専用の非特権ユーザ
sudo useradd --system --no-create-home --shell /usr/sbin/nologin upstream-watch

# 1. 本体と probe
sudo install -m 0755 upstream-watch /usr/local/bin/upstream-watch
sudo mkdir -p /usr/local/libexec/upstream-watch
sudo install -m 0755 probes/10-edgeos-bgp probes/20-mtr-upstream \
  /usr/local/libexec/upstream-watch/

# 2. 設定
sudo mkdir -p /etc/upstream-watch
sudo install -m 0644 targets.conf.example /etc/upstream-watch/targets.conf
sudo install -m 0600 -o upstream-watch env.example /etc/upstream-watch/env
sudo $EDITOR /etc/upstream-watch/targets.conf
sudo $EDITOR /etc/upstream-watch/env

# state は専用ユーザが書く
sudo install -d -o upstream-watch -g upstream-watch -m 0755 /var/lib/upstream-watch

# 3. ルータへの ssh 鍵 (systemd で ProtectHome=true にしているため /etc 配下に置く)
sudo ssh-keygen -t ed25519 -N '' -f /etc/upstream-watch/id_ed25519
sudo chown upstream-watch /etc/upstream-watch/id_ed25519
sudo cat /etc/upstream-watch/id_ed25519.pub   # これをルータの authorized-key に入れる

# 4. systemd
sudo install -m 0644 systemd/upstream-watch.service /etc/systemd/system/
sudo install -m 0644 systemd/upstream-watch.timer   /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now upstream-watch.timer
```

### ルータ側 (EdgeOS) の設定

鍵に 1 コマンドだけを許す形にする。EdgeOS の都合でこの組み合わせしか成立しない。

- **`authorized_keys` の `command="..."` は使えない。** EdgeOS は設定値に `"` を書けない
  (`Cannot use the double quote (") character in a value string`)。クォート無しの
  `command=/path` は sshd が鍵行ごと拒否する (fail-closed)
- **operator level には shell が無い。** `Vyatta::Login::User` が
  `($level eq 'admin') ? '/bin/vbash' : '/usr/sbin/nologin'` としていて、operator は
  Web GUI 専用。ssh で入れないので operator + sudoers では読めない
- **`vtysh` は `exec sudo /opt/vyatta/sbin/ubnt_vtysh` のラッパー**なので sudo が要る

残る組み合わせは 2 つで、**失敗の向きが違う**。

| | shell の出し方 | 設定が失われたとき |
|---|---|---|
| **operator + `chsh`** (推奨) | `chsh -s /bin/vbash uwatch` | **fail closed**。shell が nologin に戻り監視が止まるだけ |
| admin level | `set ... level admin` | **fail open**。sshd_config が消えると鍵が admin shell になる |

ファーム更新は `/etc/ssh/sshd_config` を消すので、admin level だとその瞬間
`ForceCommand` の檻が外れて鍵が admin shell に化ける。operator のまま shell だけ与えれば
sudo も vyattacfg も持たないので、檻が外れても `sudoers` が許す 1 本しか実行できない。

推奨形では `ForceCommand` を `sudo` 経由にする (スクリプトが内部で `vtysh` =
`sudo ubnt_vtysh` を呼ぶため、スクリプト自体が root で走る必要がある)。

```sh
# 1. 読み取りスクリプト (/config 配下なのでファーム更新で消えない)
sudo install -o root -g root -m 0755 edgeos/upstream-watch-fetch \
  /config/scripts/upstream-watch-fetch
```

```
# 2. 専用ユーザと鍵 (configure モード)
set system login user uwatch full-name 'upstream-watch (read-only)'
set system login user uwatch level operator
set system login user uwatch authentication encrypted-password '<ランダムな $6$ ハッシュ>'
set system login user uwatch authentication public-keys monitor type ssh-ed25519
set system login user uwatch authentication public-keys monitor key <base64 部分>
set system login user uwatch authentication public-keys monitor options 'restrict'
commit ; save
```

```sh
# 2b. shell を与える (operator は nologin になるため)。sudoers は fetch 1 本だけ許可する
sudo chsh -s /bin/vbash uwatch
printf '%s\n' 'uwatch ALL=(root) NOPASSWD: /config/scripts/upstream-watch-fetch' \
  | sudo tee /etc/sudoers.d/upstream-watch >/dev/null
sudo chmod 0440 /etc/sudoers.d/upstream-watch
sudo visudo -cf /etc/sudoers.d/upstream-watch
```

`chsh` は vyatta の設定外なので、`system login` を触る commit で nologin に戻る。
戻れば監視が止まるだけで露出は増えない (fail closed)。itamae で再適用する。

```sh
# 3. ForceCommand で 1 コマンドに固定 (/etc/ssh/sshd_config は commit で
#    再生成されないので直接編集でよい。ファーム更新では消える)
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.uw-bak
sudo tee -a /etc/ssh/sshd_config >/dev/null <<'SSHD'

# upstream-watch の鍵を BGP 読み取り 1 本に限定する
Match User uwatch
    PasswordAuthentication no
    ForceCommand sudo -n /config/scripts/upstream-watch-fetch
SSHD
sudo /usr/sbin/sshd -t && sudo systemctl reload ssh
```

`ForceCommand` は sftp の subsystem 要求も上書きするので、この鍵では
BGP 状態の読み取り以外に何もできない。

### operator グループのパスワード認証が切れない

`disable-password-authentication` の実装はこうなっている。

```
update: sudo sed -i -e '/^PasswordAuthentication/s/yes/no/' /etc/ssh/sshd_config
```

`^` で行頭に固定されているため、`Match Group operator` ブロック内の
**インデントされた** `PasswordAuthentication yes` に当たらない。設定ノードでは消せないので、
切りたければ直接書き換える。

```sh
sudo sed -i -e '/^Match Group operator/,/^$/ s/^\( *\)PasswordAuthentication yes/\1PasswordAuthentication no/' \
  /etc/ssh/sshd_config
sudo /usr/sbin/sshd -t && sudo systemctl reload ssh
```

operator には shell が無いので実害は小さいが、設定の意図と実態が食い違っている。

## 動作確認

設定と state は専用ユーザが持っているので、手で回すときもそのユーザになる。

```sh
# 通知も state 更新もせずシグネチャだけ見る。まずこれで各 probe が通るか確かめる
sudo -u upstream-watch upstream-watch --dry-run

# probe 単位
sudo -u upstream-watch upstream-watch --dry-run --probe edgeos-bgp

sudo -u upstream-watch upstream-watch --list     # probe 一覧
sudo -u upstream-watch upstream-watch --show     # 保存済みシグネチャ
```

systemd 経由で走らせると journal に残る。

```sh
sudo systemctl start upstream-watch        # 1 回走らせる
journalctl -u upstream-watch -n 50 --no-pager
journalctl -u upstream-watch -f
```

初回はベースラインを作るだけで通知は出ない。

### vtysh のパーサを検証する

`vtysh` の出力は機種とバージョンで揺れる。実機の生出力を保存してパーサだけ回せる。

```sh
# ルータで
for c in 'show ip bgp summary' 'show bgp ipv6 summary' 'show ip bgp 0.0.0.0/0' 'show bgp ipv6 ::/0'; do
  vtysh -c "$c"; echo '===upstream-watch-split==='
done; ip -4 route show default; echo '===upstream-watch-split==='; ip -6 route show default

# 手元で
ROUTER_RAW_FILE=/path/to/captured.txt /usr/local/libexec/upstream-watch/10-edgeos-bgp
```

## 設定

`env` の全項目は `env.example` を見ること。よく触るもの:

| 変数 | 既定 | 意味 |
|---|---|---|
| `NTFY_URL` | — | 通知先 (推奨) |
| `SLACK_WEBHOOK_URL` | — | 通知先。v4 オンリーなので非推奨。`jq` が必要 |
| `ROUTER_SSH` | — | ルータへの ssh 先 |
| `UPSTREAM_MTR_HOPS` | `4` | mtr で比較するホップ数 |
| `CONFIRM_PROBES` | `mtr-upstream` | 変化時に測り直して再現確認する probe |
| `MIN_INTERVAL_<PROBE>` | `0` | probe ごとの最短実行間隔 (秒) |

probe 名はファイル名から先頭の数字を落としたもの (`30-ripe-upstream` → `ripe-upstream`)。
`MIN_INTERVAL_` に続けるときは大文字・`-` を `_` にする (`MIN_INTERVAL_RIPE_UPSTREAM`)。

## systemd unit の落とし穴

硬める設定とメール投函が衝突する。3 つとも実機で踏んだ。

| | |
|---|---|
| `postdrop` は setgid `postdrop` | `NoNewPrivileges=true` 下では setgid が効かない。`SupplementaryGroups=postdrop` で最初から渡す |
| `ProtectSystem=strict` | `/var/spool/postfix/maildrop` も読み取り専用になる。`ReadWritePaths` に足す |
| `Type=oneshot` の `TimeoutStartSec` | **既定が無限**。`sendmail` が詰まると unit が永久に `activating` のまま居座る。明示的に設定する |

## probe を足す

`/usr/local/libexec/upstream-watch/` に実行可能ファイルを置けば拾われる。規約:

- 標準出力に、前回と比較可能な安定したテキストを書く
- 終了コード 0 で成功。**非 0 なら測定失敗として扱い、state を更新せず通知もしない**
- 毎回変わる値 (時刻・経過時間・RTT・カウンタ) を出さない
- stdin は本体が `/dev/null` に固定する。`ssh` のように stdin を読むコマンドを使っても
  probe 一覧の残りを食われない

測定できなかったことと状態が変わったことを区別するのがこの規約の目的。

## 検知できないもの

**BGP が正常なのに転送できない障害は拾えない。** 見ているのは BGP の状態・属性と
出口の先頭数ホップで、パケットが実際に届くかは見ていない。

実例として、上流のセッションを落として戻した直後に、セッションは Established・default も
その上流を指している状態のまま、v4 が 23 秒間まったく通らなかったことがある。
このとき検知できたのはセッション状態の変化だけで、断そのものは見えていない。

それを拾いたいなら、疎通そのもの (1 秒間隔の ping など) を見る別の仕組みを併用する。

ポーリングなので、2 回の観測の間に落ちて戻った変化も見えない。

## 経路監視ツールとの違い

ターゲットまでの経路を全長で比較するツール (mtr や traceroute の結果を丸ごと diff するもの)
とは目的が違う。こちらが見るのは**上流の同一性**で、

- BGP が何を言っているか (state・属性・community)
- 出口の先頭数ホップ
- 外から見た上流 AS の集合

に絞る。全長で比較すると自分の上流より外側の揺れに埋もれて、
「上流が入れ替わった」という一番知りたい事実が見えなくなる。

## License

MIT (`LICENSE` を参照)。
