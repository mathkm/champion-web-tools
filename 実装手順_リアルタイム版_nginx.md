# 実装手順書：リアルタイム版（nginx 上の CSV をポーリングで読み続ける）

## 0. この手順書について

既存のドラッグ&ドロップ版とは**完全に別バージョン**として、
**nginx が配信する `data/` 配下の CSV を定期ポーリングで読み込み続け、準リアルタイムに可視化する**版を新規に作る。
Docker（nginx コンテナ）も新規に用意する。

- 実装担当：Sonnet
- **既存の `C:\dev\ampereVisualizer\index.html`（D&D版）は一切変更しない。** 新版は別ディレクトリに作る。
- 可視化・パース・統計の**下流ロジックは D&D版を踏襲**する（新ペイロード/旧ペイロード両対応、チャンネル別 `series`、統計、期間選択、グラフ）。**変わるのはデータの入力経路のみ**（ファイル選択/D&D → サーバーのポーリング）。

### 新規作成するディレクトリ・ファイル

```
C:\dev\ampereVisualizer\realtime\
├── docker-compose.yml          （新規）
├── nginx\
│   └── default.conf            （新規：autoindex JSON 設定）
├── html\
│   └── index.html              （新規：ポーリング版。既存index.htmlをベースに改変）
└── data\                       （新規：空でよい。収集サーバーがCSVを書く先。nginxが配信）
```

> `realtime\data\` は、収集サーバー（MQTT→CSV）が書き込む先。動作確認用に既存 `..\data\*.csv` を数個コピーしてよい。

---

## 1. 設計判断（既定値。UI/定数で変更可能にする）

| 論点 | 既定 | 備考 |
|---|---|---|
| ポーリング間隔 | **10 秒**（定数 `POLL_INTERVAL_MS = 10000`） | UIの「更新間隔」セレクトでも変更可にする。 |
| 増加中ファイルの読み方 | **Range による差分取得**（バイトオフセット管理） | 既読サイズを覚え、`Range: bytes=<offset>-` で追記分だけ取得。二重計上を構造的に防ぐ。 |
| 新規/未読ファイル | 初回は**全体取得**（offset 0 から） | 以降は差分のみ。 |
| ライブ追従 | **ON**（チェックボックス、既定ON） | ON: 更新のたび最新を含む全期間表示。OFF: ユーザーが選んだ期間を維持。 |
| 一覧の取得 | nginx `autoindex_format json` の JSON を fetch | ファイル名・サイズ・mtime が得られる。 |

---

## 2. docker-compose.yml（新規・全文）

`C:\dev\ampereVisualizer\realtime\docker-compose.yml`：

```yaml
services:
  web:
    image: nginx:1.27-alpine
    container_name: ampere-realtime-web
    ports:
      - "8080:80"          # ホスト8080 → コンテナ80。必要なら変更。
    volumes:
      - ./html:/usr/share/nginx/html:ro
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
      - ./data:/usr/share/nginx/html/data:ro   # 収集サーバーの書き込み先を read-only 配信
    restart: unless-stopped
```

> - `data` を `:ro`（read-only）でマウント。nginx は読むだけ。書き込みは収集サーバー（ホスト側 `./data`）が行う。
> - 収集サーバーを同じ compose に同居させる場合は別 service を追加し、同じ `./data` を（書き込み可で）マウントする。本手順書では nginx のみを対象とする。

---

## 3. nginx 設定（新規・全文）

`C:\dev\ampereVisualizer\realtime\nginx\default.conf`：

```nginx
server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;
    charset utf-8;

    # トップ：ポーリング版 index.html
    location = / {
        try_files /index.html =404;
    }

    # データ配信：ディレクトリ一覧を JSON で返す + キャッシュ無効 + Range 対応
    location /data/ {
        autoindex on;
        autoindex_format json;          # /data/ を fetch すると [{name,type,mtime,size}, ...] が返る
        add_header Cache-Control "no-cache";
        expires -1;
        # Range(Accept-Ranges: bytes) は nginx の静的配信で既定有効
    }
}
```

> - `/data/`（末尾スラッシュ付き）への GET が JSON 配列を返す。各要素は `{"name":"...","type":"file","mtime":"...","size":12345}`。
> - `autoindex_format json` は nginx 1.7.9+。`nginx:1.27-alpine` で利用可。
> - `Cache-Control: no-cache` で一覧・増加中ファイルの陳腐化を防ぐ（フロントも `cache: 'no-store'` を併用）。

---

## 4. フロント（ポーリング版 index.html）

### 4.1 ベースと方針

- `C:\dev\ampereVisualizer\index.html`（D&D版）を `realtime\html\index.html` に**コピーして**から、以下を改変する。
- **残すもの**：`.app-title`、統計サマリ/期間選択パネル、時系列グラフパネル、日時パーサ、`series={ch0,ch1}` モデル、ダウンサンプル、`computeChannelStats`、`renderChart`、`setRange`/期間連動、`NAME_COLLATOR`、`renderFileInfo`（ファイル一覧テーブル）。
- **削除するもの**：ファイル選択ボタン `#selectBtn`、`#fileInput`、案内文、`#dropOverlay`（HTML/CSS/JS）、ページ全体D&Dの配線、`parseCsvFile`（File/Papa.parse(file) 版）、`handleFiles`、`clearAll` の D&D 依存部分。
- **追加するもの**：データソース状態パネル、ポーリング機構、テキスト/バイト列からの CSV パース、差分取り込み、ライブ追従。

### 4.2 行分類ロジックの共通化（リファクタ）

D&D版 `parseCsvFile` 内にある「1行→ch0/ch1 振り分け」を関数に切り出し、ポーリング版でも再利用する。

```js
// 1行(row = [受信日時, トピック, ペイロードJSON文字列])を分類して out.ch0/out.ch1 に push。
// 取り込めたら true、対象外なら false。
function classifyRow(row, out) {
  if (!row || row.length < 3) return false;
  const t = parseDateTime(row[0]);
  if (!t) return false; // ヘッダ/不正行

  let payload;
  try { payload = JSON.parse(row[2]); } catch (e) { return false; }

  let used = false;
  if ('ch0_A' in payload || 'ch1_A' in payload) {
    const ch0 = Number(payload.ch0_A);
    const ch1 = Number(payload.ch1_A);
    if (Number.isFinite(ch0)) { out.ch0.push({ t: t, v: ch0 }); used = true; }
    if (Number.isFinite(ch1)) { out.ch1.push({ t: t, v: ch1 }); used = true; }
  } else if ('sensor_type' in payload && 'value' in payload) {
    if (String(payload.sensor_type) === CURRENT_SENSOR_TYPE) {
      const ch = SENSOR_NO_TO_CH[Number(payload.sensor_no)];
      const v = Number(payload.value);
      if (ch && Number.isFinite(v)) { out[ch].push({ t: t, v: v }); used = true; }
    }
  }
  return used;
}

// CSVテキストを分類。{ch0, ch1, used, skipped} を返す。
function classifyCsvText(text) {
  const out = { ch0: [], ch1: [] };
  let used = 0, skipped = 0;
  const parsed = Papa.parse(text, { header: false, skipEmptyLines: true });
  for (const row of parsed.data) {
    if (classifyRow(row, out)) used++;
    else skipped++;
  }
  return { ch0: out.ch0, ch1: out.ch1, used: used, skipped: skipped };
}
```

### 4.3 状態とDOM参照

D&D版の `loadedFiles` / `totalSkipped` を、ファイル別状態マップに置き換える。

```js
  const DATA_BASE = 'data/';            // nginx が配信するデータディレクトリ（同一オリジン）
  let POLL_INTERVAL_MS = 10000;         // 更新間隔（UIで変更可）

  let series = { ch0: [], ch1: [] };
  // ファイル名 → { offset:バイト既読量, rowCount:採用点数, skipped:スキップ数 }
  let fileState = {};
  let totalSkipped = 0;
  let liveFollow = true;
  let pollTimer = null;
  let polling = false;  // 多重実行防止
```

DOM参照：`selectBtn` / `fileInput` / `dropOverlay` を削除し、状態パネル用の参照（`statusText`, `lastUpdated`, `liveFollowChk`, `intervalSel` など、4.6のHTMLに合わせる）を追加。`fileSummary` / `badgeFiles` / `badgeRows` / `badgeSkipped` / `fileInfo` / `clearBtn` は残す。

### 4.4 ポーリング機構

```js
  // ---- 一覧取得（autoindex JSON） ----
  async function fetchListing() {
    const res = await fetch(DATA_BASE + '?_=' + Date.now(), { cache: 'no-store' });
    if (!res.ok) throw new Error('listing HTTP ' + res.status);
    const arr = await res.json();
    return arr.filter(e => e.type === 'file' && /\.csv$/i.test(e.name));
  }

  // ---- 1ファイルの差分（fromByte以降）を取得して取り込む ----
  async function ingestChunk(name, fromByte) {
    const url = DATA_BASE + encodeURIComponent(name);
    const opt = { cache: 'no-store' };
    if (fromByte > 0) opt.headers = { 'Range': 'bytes=' + fromByte + '-' };
    const res = await fetch(url, opt);
    if (res.status === 416) return 0;              // これ以上の範囲なし（新規データなし）
    if (!res.ok && res.status !== 206) throw new Error('chunk HTTP ' + res.status);

    const buf = new Uint8Array(await res.arrayBuffer());
    if (buf.length === 0) return 0;

    // バイト列の「最後の改行(0x0A)」までを完全行として扱い、末尾の未完行は次回に持ち越す。
    let lastNl = -1;
    for (let i = buf.length - 1; i >= 0; i--) { if (buf[i] === 0x0A) { lastNl = i; break; } }
    if (lastNl < 0) return 0;                       // まだ1行も完結していない

    const complete = buf.subarray(0, lastNl + 1);
    const text = new TextDecoder('utf-8').decode(complete);
    const r = classifyCsvText(text);

    series.ch0 = series.ch0.concat(r.ch0);
    series.ch1 = series.ch1.concat(r.ch1);

    const st = fileState[name] || { offset: 0, rowCount: 0, skipped: 0 };
    st.offset = fromByte + (lastNl + 1);           // バイト単位で前進
    st.rowCount += r.used;
    st.skipped += r.skipped;
    fileState[name] = st;
    totalSkipped += r.skipped;

    return r.used;
  }

  // ---- 1回のポーリング ----
  async function poll() {
    if (polling) return;
    polling = true;
    try {
      const files = await fetchListing();
      files.sort((a, b) => NAME_COLLATOR.compare(a.name, b.name)); // 古い順に取り込む
      let added = 0;
      for (const f of files) {
        const seen = fileState[f.name] ? fileState[f.name].offset : 0;
        if (f.size > seen) {
          added += await ingestChunk(f.name, seen);
        } else if (!fileState[f.name]) {
          fileState[f.name] = { offset: f.size, rowCount: 0, skipped: 0 }; // 空ファイル等
        }
      }
      if (added > 0) {
        series.ch0.sort((a, b) => a.t - b.t);
        series.ch1.sort((a, b) => a.t - b.t);
        refreshView();
      }
      renderFileInfoLive(files.length);
      setStatus('ok');
    } catch (e) {
      setStatus('error', e);
    } finally {
      polling = false;
      scheduleNext();
    }
  }

  function scheduleNext() {
    if (pollTimer) clearTimeout(pollTimer);
    pollTimer = setTimeout(poll, POLL_INTERVAL_MS);
  }
```

> - `setTimeout` で「1回終わってから次を予約」（`setInterval` にしない＝処理が間隔を超えても多重化しない）。
> - 取得失敗時も既存データは保持し、次回リトライ（`finally` で必ず再スケジュール）。

### 4.5 描画更新（ライブ追従）

D&D版 `renderAll` は毎回 `setRange(null,null,'reset')` で全期間にリセットするため、そのまま毎ポーリング呼ぶとユーザーのズームを壊す。ライブ追従の有無で分岐する。

```js
  function refreshView() {
    renderChart();
    if (liveFollow) {
      // 最新を含む全期間へ追従
      setRange(null, null, 'live');
    } else {
      // ユーザーが選択中の期間を維持（入力欄の値を再適用）
      const s = fromLocalInputValue(rangeStartInput.value);
      const e = fromLocalInputValue(rangeEndInput.value);
      setRange(s, e, 'live');
    }
  }
```

- `liveFollowChk` の change で `liveFollow` を更新し、ONにした瞬間は `refreshView()` を呼んで即追従。
- `intervalSel` の change で `POLL_INTERVAL_MS` を更新し、`scheduleNext()` で反映。

### 4.6 データソースパネル（HTML置換）

`データ読み込み` パネルの中身（`#selectBtn`〜案内文）を、状態表示に置換する。`#fileSummary` と `#fileInfo` は残す。

```html
  <section class="panel">
    <h2>データソース(自動更新)</h2>
    <div id="sourceControls">
      <span id="statusText" class="status">接続中…</span>
      <span id="lastUpdated" class="muted-inline"></span>
      <label><input type="checkbox" id="liveFollowChk" checked> ライブ追従</label>
      <label>更新間隔
        <select id="intervalSel">
          <option value="5000">5秒</option>
          <option value="10000" selected>10秒</option>
          <option value="30000">30秒</option>
          <option value="60000">60秒</option>
        </select>
      </label>
    </div>
    <div id="fileSummary" class="hidden">
      <span class="badge" id="badgeFiles">ファイル: 0</span>
      <span class="badge" id="badgeRows">読込点数: 0</span>
      <span class="badge warn hidden" id="badgeSkipped">スキップ: 0</span>
    </div>
    <div id="fileInfo"></div>
  </section>
```

> `#dropOverlay` の `<div>` は削除（HTML・CSS・JS すべて）。`クリア` ボタンは任意。残すなら「`series`/`fileState`/`totalSkipped` を初期化して次回ポーリングで全再読込」に振る舞いを変える。

状態パネル用の最小CSS（既存 `<style>` に追記）：

```css
  #sourceControls {
    display: flex; flex-wrap: wrap; align-items: center; gap: 8px 16px; font-size: 13px;
  }
  #sourceControls .status { font-weight: 600; }
  #sourceControls .status.ok { color: #2e7d32; }
  #sourceControls .status.error { color: #b3261e; }
  #sourceControls .muted-inline { color: var(--muted); font-size: 12px; }
```

### 4.7 状態表示・一覧・起動

```js
  function setStatus(kind, err) {
    statusText.classList.remove('ok', 'error');
    if (kind === 'ok') {
      statusText.classList.add('ok');
      statusText.textContent = '接続OK';
      lastUpdated.textContent = '最終更新: ' + formatDate(new Date());
    } else {
      statusText.classList.add('error');
      statusText.textContent = '取得エラー（リトライします）';
      if (err) console.error('[poll]', err);
    }
  }

  // 一覧テーブル（fileState を降順自然ソートで表示）＋バッジ
  function renderFileInfoLive(fileCount) {
    const names = Object.keys(fileState);
    if (names.length === 0) { fileSummary.classList.add('hidden'); fileInfo.innerHTML = ''; return; }
    fileSummary.classList.remove('hidden');
    badgeFiles.textContent = `ファイル: ${names.length}`;
    badgeRows.textContent = `読込点数: ${(series.ch0.length + series.ch1.length).toLocaleString()}`;
    if (totalSkipped > 0) { badgeSkipped.classList.remove('hidden'); badgeSkipped.textContent = `スキップ: ${totalSkipped.toLocaleString()}`; }
    else badgeSkipped.classList.add('hidden');

    const sorted = names.slice().sort((a, b) => NAME_COLLATOR.compare(b, a)); // 降順
    let html = '<table><thead><tr><th>ファイル名</th><th>読込点数</th><th>スキップ</th></tr></thead><tbody>';
    for (const n of sorted) {
      const st = fileState[n];
      html += `<tr><td>${escapeHtml(n)}</td><td>${st.rowCount.toLocaleString()}</td><td>${st.skipped.toLocaleString()}</td></tr>`;
    }
    html += '</tbody></table>';
    fileInfo.innerHTML = html;
  }

  // イベント配線
  liveFollowChk.addEventListener('change', () => { liveFollow = liveFollowChk.checked; refreshView(); });
  intervalSel.addEventListener('change', () => { POLL_INTERVAL_MS = Number(intervalSel.value) || 10000; scheduleNext(); });

  // 起動：初期描画（空）→ 即1回ポーリング → 以降ループ
  renderAll();
  poll();
```

> `renderAll()` は既存のまま（空状態を描画）。以後の更新は `refreshView()` 経由。`renderAll` 内の `renderRssiChart` は既に無い前提（D&D版で撤去済み）。

---

## 5. 注意点・エッジケース

- **同一オリジン**：index.html と `data/` は同じ nginx（同ポート）配信のため CORS 不要。別ホスト配信にすると CORS 対応が必要になる（本構成では避ける）。
- **バイトオフセット厳守**：Range と autoindex の `size` は**バイト**。オフセット前進は必ずバイト単位で（`TextDecoder` は完全行のバイト列のみ渡す）。ヘッダ行（`受信日時…`＝多バイト）や BOM も、バイト前進なら安全に読み飛ばせる。
- **書き込み途中の行**：末尾の未完行は取り込まず持ち越し、次ポーリングで完結行として取得。二重取り込みは起きない（offset が完全行末までしか進まないため）。
- **ファイルローテーション**：新ファイルが出れば一覧に現れ、`offset=0` から取り込む。既存ファイルは以後 size 増加分のみ。
- **初回ロード**：全ファイルを 0 から読むため、履歴が多いと初回だけ重い（D&D版で全ファイル読むのと同等）。
- **キャッシュ**：フロントは `cache:'no-store'`、nginx は `Cache-Control: no-cache`。両方で陳腐化を防ぐ。
- **autoindex の公開**：`/data/` のファイル名が URL で見える。LAN内前提なら可。外部公開時は要検討。
- **時刻**：D&D版同様、時間軸は CSV 1列目 `受信日時` を使用（`sync_t` 等の話とは独立、表示用の受信時刻）。

---

## 6. 検証手順

前提：Docker Desktop 稼働。`realtime\` で作業。

1. **起動**：`docker compose up -d` → `docker compose ps` で web が Up。
2. **表示**：ブラウザで `http://localhost:8080/` → ヘッダ「電流情報可視化」、データソースパネルが「接続OK」。
3. **新規ファイル検出**：`..\data\` から CSV を1つ `realtime\data\` にコピー → **10秒以内**に一覧・グラフ・統計へ反映。
4. **追記の準リアルタイム反映**：稼働中のファイル末尾に行を追記（例：PowerShell で既存CSVの数行を `Add-Content` で追記）→ 10秒以内に新しい点が増える。
5. **Range 確認**：DevTools → Network で、増加中ファイルのリクエストが **206 Partial Content**（`Range` ヘッダ付き）になっていること。一覧 `/data/` が **200 で JSON** を返すこと。
6. **ライブ追従**：チェックONで最新に追従。OFFにしてグラフをズーム→更新が来てもズーム維持。ONに戻すと追従再開。
7. **更新間隔**：セレクトを5秒に→反映が速くなる。
8. **エラー耐性**：`docker compose stop web` → 「取得エラー（リトライします）」表示、既存グラフは保持。`start` で復帰。
9. コンソールエラーが無いこと。
10. **後片付け**：`docker compose down`。

---

## 7. 完了チェックリスト

- [ ] `realtime\docker-compose.yml`（nginx、data を :ro マウント、8080:80）
- [ ] `realtime\nginx\default.conf`（`autoindex_format json`、`/data/` に no-cache）
- [ ] `realtime\html\index.html`（D&D版ベース、入力経路をポーリングに置換）
- [ ] `realtime\data\`（存在。動作確認用CSVを数個配置可）
- [ ] D&D/ボタン/オーバーレイ関連（HTML/CSS/JS）を削除
- [ ] `classifyRow`/`classifyCsvText` に行分類を共通化
- [ ] ポーリング（`fetchListing`/`ingestChunk`/`poll`/`scheduleNext`）実装、Range 差分・バイトオフセット管理
- [ ] ライブ追従トグル・更新間隔セレクトが機能
- [ ] 一覧は `fileState` を降順自然ソート表示、`max-height:100px` スクロール踏襲
- [ ] 新規ファイル・追記が10秒以内に反映（206 Range を確認）
- [ ] 取得失敗時に既存データ保持＋自動リトライ
- [ ] 既存 `..\index.html`（D&D版）は無改変
- [ ] コンソールエラーなし
