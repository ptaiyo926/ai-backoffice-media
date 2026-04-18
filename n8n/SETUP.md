# n8n ワークフロー セットアップガイド

## 概要

4つのワークフローが連携して毎朝5:30〜7:00の間に自動実行されます。

```
[WF01: RSS収集+フィルタ] → [WF02: 記事生成] → [WF03: ファクトチェック] → [WF04: GitHub Push+配信]
      ↓                           ↓                      ↓                          ↓
  毎朝5:30実行            Gemini Flash生成          Gemini Flash-Lite          GitHub commit
  3つのRSSを取得          Hugo Markdown形式         数値・制度名確認           Beehiiv予約(7:00)
  関連度スコアリング                                                           X(Twitter)投稿
```

---

## ステップ1: 環境変数を設定

n8n管理画面 → Settings → Environment Variables に以下を追加:

| 変数名 | 説明 | 取得場所 |
|--------|------|----------|
| `GEMINI_API_KEY` | Gemini APIキー | https://aistudio.google.com/ |
| `GITHUB_TOKEN` | GitHubアクセストークン (repo権限) | GitHub → Settings → Developer settings → PAT |
| `GITHUB_OWNER` | GitHubユーザー名 | 例: `your-username` |
| `GITHUB_REPO` | リポジトリ名 | 例: `ai-backoffice-media` |
| `BEEHIIV_API_KEY` | Beehiiv APIキー | Beehiiv → Settings → Integrations → API |
| `BEEHIIV_PUBLICATION_ID` | BeehiivのPublication ID | 同上 |
| `SITE_URL` | サイトURL | 例: `https://ai-backoffice.jp` |

---

## ステップ2: X (Twitter) 認証情報を設定

n8n管理画面 → Credentials → New Credential → **OAuth1 API**

- Consumer Key: Twitter Developer Portalで取得
- Consumer Secret: 同上
- Access Token: 同上
- Access Token Secret: 同上

作成後、WF04の「X (Twitter): 自動投稿」ノードでこの認証情報を選択してください。

---

## ステップ3: ワークフローをインポート

n8n管理画面 → Workflows → Import from file で以下の順にインポート:

1. `workflow-01-rss-filter.json`
2. `workflow-02-article-generation.json`
3. `workflow-03-fact-check.json`
4. `workflow-04-github-push-email.json`

インポート後、各ワークフローのIDをメモします。
（ワークフローを開いたときのURLの末尾の数字がID）

---

## ステップ4: ワークフロー間のIDを設定

各ワークフローのIDを確認し、以下の箇所を更新:

### WF01 (RSS収集) → WF02を呼び出すノード
`→ WF02: 記事生成` ノードの `workflowId` を WF02のIDに変更

### WF02 (記事生成) → WF03を呼び出すノード
`→ WF03: ファクトチェック` ノードの `workflowId` を WF03のIDに変更

### WF03 (ファクトチェック) → WF04を呼び出すノード
`→ WF04: GitHub Push + 配信` ノードの `workflowId` を WF04のIDに変更

---

## ステップ5: テスト実行

### 個別テスト（推奨順序）

1. **WF04から逆順でテスト**
   - WF04を手動実行 → GitHubにファイルが作成されるか確認

2. **WF03のテスト**
   - サンプル記事データでWF03を手動実行 → ファクトチェック結果を確認

3. **WF02のテスト**
   - サンプル記事データでWF02を手動実行 → Markdown生成を確認

4. **WF01のテスト（全体テスト）**
   - WF01を手動実行 → 全パイプラインが通るか確認

### テスト用ダミーデータ（WF03/WF04直接テスト用）

n8nのWF03 trigger ノードで「Test step」→ 以下のJSONを入力:
```json
{
  "filename": "2026-04-16-test-article.md",
  "content": "---\ntitle: \"テスト記事\"\ndate: 2026-04-16T07:00:00+09:00\ndraft: false\ntype: \"daily\"\nsummary: \"テスト記事です\"\nimpact_score: 3\ntags: [\"テスト\"]\nsources:\n  - \"https://example.com\"\n---\n\n## 概要\n\nこれはテスト記事です。",
  "originalTitle": "テスト記事",
  "sourceUrl": "https://example.com",
  "today": "2026-04-16"
}
```

---

## ステップ6: WF01を有効化

テストが成功したら、WF01の「毎朝5:30 JST」スケジュールを有効化します。

n8n上でWF01を開き、右上の「Active」トグルをONにする。

---

## トラブルシューティング

| 症状 | 原因 | 対処法 |
|------|------|--------|
| RSS取得が空 | フィードURLが変更された | n8nでURLを手動確認・更新 |
| Gemini APIエラー 429 | レート制限超過 | 1記事ずつ処理 + 5秒待機ノードを追加 |
| GitHubエラー 422 | ファイルが既に存在する | SHAを取得してからPUTする（後述） |
| Beehiivエラー 401 | APIキーが無効 | キーを再生成して環境変数を更新 |

### GitHubの422エラー対処法

同じファイル名でコミットしようとすると422エラーが発生します。WF04の`GitHub: 記事をコミット`ノードの前に以下のノードを追加してください:

1. HTTP Request (GET) で既存ファイルのSHAを取得
2. Codeノードで `sha` フィールドをbodyに追加
3. PUT リクエストに `sha` を含める
