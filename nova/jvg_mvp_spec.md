# jvg MVP spec — 零服務版開票派工（git ＋ 一個共享目錄）

寫於 2026-09-08 13:4x 台北，13:5x 依 Ray 逐條審查改五處（見第 9 節）。給要在另一台機器上把它蓋起來的 agent。**不裝 gitea、不裝 valkey、不開新 server。** 硬需求只有：git、python3、一個所有席位都寫得到的目錄、每席一個能「吃 prompt、在指定目錄工作、回成功／失敗」的執行端。搬家版全文（含現行系統對照）在 https://zenspam.github.io/family-pages/nova/jvg_port_spec.html ；本頁只寫 MVP。

## 0. 目標與不做

目標：一張活從開票到關票，全程機器可讀、可重放、可無人值守；驗收綠燈只准機器翻。
不做：網頁介面、權限系統、多隊列優先級、跨機器訊息服務、自動重構 prompt。這些等七條驗收綠了再談。

## 1. 目錄佈局（一個 git repo＝真相，一個共享目錄＝工作區）

```
repo/                      # git 追蹤，教練與席位都 clone
  tickets/0001.md          # 一張票一個檔，編號＝檔名
  events.log               # 只增不改，一行一筆 JSON；只有教練寫（hs sync 從 shared/events/ 收）
  progress.log             # 教練日誌，只增
  handoff.json             # 教練停在哪
  checks/0001.sh           # 驗收腳本，開票時寫，派工前封存 sha 進票
  tests/t1_e2e.sh … t7_three_traces.sh
  bin/hs                   # 教練 CLI（python，單檔）
  bin/worker               # 席位 daemon（python，單檔）
shared/                    # 共享目錄，不進 git
  queue/inbox/             # 待接：0001.json
  queue/taken/<seat>/      # 已接：0001.json（原子 mv 進來）
  queue/done/              # 交回
  queue/dead/              # 死信
  hb/<seat>                # 心跳：檔案 mtime
  events/<ts>-<seat>-<kind>-<task_id>.json   # 席位寫的事件，一檔一事；sync 收走後留著不刪
  runs/<YYYYMMDD>/<task_id>/   # 交付目錄：00_prompt.txt、REPORT.md、manifest.json、產物
```

## 2. 檔案格式

**票 `tickets/0001.md`**（frontmatter＋三段）：
```
---
id: 0001
feature: demo-echo
state: queued            # queued | taken | done | dead | closed
assignee:                # 席位名，認領時填
task_id: jvg.demo-echo.09081340
out_dir: shared/runs/20260908/jvg.demo-echo.09081340
check_sha: <sha256 of checks/0001.sh>
opened_at: 2026-09-08T13:40:00+08:00
passes: false            # 只准 hs check 翻
---
## 行為
在 out_dir 寫 REPORT.md，內容一行 ok。
## 驗收
checks/0001.sh（封存 sha 見上）
## 來源
Ray 2026-09-08 語音；只做這一件。
```

**隊列訊息 `queue/inbox/0001.json`**：`{"task_id","ticket":"0001","spec_path":"repo/tickets/0001.md","out_dir","timeout_s":1500,"expected":["REPORT.md"],"attempts":0,"enqueued_at"}`

**語義聲明**：本系統 at-least-once；重複執行由 `manifest.json` 擋（同 task_id 只會成功一次）；席位可能重複看到同一筆訊息，但不會重複交付。

**事件**：席位寫 `shared/events/<ts>-<seat>-<kind>-<task_id>.json`（一檔一事，先寫 `.tmp` 再 rename）；`hs sync` 按檔名排序收集、append 進 repo 的 `events.log`、commit；游標＝最後收走的檔名，存 `handoff.json` 的 `events_cursor`。`events.log` 一行：`{"ts":"…+08:00","kind":"taken|done|failed|dead|skipped|opened|checked|closed","task_id":"…","ticket":"0001","seat":"ds2","note":"…"}`

**心跳 `hb/<seat>`**：worker 每 15 秒 `touch`；mtime 超過 60 秒＝死。

**manifest.json**（席位成功時寫）：`{"task_id","executed_by":"<seat>","started","finished","outputs":[…],"rc":0}`

**驗收腳本 `checks/0001.sh`**：`set -e; cd "$OUT_DIR"; test -s REPORT.md; grep -q '^ok$' REPORT.md`。席位對 `checks/` 只讀（git 權限或目錄權限擋）；派工時票裡記 sha，check 前先比對，不合就紅。

## 3. 教練 CLI `bin/hs`（五個動詞，全部冪等、可放 cron）

| 動詞 | 做什麼 | 寫哪裡 |
|---|---|---|
| `hs open <feature> "<行為句>" --check "<shell>"` | 取下一個編號；寫票（state=queued）；寫 `checks/NNNN.sh`；算 sha 進票；events `opened`；commit | tickets／checks／events |
| `hs dispatch <NNNN> [--timeout 1500] [--expected REPORT.md,…] [--again]` | 票必須存在且 state∈{queued, dead(--again)}；建 out_dir＋`00_prompt.txt`（票全文＋執行紀律）；寫 `queue/inbox/.NNNN.json.tmp` 再 `os.rename` 成 `NNNN.json`（worker 永遠讀不到半個檔）；票 task_id／out_dir 回填；commit。**`--again` 鑄新 task_id（時戳不同）與新 out_dir**，舊 out_dir 留著驗屍；否則舊 manifest 會讓冪等鍵直接 skipped、永遠翻不了身 | shared／tickets |
| `hs sync` | 收 `shared/events/` 裡游標之後的檔，排序 append 進 `events.log`；把 taken／done／dead 寫回票的 state／assignee；游標寫 `handoff.json`；commit | events／tickets／handoff |
| `hs check <NNNN>` | 比對 check_sha；`OUT_DIR=<out_dir> bash checks/NNNN.sh`；rc=0 → passes=true、state=closed、events `checked`＋`closed`；rc≠0 → events `checked` 帶 rc，state 留 done；commit | tickets／events |
| `hs finish -m "<msg>"` | 三痕跡：progress.log 有新行、handoff.json 有更新、工作樹有變更；缺一拒；齊了 commit | git |

另兩個小工具：`hs note "<事>"`（progress.log 加一行）、`hs status`（印停在哪＋git log -1＋各 state 票數）。
cron：`*/10 * * * * hs sync && hs check --all-done`（把 state=done 且 passes=false 的全跑一遍）。

**信任邊界（直說）**：repo 的 git 寫權限只給教練；席位拿唯讀 clone（要改碼走 fork＋PR）。`passes` 只准 `hs check` 翻是靠這道權限擋的，不是靠約定；`hs verify` 是抓漏不是擋漏。席位對 `shared/` 全寫、對 `repo/` 只讀。

**執行紀律（附在 `00_prompt.txt` 尾）**：只做票的「行為」段；產物寫在 `$OUT_DIR`；一定要有 `REPORT.md`（做了什麼、怎麼驗、沒做什麼）；不改 `repo/checks/`；不碰 out_dir 以外的檔。

## 4. 席位 daemon `bin/worker <seat>`（約 200 行）

```
loop 每 5 秒:
  touch hb/<seat>
  for f in sorted(queue/inbox/*.json):
      try: os.rename(f, queue/taken/<seat>/f)      # 原子；輸了就 continue
      except FileNotFoundError: continue
      msg = load
      if exists(out_dir/manifest.json): event skipped; mv → done/; continue      # 冪等
      event taken
      rc, stdout = executor.run(prompt=read(out_dir/00_prompt.txt), cwd=out_dir, timeout=msg.timeout_s)
      ok = rc==0 and all(size(out_dir/e)>0 for e in msg.expected)
      if ok: write manifest.json; event done; mv → done/
      else:
          msg.attempts += 1; event failed(attempts, 摘 stdout 尾 500 字)
          if msg.attempts >= 2: event dead; mv → dead/
          else: mv → inbox/（重排隊尾：改檔名時間戳）
孤兒撿回（啟動時＋之後每 12 輪＝約 60 秒一次）: taken/<其他席>/ 裡 mtime 超過 2×timeout 且該席 hb 死 → mv 回 inbox/（events 記 reclaimed）
```

**執行端介面 `executor.run(prompt, cwd, timeout) -> (rc, stdout)`**，三個現成實作擇一：
- `pi`：`subprocess.run(["pi","-p","--approve", prompt], cwd=cwd, timeout=timeout)`（非互動模式不問 project trust，`--approve` 放行 `.pi/` 資源）。
- `claude`：`subprocess.run(["claude","-p","--dangerously-skip-permissions", prompt], cwd=cwd, timeout=timeout)`。
- `http`：POST 任何 `/control` 類端點。
逾時＝rc 124（跟執行端自己的錯誤分開記）。

## 5. 七條驗收（`tests/`，先寫先紅，做一步綠一條）

| # | 名 | 步驟 | 綠的定義 |
|---|---|---|---|
| t1 | e2e | open → dispatch → 起一個 worker（executor=echo 假執行端，直接寫 REPORT.md）→ 30 秒內 events 有 taken 再 done → sync → check → finish | 票 state=closed、passes=true、git log 多一筆、events 有 opened/taken/done/checked/closed |
| t2 | 沒票不派 | `hs dispatch 9999` | 非零退出；inbox 空 |
| t3 | 死信 | expected 指定 `NOPE.md`；worker 跑 | 兩筆 failed 後一筆 dead；票 state=dead；passes=false；檔在 dead/ |
| t4 | 人手翻不算 | 手改票 passes=true；`hs verify` | 非零退出並點名該票（verify＝重跑 check 對照） |
| t5 | 考卷不可改 | 派工後改 `checks/NNNN.sh` 一個字元；`hs check` | 紅，理由 `check_sha mismatch` |
| t6 | 冪等 | 同一筆 inbox 訊息投兩次 | 第二次 events 為 skipped，manifest 不變、REPORT 不重寫 |
| t7 | 三痕跡 | 不 `hs note` 直接 `hs finish` | 拒絕、無 commit |
| t8（加碼） | 孤兒撿回 | 起 worker A 接單後 kill -9、hb 過期；起 worker B | B 撿回並完成，events 有 `taken seat=B` |

## 6. 建置順序（每步結束都跑一次 tests/，只准變綠不准變紅）

1. 偵察一小時 → `recon.md`：OS、python、git、共享目錄路徑與權限、執行端指令、席位數、票務系統有沒有。
2. 寫 tests/ 七支，全紅。
3. `bin/hs` open／dispatch／sync／check／finish → t2、t7 綠。
4. `bin/worker`＋echo 假執行端 → t1、t3、t6 綠。
5. check_sha 與 verify → t4、t5 綠。
6. 換真執行端（pi 或 claude），第二個席位 → t8 綠，t1 用真模型重跑一次。
7. cron 兩行（sync、check --all-done）；`hs status` 印各 state 票數。
8. 之後才談：接現成票務系統當載體（票的 frontmatter 對到 label／assignee／留言）、看守（taken 超過 30 分鐘沒事件就喊）、票表網頁。

## 7. 坑（現行系統流過血的）

- 席位回「做好了」但沒檔＝假交付：成敗只看 expected 與 manifest，不看 stdout。
- 判活看 hb 檔 mtime，不用 `kill -0`（殭屍會回成功）。
- 派工端寫了 inbox 不等於有人接：`hs sync` 要在 cron 上，不能靠人記得。
- 逾時 rc 跟執行端錯誤分開記；先看 out_dir mtime 再定罪。
- 看守第一版一定誤報：起 daemon 前先乾跑數一次。

## 8. 要 Ray 給的四樣

進那台機器的方式｜席位用哪個執行端｜票放純 git 還是接現成票務系統｜共享目錄路徑。

## 9. 審查紀錄（Ray 2026-09-08 13:46 逐條對照，五處全收）

1. events.log 寫入者矛盾 → 席位寫 `shared/events/` 一檔一事，`hs sync` 收集排序 append 並 commit；游標存 `handoff.json`。
2. dispatch 寫 inbox 不原子 → 先寫 `.tmp` 再 rename。
3. `--again` 與冪等鍵碰撞 → 重派鑄新 task_id＋新 out_dir。
4. 孤兒撿回只在啟動 → 每 12 輪撿一次。
5. 信任邊界沒講破 → 明寫 git 寫權只給教練、席位唯讀 clone、verify 是抓漏不是擋漏。
另補 at-least-once 語義聲明一句。
