# 実装手順書：ヘッダ簡素化・ページ全体D&D・ファイル一覧高さ変更

## 0. この手順書について

UI を以下の3点で改善する。

- 対象ファイル：`C:\dev\ampereVisualizer\index.html`（**このファイルのみ**を修正）
- 実装担当：Sonnet
- 変更範囲：ヘッダ表示、CSV の読み込み UI（ドロップゾーン→ボタン＋ページ全体D&D）、ファイル一覧の最大高。
  **読込・パース・集計・グラフ・統計のロジックには手を入れない**（`handleFiles` 等の呼び出し口だけ差し替える）。

---

## 1. 変更要件

| # | 要件 |
|---|---|
| 1 | タイトルバー（現在の濃紺ヘッダ `ampereVisualizer`）を廃止し、上部に **`12px`・グレー文字**で **`電流情報可視化`** と表示する。 |
| 2 | CSV の**ドラッグ＆ドロップ専用エリア（`#dropzone`）を廃止**し、**ページ全体を D&D の判定対象**にする。元のエリアがあった場所には**ファイル選択ボタン**を置く。 |
| 3 | ファイル名一覧（`#fileInfo`）の**最大高を `100px`** に変更する（現在 150px）。 |

---

## 2. 現状（変更前）

- HTML：`<header><h1>ampereVisualizer</h1></header>`（濃紺バー）。
- HTML：`データ読み込み` パネル内に `#dropzone`（破線枠のクリック可能エリア。中に隠し `#fileInput`）。
- JS：`#dropzone` に対して dragenter/dragover/dragleave/drop と click を配線（`index.html` 内「4. ドロップゾーン UI」ブロック）。`#fileInput` の change で `handleFiles`。
- CSS：`header` / `header h1` / `#dropzone` / `#dropzone.dragover` / `#dropzone .hint` / `#fileInput{display:none}` / `#fileInfo{... max-height:150px ...}`。
- DOM参照：`const dropzone = document.getElementById('dropzone');`（他に `fileInput` 等）。
- `handleFiles(fileList)` は内部で `.csv` のみ抽出（`/\.csv$/i`）するため、非CSVのドロップは無害に無視される。

---

## 3. 具体的な修正手順

### 手順1：ヘッダを廃止し 12px グレーの見出しに置換（HTML）

`<body>` 直後の以下を、

```html
<header>
  <h1>ampereVisualizer</h1>
</header>
```

次に置換：

```html
<div class="app-title">電流情報可視化</div>
```

> ブラウザのタブ名 `<title>ampereVisualizer</title>`（`<head>` 内）は**変更しない**（画面上の“タイトルバー”ではないため要件対象外）。

### 手順2：ヘッダ用CSSの削除と見出しCSSの追加

`<style>` 内の以下2ルールを**削除**：

```css
  header {
    padding: 16px 24px;
    background: #20293a;
    color: #fff;
  }
  header h1 {
    margin: 0;
    font-size: 20px;
    font-weight: 600;
  }
```

代わりに次を追加（`body` ルールの直後あたり）：

```css
  .app-title {
    padding: 8px 20px;
    font-size: 12px;
    color: var(--muted); /* グレー文字 (#666) */
  }
```

### 手順3：ドロップゾーンをファイル選択ボタンに置換（HTML）

`データ読み込み` パネル内の以下（`#dropzone` ブロック）を、

```html
    <div id="dropzone">
      <div>CSVをここにドラッグ&amp;ドロップ</div>
      <div class="hint">またはクリックしてファイルを選択(複数選択可)</div>
      <input id="fileInput" type="file" accept=".csv" multiple>
    </div>
```

次に置換（ボタン＋隠しinput＋案内文）：

```html
    <button id="selectBtn" type="button">CSVファイルを選択(複数可)</button>
    <input id="fileInput" type="file" accept=".csv" multiple>
    <div class="hint">またはページのどこにでもCSVをドラッグ＆ドロップ</div>
```

> `#fileInput` は従来どおり `#fileInput{display:none}` で隠したまま。ボタン経由で `fileInput.click()` する。

### 手順4：ページ全体D&D用オーバーレイの追加（HTML）

ドラッグ中に「ページ全体が対象」だと分かる視覚フィードバックを出す。`</body>` の直前に追加：

```html
<div id="dropOverlay" class="hidden"><div class="drop-msg">CSVをドロップして読み込み</div></div>
```

### 手順5：ドロップゾーン系CSSの整理（CSS）

- `#dropzone` と `#dropzone.dragover` のルールを**削除**。
- `#dropzone .hint` を、単独クラス `.hint` に置換（案内文で使う）：

```css
  .hint {
    font-size: 13px;
    margin-top: 6px;
    color: var(--muted);
  }
```

- `#fileInput { display: none; }` は**残す**。
- オーバーレイ用CSSを追加：

```css
  #dropOverlay {
    position: fixed;
    inset: 0;
    background: rgba(61, 126, 224, 0.12);
    border: 3px dashed #3d7ee0;
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000;
    pointer-events: none; /* dragイベントを奪わない（ちらつき防止） */
  }
  #dropOverlay .drop-msg {
    font-size: 18px;
    color: #3d7ee0;
    background: #fff;
    padding: 12px 24px;
    border-radius: 8px;
    box-shadow: 0 2px 12px rgba(0,0,0,0.12);
  }
```

> `.hidden { display:none !important; }` は既存を流用（オーバーレイの表示/非表示に使う）。

### 手順6：DOM参照の差し替え（JS）

DOM参照ブロック（`const dropzone = document.getElementById('dropzone');` 付近）で、`dropzone` を廃し `selectBtn` と `dropOverlay` を追加：

```js
  const selectBtn = document.getElementById('selectBtn');
  const fileInput = document.getElementById('fileInput');
  const dropOverlay = document.getElementById('dropOverlay');
```

（`dropzone` の参照は削除。`fileInput` 等はそのまま。）

### 手順7：イベント配線の差し替え（JS）

「4. ドロップゾーン UI」ブロック（`['dragenter','dragover'].forEach(...)` から
`fileInput.addEventListener('change', ...)` と `clearBtn.addEventListener('click', clearAll);` まで）を、
**ボタン配線＋ページ全体D&D**に置換する。`clearBtn` の配線は残すこと。

```js
  // =========================================================
  // 4. ファイル選択ボタン & ページ全体ドラッグ&ドロップ
  // =========================================================
  selectBtn.addEventListener('click', () => fileInput.click());
  fileInput.addEventListener('change', (e) => {
    if (e.target.files && e.target.files.length) {
      handleFiles(e.target.files);
    }
    fileInput.value = ''; // 同じファイルを再選択できるようにする
  });
  clearBtn.addEventListener('click', clearAll);

  // ---- ページ全体のD&D ----
  // 子要素間の移動でdragleaveが発火してもちらつかないよう、enter/leaveを数える。
  let dragDepth = 0;

  function isFileDrag(e) {
    // ファイルのドラッグのみ反応（テキスト等の選択ドラッグは無視）
    const types = e.dataTransfer && e.dataTransfer.types;
    return !!types && Array.prototype.indexOf.call(types, 'Files') !== -1;
  }

  window.addEventListener('dragenter', (e) => {
    if (!isFileDrag(e)) return;
    e.preventDefault();
    dragDepth++;
    dropOverlay.classList.remove('hidden');
  });
  window.addEventListener('dragover', (e) => {
    if (!isFileDrag(e)) return;
    e.preventDefault();
    if (e.dataTransfer) e.dataTransfer.dropEffect = 'copy';
  });
  window.addEventListener('dragleave', (e) => {
    if (!isFileDrag(e)) return;
    e.preventDefault();
    dragDepth--;
    if (dragDepth <= 0) {
      dragDepth = 0;
      dropOverlay.classList.add('hidden');
    }
  });
  window.addEventListener('drop', (e) => {
    e.preventDefault();
    dragDepth = 0;
    dropOverlay.classList.add('hidden');
    const dt = e.dataTransfer;
    if (dt && dt.files && dt.files.length) {
      handleFiles(dt.files);
    }
  });
```

> ポイント：
> - `drop` は常に `preventDefault()`（ブラウザがファイルを開いてしまうのを防止）。ファイルがあるときだけ `handleFiles`。
> - `dragover` の `preventDefault()` が無いと `drop` が発火しないため必須。
> - `.csv` 以外を落としても `handleFiles` 側で無視される（既存仕様）。

### 手順8：ファイル一覧の最大高を100pxに（CSS）

`#fileInfo` の `max-height` を変更：

```css
  #fileInfo {
    margin-top: 14px;
    font-size: 13px;
    max-height: 100px;   /* 150px から変更 */
    overflow-y: auto;
  }
```

（`overflow-y:auto` と `thead th` の sticky 固定は既存のまま。）

---

## 4. 留意点・エッジケース

- `#dropzone` を参照していたコードが残っていないこと（削除漏れがあると `null` 参照でエラー）。
- ページ全体D&Dのため、**画面のどこにドロップしても**読み込まれる（要件どおり）。
- ドラッグをウィンドウ外へ抜けた／ドロップした際に、オーバーレイが確実に消えること（`dragDepth` のリセット）。
- ボタンは既存の汎用 `button` スタイルを継承（追加スタイルは不要。必要なら余白のみ調整可）。
- 空状態・クリア（`clearAll`）・バッジ表示など他挙動は不変。

---

## 5. 検証手順

`.claude/launch.json` のサーバでローカル起動し、ブラウザで確認する。

1. **ヘッダ**：濃紺バーが無く、上部に 12px・グレーで `電流情報可視化` と表示される。
2. **ボタン読込**：`CSVファイルを選択` ボタン→ファイル選択ダイアログ→複数CSV読込でグラフ・統計が出る。
3. **ページ全体D&D**：
   - ファイルをウィンドウ上にドラッグ開始 → 画面全体にオーバーレイ（破線＋「CSVをドロップして読み込み」）が出る。
   - パネル外の余白など**任意の位置**にドロップ → 読み込まれる。
   - ドロップ後／ウィンドウ外へドラッグを抜けた後、オーバーレイが消える。
   - パネルやテキスト上を横切ってもオーバーレイがちらつかない。
   - 非CSV（例：画像）をドロップしても読み込まれず、エラーも出ない。
4. **ファイル一覧**：多数読み込み時、一覧が **100px** で頭打ち＋縦スクロール。ヘッダ行が固定表示される。
5. グラフ・統計・期間選択に回帰がない。コンソールにエラーが出ない。

---

## 6. 完了チェックリスト

- [ ] 濃紺ヘッダ廃止、`.app-title`（12px・`var(--muted)`）で `電流情報可視化`
- [ ] `header` / `header h1` CSS 削除
- [ ] `#dropzone` 廃止 → `#selectBtn` ボタン＋隠し `#fileInput`＋案内文
- [ ] `#dropzone` 系CSS削除、`.hint` 化、`#fileInput{display:none}` 維持
- [ ] `#dropOverlay` 追加＋CSS（`pointer-events:none`）
- [ ] DOM参照から `dropzone` 削除、`selectBtn`/`dropOverlay` 追加
- [ ] ページ全体D&D（window の dragenter/over/leave/drop、`dragDepth` カウンタ）
- [ ] `selectBtn` クリック→`fileInput.click()`、change→`handleFiles`、`clearBtn`→`clearAll` 維持
- [ ] `#fileInfo` の `max-height:100px`
- [ ] 任意位置ドロップで読込・オーバーレイ消去・ちらつき無し・非CSV無害
- [ ] 他機能に回帰なし・コンソールエラーなし
