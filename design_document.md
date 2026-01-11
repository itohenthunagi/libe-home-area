# 詳細設計書：対応エリア確認システム

## 目次
1. [システムアーキテクチャ設計](#1-システムアーキテクチャ設計)
2. [データ構造とCSV解析仕様](#2-データ構造とcsv解析仕様)
3. [UI/UX設計](#3-uiux設計)
4. [主要機能のロジックフロー](#4-主要機能のロジックフロー)
5. [ファイル構成と実装計画](#5-ファイル構成と実装計画)
6. [エラーハンドリング設計](#6-エラーハンドリング設計)
7. [パフォーマンス最適化](#7-パフォーマンス最適化)
8. [デプロイメント設計](#8-デプロイメント設計)

---

## 1. システムアーキテクチャ設計

### 1.1 全体構成図

```
┌─────────────────────────────────────────────────────────┐
│                    ユーザー（ブラウザ）                    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐  │
│  │         対応エリア確認 UI (index.html)            │  │
│  │                                                   │  │
│  │  ┌─────────────┐  ┌─────────────────────────┐  │  │
│  │  │ 入力フォーム  │  │  結果表示エリア         │  │  │
│  │  │ - 都道府県    │  │  - カード一覧          │  │  │
│  │  │ - 市区町村    │  │  - ステータスバッジ     │  │  │
│  │  └─────────────┘  └─────────────────────────┘  │  │
│  └─────────────────────────────────────────────────┘  │
│                          ↓                              │
│  ┌─────────────────────────────────────────────────┐  │
│  │       JavaScript アプリケーションロジック          │  │
│  │                                                   │  │
│  │  ┌─────────┐  ┌─────────┐  ┌──────────────┐  │  │
│  │  │CSV解析   │  │都道府県  │  │サジェスト      │  │  │
│  │  │エンジン  │  │市区町村  │  │フィルタリング  │  │  │
│  │  │(PapaParse)│ │インデックス│ │エンジン        │  │  │
│  │  └─────────┘  └─────────┘  └──────────────┘  │  │
│  │                                                   │  │
│  │  ┌─────────────────────────────────────────┐  │  │
│  │  │        判定ロジック（コアエンジン）        │  │  │
│  │  │  - 完全一致検索                          │  │  │
│  │  │  - ヘッダ解析                            │  │  │
│  │  │  - 判定セル読み取り                       │  │  │
│  │  └─────────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────────┘  │
│                          ↓                              │
│                     fetch API                           │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│              Googleスプレッドシート                        │
│                                                         │
│  ┌─────────────────┐        ┌──────────────────┐      │
│  │ 非公開シート      │        │ 公開用シート       │      │
│  │ 「まとめ」        │  →→→  │ 「公開_まとめ」    │      │
│  │                  │  転記  │                  │      │
│  │ - 社内情報含む   │        │ - 公開情報のみ    │      │
│  └─────────────────┘        └──────────────────┘      │
│                                       ↓                 │
│                              「ウェブに公開（CSV）」      │
│                                       ↓                 │
│                     https://docs.google.com/            │
│                     spreadsheets/d/xxx/                 │
│                     export?format=csv&gid=xxx           │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                   GitHub Pages                          │
│                                                         │
│  https://[username].github.io/[repo-name]/              │
│                                                         │
│  - 静的ホスティング                                      │
│  - GASバナーなし                                         │
│  - HTTPS対応                                            │
└─────────────────────────────────────────────────────────┘
```

### 1.2 アーキテクチャ特性

| 特性 | 採用方式 | 理由 |
|------|---------|------|
| **フロントエンド** | 静的HTML/JS/CSS | GASバナー回避、シンプル、低コスト |
| **バックエンド** | なし | データ更新頻度が低い、リアルタイム性不要 |
| **データソース** | 公開CSV | ノーコードで更新可能、Googleスプレッドシート活用 |
| **ホスティング** | GitHub Pages | 無料、HTTPS対応、CI/CD統合可能 |
| **CSVパーサー** | PapaParse | 実績豊富、軽量、ブラウザ対応 |
| **状態管理** | Vanilla JS（グローバルオブジェクト） | シンプル、依存なし |

### 1.3 データフロー

```
1. ページロード
   ↓
2. CSV_URL から fetch でCSV取得
   ↓
3. PapaParse で2次元配列に変換
   ↓
4. インデックス構築
   - providers[] (2行目 D列以降)
   - categories[] (3行目 D列以降)
   - rowsByPref: Map<pref, Array<row>>
   ↓
5. 都道府県プルダウン生成
   ↓
6. ユーザー操作待ち

【都道府県選択時】
   ↓
7. 市区町村候補をフィルタリング
   ↓
8. datalist更新

【確認ボタン押下時】
   ↓
9. バリデーション
   - 都道府県選択チェック
   - 市区町村入力チェック
   - 候補リスト存在チェック
   ↓
10. 該当行検索（完全一致）
    - 0件: エラー表示
    - 1件: 判定処理へ
    - 2件以上: データ重複エラー
    ↓
11. 判定セル読み取り（D列以降）
    ↓
12. 結果オブジェクト生成
    {ok: true, pref, muni, items: [{provider, category, value}]}
    ↓
13. UI表示（カード形式）
```

---

## 2. データ構造とCSV解析仕様

### 2.1 CSVデータ構造（サンプル画像より）

```
行番号 | A列(都道府県) | B列(市区町村) | C列(ふりがな) | D列       | E列    | F列    | G列           | H列    | I列           |
-------|--------------|--------------|--------------|-----------|--------|--------|---------------|--------|---------------|
1      | # 全エリアの対応状況                                                                                                   |
2      |              |              |              | 工務店    | 工務店  | 工務店  | 工務店         | 職人    | 職人           |
3      | 都道府県      | 市区町村      | ふりがな      | 注文住宅  | 大規模  | 小規模  | オフィス・店舗 | 小規模  | オフィス・店舗 |
4      | 北海道       | 札幌市        | さっぽろし    |          | ○      | ○      | ○             |        |               |
5      | 北海道       | 函館市        | はこだてし    |          |        |        |               |        |               |
6      | 北海道       | 小樽市        | おたるし      |          | ○      | ○      | ○             |        |               |
7      | 北海道       | 旭川市        | あさひかわし  |          | 要相談  | 要相談  | 要相談         |        |               |
...
```

### 2.2 解析後のデータ構造

#### 2.2.1 グローバルデータオブジェクト

```javascript
const appData = {
  // 元データ（2次元配列）
  table: [],  // PapaParseの結果

  // インデックス化されたデータ
  providers: [],   // ['工務店', '工務店', '工務店', '工務店', '職人', '職人']
  categories: [],  // ['注文住宅', '大規模', '小規模', 'オフィス・店舗', '小規模', 'オフィス・店舗']

  // 都道府県別インデックス
  rowsByPref: new Map(),
  // Map<string, Array<{muni: string, kana: string, rowIndex: number, rowData: array}>>

  // 設定
  config: {
    CSV_URL: 'https://docs.google.com/spreadsheets/d/.../export?format=csv&gid=xxx',
    CONTACT_URL: 'https://example.com/contact',
    HEADER_PROVIDER_ROW: 2,
    HEADER_CATEGORY_ROW: 3,
    DATA_START_ROW: 4,
    FIRST_JUDGE_COL: 4,  // D列（0始まりで3）
    SUGGEST_LIMIT: 30
  }
};
```

#### 2.2.2 rowsByPref の構造例

```javascript
Map {
  '北海道' => [
    {
      muni: '札幌市',
      kana: 'さっぽろし',
      rowIndex: 4,
      rowData: ['北海道', '札幌市', 'さっぽろし', '', '○', '○', '○', '', '']
    },
    {
      muni: '函館市',
      kana: 'はこだてし',
      rowIndex: 5,
      rowData: ['北海道', '函館市', 'はこだてし', '', '', '', '', '', '']
    },
    ...
  ],
  '東京都' => [
    ...
  ]
}
```

### 2.3 CSV解析処理フロー

```javascript
/**
 * CSVを取得して解析、インデックス構築
 */
async function loadAndParseCSV() {
  try {
    // 1. CSV取得
    const response = await fetch(appData.config.CSV_URL);
    if (!response.ok) throw new Error('CSV取得失敗');
    const csvText = await response.text();

    // 2. PapaParseで解析
    const parsed = Papa.parse(csvText, {
      skipEmptyLines: false  // 空行も保持（行番号を正確に）
    });

    appData.table = parsed.data;

    // 3. ヘッダ読み取り
    const providerRow = appData.table[appData.config.HEADER_PROVIDER_ROW - 1];
    const categoryRow = appData.table[appData.config.HEADER_CATEGORY_ROW - 1];

    // D列以降（index 3以降）を取得
    appData.providers = providerRow.slice(appData.config.FIRST_JUDGE_COL - 1);
    appData.categories = categoryRow.slice(appData.config.FIRST_JUDGE_COL - 1);

    // 4. データ行のインデックス構築
    buildIndex();

    // 5. UI初期化
    initializeUI();

  } catch (error) {
    showError('データの読み込みに失敗しました（CSV公開URLを確認してください）');
    console.error(error);
  }
}

/**
 * インデックス構築（都道府県別Map）
 */
function buildIndex() {
  appData.rowsByPref = new Map();

  // DATA_START_ROW（4行目）以降を処理
  for (let i = appData.config.DATA_START_ROW - 1; i < appData.table.length; i++) {
    const row = appData.table[i];
    const pref = row[0]?.trim();
    const muni = row[1]?.trim();
    const kana = row[2]?.trim();

    // 空行スキップ
    if (!pref || !muni) continue;

    // Map初期化
    if (!appData.rowsByPref.has(pref)) {
      appData.rowsByPref.set(pref, []);
    }

    // 追加
    appData.rowsByPref.get(pref).push({
      muni,
      kana,
      rowIndex: i + 1,  // 1始まり行番号
      rowData: row
    });
  }
}
```

---

## 3. UI/UX設計

### 3.1 画面レイアウト（ワイヤーフレーム）

```
┌──────────────────────────────────────────────────────┐
│                                                      │
│                  対応エリア確認                       │
│                                                      │
├──────────────────────────────────────────────────────┤
│                                                      │
│  ┌─────────────────────────────────────────────┐  │
│  │ 都道府県を選択してください ▼                  │  │
│  └─────────────────────────────────────────────┘  │
│                                                      │
│  ┌─────────────────────────────────────────────┐  │
│  │ 市区町村を入力してください                    │  │
│  └─────────────────────────────────────────────┘  │
│                                                      │
│  ┌─────────────────────────────────────────────┐  │
│  │                  確認する                      │  │
│  └─────────────────────────────────────────────┘  │
│                                                      │
├──────────────────────────────────────────────────────┤
│                                                      │
│  【結果表示エリア】                                  │
│                                                      │
│  ○○県 ○○市                                       │
│                                                      │
│  ┌────────────────┐  ┌────────────────┐       │
│  │ 工務店             │  │ 工務店             │       │
│  │ 大規模リフォーム    │  │ 小規模リフォーム    │       │
│  │                    │  │                    │       │
│  │   ○ 対応可能      │  │  要相談            │       │
│  └────────────────┘  └────────────────┘       │
│                                                      │
│  ┌────────────────┐  ┌────────────────┐       │
│  │ 職人               │  │ 職人               │       │
│  │ 小規模             │  │ オフィス・店舗      │       │
│  │                    │  │                    │       │
│  │  対応エリア外      │  │  対応エリア外      │       │
│  └────────────────┘  └────────────────┘       │
│                                                      │
├──────────────────────────────────────────────────────┤
│                                                      │
│              お問い合わせはこちら →                  │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### 3.2 UIコンポーネント設計

#### 3.2.1 入力フォーム

```html
<!-- 都道府県プルダウン -->
<div class="form-group">
  <label for="pref-select">都道府県</label>
  <select id="pref-select" class="form-control">
    <option value="">選択してください</option>
    <!-- 動的生成 -->
  </select>
</div>

<!-- 市区町村入力（datalist） -->
<div class="form-group">
  <label for="muni-input">市区町村</label>
  <input
    type="text"
    id="muni-input"
    class="form-control"
    list="muni-list"
    placeholder="入力して候補から選択"
    autocomplete="off"
  />
  <datalist id="muni-list">
    <!-- 動的生成 -->
  </datalist>
</div>

<!-- 確認ボタン -->
<button id="check-btn" class="btn btn-primary">確認する</button>
```

#### 3.2.2 結果表示カード

```html
<!-- カードコンテナ -->
<div id="result-area" class="result-area" style="display:none;">
  <h2 id="result-title"></h2>
  <div id="result-cards" class="cards-grid">
    <!-- カードテンプレート -->
  </div>
</div>

<!-- カードテンプレート -->
<template id="card-template">
  <div class="service-card">
    <div class="card-header">
      <div class="provider-type"></div>
      <div class="category"></div>
    </div>
    <div class="card-body">
      <span class="status-badge"></span>
    </div>
  </div>
</template>
```

### 3.3 スタイル設計（CSS）

#### 3.3.1 カラーパレット

```css
:root {
  /* プライマリカラー */
  --primary-color: #4A90E2;
  --primary-hover: #357ABD;

  /* ステータスカラー */
  --status-available: #4CAF50;      /* 対応可能 (○) */
  --status-consult: #FF9800;        /* 要相談 */
  --status-partial: #FFC107;        /* 一部対応 */
  --status-unavailable: #9E9E9E;    /* 対応エリア外 */

  /* グレースケール */
  --gray-100: #F5F5F5;
  --gray-300: #E0E0E0;
  --gray-500: #9E9E9E;
  --gray-700: #616161;
  --gray-900: #212121;

  /* フォント */
  --font-family: 'Noto Sans JP', sans-serif;
  --font-size-base: 16px;
  --font-size-lg: 18px;
  --font-size-xl: 24px;

  /* スペーシング */
  --spacing-xs: 8px;
  --spacing-sm: 12px;
  --spacing-md: 16px;
  --spacing-lg: 24px;
  --spacing-xl: 32px;
}
```

#### 3.3.2 カードスタイル

```css
.service-card {
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
  padding: var(--spacing-md);
  transition: transform 0.2s, box-shadow 0.2s;
}

.service-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0,0,0,0.15);
}

.card-header {
  margin-bottom: var(--spacing-sm);
}

.provider-type {
  font-size: var(--font-size-lg);
  font-weight: bold;
  color: var(--gray-900);
}

.category {
  font-size: var(--font-size-base);
  color: var(--gray-700);
  margin-top: var(--spacing-xs);
}

/* ステータスバッジ */
.status-badge {
  display: inline-block;
  padding: 6px 12px;
  border-radius: 16px;
  font-size: 14px;
  font-weight: 500;
}

.status-badge.available {
  background-color: var(--status-available);
  color: white;
}

.status-badge.consult {
  background-color: var(--status-consult);
  color: white;
}

.status-badge.partial {
  background-color: var(--status-partial);
  color: white;
}

.status-badge.unavailable {
  background-color: var(--status-unavailable);
  color: white;
}
```

### 3.4 レスポンシブデザイン

```css
/* モバイルファースト */
.cards-grid {
  display: grid;
  grid-template-columns: 1fr;
  gap: var(--spacing-md);
}

/* タブレット（768px以上） */
@media (min-width: 768px) {
  .cards-grid {
    grid-template-columns: repeat(2, 1fr);
  }
}

/* デスクトップ（1024px以上） */
@media (min-width: 1024px) {
  .cards-grid {
    grid-template-columns: repeat(3, 1fr);
  }

  .container {
    max-width: 1200px;
    margin: 0 auto;
  }
}
```

---

## 4. 主要機能のロジックフロー

### 4.1 都道府県プルダウン生成

```javascript
/**
 * 都道府県リスト生成
 */
function buildPrefectureList() {
  const prefs = new Set();

  for (const [pref, _] of appData.rowsByPref) {
    if (pref) prefs.add(pref);
  }

  return Array.from(prefs);
}

/**
 * プルダウン初期化
 */
function initializePrefectureSelect() {
  const select = document.getElementById('pref-select');
  const prefs = buildPrefectureList();

  prefs.forEach(pref => {
    const option = document.createElement('option');
    option.value = pref;
    option.textContent = pref;
    select.appendChild(option);
  });
}
```

### 4.2 市区町村サジェスト

```javascript
/**
 * 市区町村候補取得（部分一致）
 */
function suggestMunicipalities(pref, query) {
  if (!pref) return [];

  const rows = appData.rowsByPref.get(pref) || [];
  const queryLower = query.toLowerCase();
  const results = [];
  const seen = new Set();  // 重複除外用

  for (const row of rows) {
    // すでに追加済みならスキップ
    if (seen.has(row.muni)) continue;

    // 部分一致チェック（市区町村名 or ふりがな）
    const muniMatch = row.muni.toLowerCase().includes(queryLower);
    const kanaMatch = row.kana.toLowerCase().includes(queryLower);

    if (muniMatch || kanaMatch) {
      results.push({
        muni: row.muni,
        kana: row.kana
      });
      seen.add(row.muni);

      // 上限チェック
      if (results.length >= appData.config.SUGGEST_LIMIT) break;
    }
  }

  return results;
}

/**
 * datalist更新（debounce付き）
 */
let debounceTimer = null;
function updateMunicipalitySuggestions(pref, query) {
  clearTimeout(debounceTimer);

  debounceTimer = setTimeout(() => {
    const suggestions = suggestMunicipalities(pref, query);
    const datalist = document.getElementById('muni-list');

    // クリア
    datalist.innerHTML = '';

    // 追加
    suggestions.forEach(item => {
      const option = document.createElement('option');
      option.value = item.muni;
      option.textContent = `${item.muni} (${item.kana})`;
      datalist.appendChild(option);
    });
  }, 200);  // 200ms debounce
}

/**
 * イベントリスナー設定
 */
document.getElementById('pref-select').addEventListener('change', (e) => {
  const pref = e.target.value;
  document.getElementById('muni-input').value = '';  // クリア

  if (pref) {
    // 全候補を表示（queryは空文字）
    updateMunicipalitySuggestions(pref, '');
  }
});

document.getElementById('muni-input').addEventListener('input', (e) => {
  const pref = document.getElementById('pref-select').value;
  const query = e.target.value;

  if (pref && query) {
    updateMunicipalitySuggestions(pref, query);
  }
});
```

### 4.3 バリデーション

```javascript
/**
 * 入力バリデーション
 */
function validateInput() {
  const pref = document.getElementById('pref-select').value;
  const muni = document.getElementById('muni-input').value.trim();

  // 都道府県チェック
  if (!pref) {
    return { ok: false, message: '都道府県を選択してください' };
  }

  // 市区町村チェック
  if (!muni) {
    return { ok: false, message: '市区町村を入力してください' };
  }

  // 候補リスト存在チェック
  const rows = appData.rowsByPref.get(pref) || [];
  const exists = rows.some(row => row.muni === muni);

  if (!exists) {
    return {
      ok: false,
      message: '市区町村は候補（プルダウン）から選択してください'
    };
  }

  return { ok: true, pref, muni };
}
```

### 4.4 判定処理（コアロジック）

```javascript
/**
 * 判定取得
 */
function getJudgement(pref, muni) {
  // 該当行検索
  const rows = appData.rowsByPref.get(pref) || [];
  const matches = rows.filter(row => row.muni === muni);

  // ヒット件数チェック
  if (matches.length === 0) {
    return {
      ok: false,
      message: '該当する市区町村が見つかりませんでした'
    };
  }

  if (matches.length > 1) {
    return {
      ok: false,
      message: '同一の都道府県×市区町村が複数行に存在します（データ重複）'
    };
  }

  // 判定セル読み取り
  const targetRow = matches[0];
  const values = targetRow.rowData.slice(appData.config.FIRST_JUDGE_COL - 1);

  // 結果アイテム生成
  const items = [];
  for (let i = 0; i < values.length; i++) {
    const provider = appData.providers[i];
    const category = appData.categories[i];
    const value = values[i];

    // 空ヘッダ列は除外
    if (!provider || !category) continue;

    items.push({
      provider: provider.trim(),
      category: category.trim(),
      value: value ? value.trim() : ''
    });
  }

  return {
    ok: true,
    pref,
    muni,
    items
  };
}
```

### 4.5 結果表示

```javascript
/**
 * カテゴリ名変換マップ
 */
const CATEGORY_DISPLAY_MAP = {
  '大規模': '大規模リフォーム',
  '小規模': '小規模リフォーム'
};

/**
 * ステータス判定
 */
function getStatusClass(value) {
  if (!value || value === '—') return 'unavailable';
  if (value.includes('要相談')) return 'consult';
  if (value.includes('一部') || value.includes('除く')) return 'partial';
  if (value === '○' || value.includes('対応')) return 'available';
  return 'unavailable';
}

/**
 * ステータス表示文言
 */
function getStatusText(value, statusClass) {
  if (statusClass === 'unavailable') return '対応エリア外';
  if (statusClass === 'available') return '対応可能';
  return value;  // 要相談、一部対応などはそのまま
}

/**
 * 結果表示
 */
function displayResult(result) {
  if (!result.ok) {
    showError(result.message);
    return;
  }

  // タイトル設定
  document.getElementById('result-title').textContent =
    `${result.pref} ${result.muni}`;

  // カード生成
  const container = document.getElementById('result-cards');
  const template = document.getElementById('card-template');

  container.innerHTML = '';  // クリア

  result.items.forEach(item => {
    const card = template.content.cloneNode(true);

    // プロバイダー種別
    card.querySelector('.provider-type').textContent = item.provider;

    // カテゴリ（変換）
    const displayCategory = CATEGORY_DISPLAY_MAP[item.category] || item.category;
    card.querySelector('.category').textContent = displayCategory;

    // ステータスバッジ
    const statusClass = getStatusClass(item.value);
    const statusText = getStatusText(item.value, statusClass);
    const badge = card.querySelector('.status-badge');
    badge.textContent = statusText;
    badge.classList.add(statusClass);

    container.appendChild(card);
  });

  // 結果エリア表示
  document.getElementById('result-area').style.display = 'block';

  // スクロール（モバイル対応）
  document.getElementById('result-area').scrollIntoView({
    behavior: 'smooth'
  });
}

/**
 * エラー表示
 */
function showError(message) {
  // 既存の結果を非表示
  document.getElementById('result-area').style.display = 'none';

  // エラーメッセージ表示（アラートまたは専用UI）
  alert(message);  // シンプル版

  // または専用のエラー表示エリア
  // document.getElementById('error-area').textContent = message;
  // document.getElementById('error-area').style.display = 'block';
}
```

### 4.6 確認ボタン処理

```javascript
/**
 * 確認ボタンクリック
 */
document.getElementById('check-btn').addEventListener('click', async () => {
  // バリデーション
  const validation = validateInput();
  if (!validation.ok) {
    showError(validation.message);
    return;
  }

  // ローディング表示
  showLoading(true);

  try {
    // 判定取得
    const result = getJudgement(validation.pref, validation.muni);

    // 結果表示
    displayResult(result);

  } catch (error) {
    showError('エラーが発生しました');
    console.error(error);
  } finally {
    showLoading(false);
  }
});

/**
 * ローディング表示
 */
function showLoading(show) {
  const btn = document.getElementById('check-btn');
  if (show) {
    btn.disabled = true;
    btn.textContent = '検索中...';
  } else {
    btn.disabled = false;
    btn.textContent = '確認する';
  }
}
```

---

## 5. ファイル構成と実装計画

### 5.1 ファイル構成

```
libe_koumu_area/
├── index.html          # メインHTML（All-in-one または分割）
├── css/
│   └── style.css       # スタイルシート（任意：HTMLに内包も可）
├── js/
│   ├── config.js       # 設定ファイル
│   ├── data.js         # データ管理・CSV解析
│   └── app.js          # UIロジック
├── assets/
│   └── (画像等)
├── README.md           # プロジェクト説明
└── .gitignore

※シンプル版：index.html に全てを内包（CSS/JS）
```

### 5.2 実装順序

1. **フェーズ1：データ解析基盤**
   - CSV取得・解析処理
   - インデックス構築
   - 単体テスト（コンソールで動作確認）

2. **フェーズ2：UI構築**
   - HTML/CSSレイアウト
   - 都道府県プルダウン
   - 市区町村サジェスト

3. **フェーズ3：判定ロジック**
   - バリデーション
   - 判定処理
   - 結果表示

4. **フェーズ4：統合・調整**
   - エラーハンドリング
   - ローディング表示
   - レスポンシブ対応

5. **フェーズ5：デプロイ**
   - GitHub Pagesセットアップ
   - 本番CSV URL設定
   - 動作確認

---

## 6. エラーハンドリング設計

### 6.1 エラー分類

| エラータイプ | トリガー | 表示方法 | メッセージ |
|------------|---------|---------|-----------|
| **入力エラー** | バリデーション失敗 | アラート | 「都道府県を選択してください」等 |
| **データエラー** | 該当行0件・重複 | アラート | 「該当する市区町村が見つかりませんでした」等 |
| **ネットワークエラー** | CSV取得失敗 | アラート | 「データの読み込みに失敗しました」 |
| **パースエラー** | CSV解析失敗 | コンソール+アラート | 同上 |

### 6.2 エラー表示UI（改善版）

```html
<!-- エラー表示エリア -->
<div id="error-area" class="error-message" style="display:none;">
  <span class="error-icon">⚠️</span>
  <span id="error-text"></span>
</div>
```

```css
.error-message {
  background-color: #FFF3CD;
  border: 1px solid #FFC107;
  border-radius: 4px;
  padding: 12px 16px;
  margin: 16px 0;
  display: flex;
  align-items: center;
  gap: 8px;
}

.error-icon {
  font-size: 20px;
}
```

```javascript
function showError(message) {
  const errorArea = document.getElementById('error-area');
  const errorText = document.getElementById('error-text');

  errorText.textContent = message;
  errorArea.style.display = 'flex';

  // 結果エリアは非表示
  document.getElementById('result-area').style.display = 'none';
}

function hideError() {
  document.getElementById('error-area').style.display = 'none';
}
```

---

## 7. パフォーマンス最適化

### 7.1 最適化項目

| 項目 | 手法 | 効果 |
|------|------|------|
| **CSV取得** | 1回のみ（ページロード時） | ネットワーク負荷削減 |
| **サジェスト** | debounce (200ms) | タイピング連打を防ぐ |
| **インデックス** | Map構造（都道府県別） | O(1)検索 |
| **重複除外** | Set使用 | 高速 |
| **DOM操作** | template使用 | 効率的な生成 |

### 7.2 大規模データ対応（将来拡張）

データが1万行を超える場合の対策：

1. **仮想スクロール**：カード表示を仮想化（react-window等）
2. **都道府県別CSV分割**：47都道府県で分散
3. **Web Worker**：CSV解析をバックグラウンド処理
4. **IndexedDB**：ブラウザキャッシュ活用

※現状は数百〜数千行程度を想定し、上記は不要

---

## 8. デプロイメント設計

### 8.1 GitHub Pages デプロイ手順

```bash
# 1. リポジトリ作成
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/[username]/[repo-name].git
git push -u origin main

# 2. GitHub Pages設定
# Settings → Pages
# Source: Deploy from a branch
# Branch: main / root
# Save

# 3. 公開URL確認
# https://[username].github.io/[repo-name]/
```

### 8.2 CSV公開URL取得方法

```
1. Googleスプレッドシートを開く
2. 「公開_まとめ」シートを選択
3. ファイル → 共有 → ウェブに公開
4. 「公開_まとめ」シートを選択
5. 形式：「カンマ区切り形式(.csv)」
6. 公開ボタン → URLコピー

形式例：
https://docs.google.com/spreadsheets/d/[SHEET_ID]/export?format=csv&gid=[GID]
```

### 8.3 config.js 設定例

```javascript
const appData = {
  config: {
    // 本番用CSV URL（要置換）
    CSV_URL: 'https://docs.google.com/spreadsheets/d/1AbCdEfGhIjKlMnOpQrStUvWxYz/export?format=csv&gid=123456789',

    // お問い合わせURL
    CONTACT_URL: 'https://example.com/contact',

    // ヘッダ行設定
    HEADER_PROVIDER_ROW: 2,
    HEADER_CATEGORY_ROW: 3,
    DATA_START_ROW: 4,
    FIRST_JUDGE_COL: 4,

    // サジェスト上限
    SUGGEST_LIMIT: 30
  }
};
```

### 8.4 更新フロー

```
【データ更新時】
1. Googleスプレッドシート「まとめ」シートを編集
2. 「公開_まとめ」シートに自動転記される（ARRAYFORMULA等）
3. 数分待つ（Google側のキャッシュ更新）
4. GitHub Pagesを開いてブラウザキャッシュクリア（Ctrl+F5）
5. 更新反映を確認

【コード更新時】
1. ローカルで修正
2. git commit & push
3. GitHub Pagesは自動デプロイ（数秒〜数分）
4. 反映確認
```

---

## 9. テスト計画

### 9.1 テストケース

| ID | テスト項目 | 入力 | 期待結果 |
|----|----------|------|---------|
| T01 | 都道府県プルダウン生成 | - | CSV A列から重複なしリスト |
| T02 | 都道府県未選択 | 都道府県=空, 市区町村=入力 | エラー「都道府県を選択」 |
| T03 | 市区町村未入力 | 都道府県=選択, 市区町村=空 | エラー「市区町村を入力」 |
| T04 | 候補外入力 | 都道府県=北海道, 市区町村=存在しない市 | エラー「候補から選択」 |
| T05 | 正常検索 | 都道府県=北海道, 市区町村=札幌市 | 該当行の判定結果表示 |
| T06 | サジェスト（muni一致） | 都道府県=北海道, query=札幌 | 札幌市が候補に表示 |
| T07 | サジェスト（kana一致） | 都道府県=北海道, query=さっぽろ | 札幌市が候補に表示 |
| T08 | データ重複 | 同一pref×muniが2行存在 | エラー「データ重複」 |
| T09 | CSV取得失敗 | 無効URL | エラー「読み込み失敗」 |
| T10 | レスポンシブ | モバイル/タブレット/PC | 各画面で適切表示 |

### 9.2 動作確認環境

- Chrome（最新版）
- Safari（最新版）
- Edge（最新版）
- モバイルブラウザ（iOS Safari, Chrome Android）

---

## 10. 付録

### 10.1 現行HTML参照パス

```
C:\Users\81905\Dropbox (個人用)\CursorProjects\libe_koumu_area\index.html
```

このファイルから以下を踏襲：
- フォント設定（Noto Sans JP）
- カラーパレット
- カードUI構造
- ボタンスタイル

### 10.2 サンプルデータ（sample_image.png より）

```csv
# 全エリアの対応状況,,,,,,,
,,, 工務店, 工務店, 工務店, 工務店, 職人, 職人
都道府県, 市区町村, ふりがな, 注文住宅, 大規模, 小規模, オフィス・店舗, 小規模, オフィス・店舗
北海道, 札幌市, さっぽろし,, ○, ○, ○,,
北海道, 函館市, はこだてし,,,,,,
北海道, 小樽市, おたるし,, ○, ○, ○,,
北海道, 旭川市, あさひかわし,, 要相談, 要相談, 要相談,,
北海道, 室蘭市, むろらんし,, 要相談, 要相談,,,
北海道, 釧路市, くしろし,,,,,,
北海道, 帯広市, おびひろし,,,,,,
北海道, 北見市, きたみし,,,,,,
北海道, 夕張市, ゆうばりし,, 要相談, 要相談, 要相談,,
北海道, 岩見沢市, いわみざわし,, ○, ○, ○,,
```

---

**以上、詳細設計書を完成させました。**

次の実装フェーズでは、この設計書に基づいて：
1. index.html（HTML/CSS/JS統合版）
2. または分割版（html, css, js）

のいずれかを選択して実装を進めます。
