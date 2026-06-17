# 設計書：XSS修正 & Claude AI機能追加

**日付：** 2026-06-17  
**対象：** d:\WebSite\mermaid-studio\index.html

---

## 概要

Mermaid Studio に Claude API を使った図の生成・修正機能を追加する。
APIキーはメモリ（JS変数）のみに保持し、セキュリティリスクを最小化する。
実装前提として、既存コードの XSS 脆弱性を先に修正する。

---

## フェーズ1：XSS修正

### 背景

セキュリティレビューにより、既存コードに2箇所の XSS リスクが発見された。
Claude API キーをメモリ保存しても、XSS が成立すれば盗まれるため先に対処する。

### 修正対象

#### 1. フォルダ名のテンプレート展開

- **問題：** Firestoreから取得したフォルダ名を innerHTML のテンプレートリテラルに直接埋め込んでいる
- **修正：** `createElement` + `textContent` を使い、HTMLエスケープなしに安全に挿入する

**修正前（例）：**
```js
el.innerHTML = `<span class="folder-name">${folderName}</span>`;
```

**修正後：**
```js
const span = document.createElement('span');
span.className = 'folder-name';
span.textContent = folderName;
el.appendChild(span);
```

#### 2. Mermaid SVG挿入

- **問題：** `stage.innerHTML = svg` でMermaid生成SVGをそのまま挿入
- **リスク評価：** SVGはユーザー自身が書いたMermaidコードから生成されるため自己攻撃のみ。リスクは低いが修正する
- **修正：** `DOMParser` でパースし、`stage.appendChild()` で挿入する

**修正後：**
```js
const doc = new DOMParser().parseFromString(svg, 'image/svg+xml');
const svgEl = doc.querySelector('svg');
stage.innerHTML = '';
stage.appendChild(svgEl);
```

---

## フェーズ2：Claude AI機能

### アーキテクチャ

```
ユーザー入力（プロンプトバー）
    ↓
callClaude(instruction, currentCode)
    ↓
fetch → Anthropic API (/v1/messages)
    ↓
Mermaidコードを抽出
    ↓
エディタ更新 → 再レンダリング
```

### UIレイアウト

エディタ下部に常時表示のプロンプトバーを追加：

```
[ コードエディタ（textarea）            ]
────────────────────────────────────────
[🤖] [ 指示を入力…              ] [生成▶]
```

- `🤖` ボタン：APIキー入力モーダルを開く
- テキスト入力：指示文（例：「エラーハンドリングを追加して」「ユーザー登録フローを作って」）
- `生成` ボタン：APIを呼び出す。APIキー未設定時は自動でキーモーダルを開く

### APIキー管理

- **保存場所：** JS変数（メモリのみ）
- **ページ読み込みごとに消える**：セッション開始時に毎回入力が必要
- **入力UI：** モーダルダイアログ（テキスト入力 + 保存ボタン）
- localStorage / sessionStorage には一切保存しない

### `callClaude()` 処理フロー

1. APIキー変数が空 → キー入力モーダルを開いて待機
2. ローディング表示（生成ボタンをスピナーに変更）
3. `fetch('https://api.anthropic.com/v1/messages', {...})` を呼び出し
4. レスポンスからMermaidコードを抽出
5. 成功：エディタ内容を更新 → `render()` 実行
6. 失敗：トーストでエラー表示（`AI生成に失敗しました：{code}` 形式）

### Anthropic API 呼び出し仕様

- **モデル：** `claude-haiku-4-5-20251001`（安価・高速）
- **max_tokens：** 2048
- **system prompt：**
  ```
  You are a Mermaid diagram expert.
  Output ONLY valid Mermaid diagram code.
  No explanations, no markdown fences, no comments.
  ```
- **user message（修正時）：**
  ```
  Current diagram:
  {currentCode}

  Instruction: {instruction}
  ```
- **user message（新規生成時）：**
  ```
  Instruction: {instruction}
  ```
  ※ エディタが空の場合は currentCode を含めない

### CORSについて

Anthropic API は `api.anthropic.com` への直接 fetch をブラウザからサポートしている。
`anthropic-dangerous-direct-browser-access: true` ヘッダーが必要。

### エラーハンドリング

| エラー | 対応 |
|--------|------|
| APIキー不正（401） | 「APIキーが無効です。再設定してください」+ キーをクリア |
| レート制限（429） | 「しばらく待ってから再試行してください」 |
| ネットワークエラー | 「通信エラーが発生しました」 |
| Mermaid構文エラー | 既存の `render()` エラー表示に委ねる |

---

## 対象外（スコープ外）

- チャット履歴の保持
- モデル選択UI（Haiku固定）
- ストリーミングレスポンス
- Undo/Redo（既存の textarea 操作で対応可能）
