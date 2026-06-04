# 引き継ぎ指示書 — PJ管理ダッシュボード

> このドキュメントは Claude Code への引き継ぎ用です。
> 上から順に読めば、背景・目的・仕様・現状・次にやることが把握できます。

---

## 1. このプロジェクトの経緯

複数のプロジェクト（PJ）を、ディレクトリを分けて Claude Code で並行進行している。
各PJの進捗は Claude Code が作業ログを随時確認しているため Claude Code 自身は把握しているが、
**人間（オーナー）が全体像を知るには、その都度 Claude Code に聞かなければならない**という課題があった。

そこで「複数PJの進捗を一目で確認できるダッシュボード」を作ることにした。

設計・UIの土台づくりは別のチャットで完了済み。
**ここから先（ローカルのファイル操作・status.json の自動更新・GitHub公開）を Claude Code に引き継ぐ。**

---

## 2. 目的（ゴール）

1. 複数PJの進捗・状態・最新ログ・要対応事項を**1画面で**確認できる
2. ダッシュボードは**手動更新がデフォルト**、トグルONで自動更新（30秒ごと）
3. 最終的に **GitHub Pages で公開**し、固定URLで他者にも共有できる（閲覧のみでよい）

---

## 3. 実現手段（全体アーキテクチャ）

```
各PJディレクトリ（Claude Codeが作業中）
        │  作業の節目で status.json を更新
        ▼
   status.json  ← 全PJの状態を集約した1ファイル
        │  ダッシュボードHTMLが読み込む
        ▼
   dashboard.html  ← 手動/自動更新で表示
        │  git push
        ▼
   GitHub Pages（固定URLで公開・閲覧のみ）
```

ポイント：**ダッシュボードは status.json を読むだけ**のシンプルな静的サイト。
頭脳（状況の集約）は Claude Code 側が担い、結果を status.json に書き出す。

---

## 4. 現在の進行状況

### ✅ 完了済み
- `dashboard.html` … ダッシュボード本体（完成・動作確認済み）
  - status.json を fetch して描画
  - status.json が無い場合はダミーデータ＋警告バナーを表示
  - 右上に「⟳ PJ進捗更新」ボタン（手動更新）と「自動更新」トグル（OFFが初期値）
  - レスポンシブ対応（スマホ可）
- `status.json` … データのサンプル（フォーマット確定用）
- ローカルでの表示確認済み（`python3 -m http.server` で表示できることを確認）

### 📂 現在のファイル配置
```
/Users/yastaka_watanabe_auji_industries/Desktop/Bern/board of PJ/files/
├── dashboard.html
└── status.json
```

### ⬜ これからやること（このドキュメントの「6. 依頼タスク」参照）
- status.json を各PJの実状況から生成・更新する仕組み
- 各PJの CLAUDE.md に「作業の節目で status.json を更新する」ルールを追記
- Git 初期化 → GitHub リポジトリ作成 → GitHub Pages で公開

---

## 5. status.json の仕様

ダッシュボードはこの構造を前提に描画する。**キー名・構造は変更しないこと。**
（フィールドを増やす場合は dashboard.html 側の描画も合わせて変更が必要）

```jsonc
{
  "title": "Project Dashboard",            // ダッシュボードのタイトル
  "updated_at": "2024-06-04T14:32:00Z",    // このファイルの最終更新（ISO8601 / UTC）
  "summary": {
    "total": 6,            // 総PJ数
    "active": 3,           // 作業中の数
    "waiting": 1,          // 承認待ちの数
    "done": 1,             // 完了の数
    "error": 1,            // エラーの数
    "commits_today": 27,   // 今日のコミット総数
    "commits_add": 841,    // 今日の追加行数
    "commits_del": 203,    // 今日の削除行数
    "tasks_done_today": 14,// 今日完了したタスク数
    "alerts": 2            // 要確認件数（= alerts配列の件数）
  },
  "projects": [
    {
      "id": "ec-renewal",                  // 一意のID（ディレクトリ名推奨）
      "name": "ECサイトリニューアル",        // 表示名
      "dir": "/projects/ec-renewal",       // ディレクトリパス
      "status": "active",                  // active | waiting | done | error のいずれか
      "progress": 68,                      // 進捗率（0〜100の整数）
      "latest_log": "カート機能の…",         // 最新の状況を1〜2文で
      "updated_at": "2024-06-04T14:29:00Z",// このPJの最終更新（ISO8601 / UTC）
      "todos": 5                           // 残TODO件数
    }
    // …PJの数だけ繰り返し
  ],
  "alerts": [
    {
      "type": "error",                     // error | waiting （ダッシュボードで色分け）
      "project_id": "data-migration",      // 対象PJのid
      "message": "DB接続エラー — …",         // 人間が見て対応できる内容
      "time": "2024-06-04T13:47:00Z"
    }
    // …要確認事項の数だけ
  ]
}
```

### status の意味（4状態）
| status | 意味 | 色 |
|--------|------|----|
| `active` | 作業中 | 緑 |
| `waiting` | 承認待ち・人間の判断待ちで停止中 | オレンジ |
| `done` | 完了 | 青 |
| `error` | エラー・要対応 | 赤 |

---

## 6. 依頼タスク（Claude Code への作業指示）

以下を順番に進めてほしい。各ステップ完了時に簡単に報告すること。

### Task 1 — status.json 生成スクリプトの作成
- 各PJディレクトリ（実際のパスはオーナーに確認）を走査し、状況を集約して
  `status.json` を生成・上書きするスクリプトを作る（言語は Python か シェルが扱いやすい）。
- 進捗の判定材料の候補（取得できるものを使う／無理に全部使わなくてよい）：
  - 各PJの `TODO.md` や `.claude/` 内のログ → 残TODO数・最新ログ・status
  - `git log` → 今日のコミット数、追加/削除行数（`git log --since=midnight --numstat` など）
  - 進捗率は TODO の消化率などから概算（難しければ手動値でも可）
- **判定ロジックが曖昧な項目は、勝手に決めず一度オーナーに確認すること。**

### Task 2 — 各PJの CLAUDE.md にルール追記
各PJの `CLAUDE.md`（無ければ作成）に、次の趣旨のルールを追記：
> 「作業の節目（タスク完了・エラー発生・承認待ちで停止する時）に、
>  このPJの状況を共有ダッシュボード用に記録すること。
>  具体的には Task 1 のスクリプトを実行して status.json を更新する。」

※ 文面はオーナーと相談して調整可。

### Task 3 — GitHub Pages で公開
- `dashboard.html` と `status.json` を置くリポジトリを用意する。
- リポジトリ名は `dashboard` などオーナーに確認。**公開（public）でよいか必ず確認すること**
  （status.json の中身が外部に見える＝PJ名やログが公開される点をオーナーに説明し、了承を得る）。
- GitHub Pages を有効化し、`dashboard.html` が固定URLで閲覧できる状態にする。
  - ルートに置く場合 `https://<user>.github.io/<repo>/dashboard.html`
  - `index.html` にリネームすれば `https://<user>.github.io/<repo>/` で開ける（お好みで）
- 可能なら status.json を push したら自動反映される運用にする
  （単純な push 反映で十分。GitHub Actions は必須ではない）。

---

## 7. オーナーに確認すべきこと（着手前に）

1. 各PJの**実際のディレクトリパス**（一覧）
2. 進捗率や status の**判定ロジック**（自動算出か手動入力か）
3. GitHubリポジトリを**public で公開してよいか**（status.json の中身が外部に見える点を含めて）
4. リポジトリ名・公開URLの希望

---

## 8. 動作確認方法（参考）

ローカルでの表示確認：
```bash
cd "/Users/yastaka_watanabe_auji_industries/Desktop/Bern/board of PJ/files"
python3 -m http.server 8080
# ブラウザで http://localhost:8080/dashboard.html を開く
```
- 右上「⟳ PJ進捗更新」で手動更新
- 「自動更新」トグルONで30秒ごとに自動更新
- status.json を編集して更新 → 画面に反映されればOK

---

## 9. 設計上の注意点・思想

- **ダッシュボードは「読むだけ」に徹する**。ロジックは status.json 生成側に寄せる。
  そうすることで表示と集計が分離され、片方を壊してももう片方に影響しない。
- status.json の**キー構造を勝手に変えない**。変える場合は dashboard.html の描画コードも同時に直す。
- 公開する以上、**status.json に機密情報（鍵・トークン・社外秘の詳細）を書かない**。
  latest_log は「外に見えても問題ない粒度」で書くこと。
- 困ったら（仕様が曖昧、判断に迷う）**勝手に進めず、オーナーに確認する**。
