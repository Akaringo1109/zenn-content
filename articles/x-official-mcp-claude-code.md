---
title: "Xへの投稿をMCPで自動化したい。公式MCPの全24ツールを試した結論"
emoji: "🐦"
type: "tech"
topics: ["claudecode", "mcp", "x", "twitter", "api"]
published: true
published_at: 2026-07-18 19:00
---

Claude Code で自分用の仕組みを作りながら、外資系コンサルで AI 導入をリードしているあかりんごです。
インプットの自動化はずいぶん進んだのに、アウトプット、つまり X（旧 Twitter）への発信だけは毎回手で打っている。そんなちぐはぐさ、ありませんか。

私は情報収集まわりはかなり自動化してきました。でも X への投稿は、結局その都度ブラウザを開いて書いて出す、の繰り返しです。この投稿という行為そのものを Claude Code から自動で流せたら、と前から思っていました。

そこに 2026 年 6 月末、X が公式の MCP サーバーを出しました。「これで投稿を自動化できる」と期待して繋いだのですが、結論から言うと、狙っていた投稿の自動化はこの MCP ではできませんでした。ただ、全24ツールを実際に叩いてみると、できること・できないこと・課金しないと届かないことの線引きがはっきりしたので、そこを共有します。

:::message
実測は 2026 年 7 月時点、X API の **Pay Per Use（従量課金・クレジット制）アカウント**で確認したものです。手元はクレジットが尽きた状態で、`get_users_me` 以外はほぼ `402 credits depleted` が返りました。各表の「実測」列はこの状態での結果です。「必要ティア」列は X 公式ドキュメント基準の目安で、料金は執筆時点のものなので最新は公式で確認してください。
:::

## 期待して繋いだら、投稿ツールが無かった

X は 2026 年 6 月 30 日に公式 MCP サーバーを公開しました。`@xdevplatform/xurl` という CLI が、Claude Code の標準入出力と X の API（`https://api.x.com/mcp`）の間をリレーします。

期待して繋いで、まず投稿系のツールを探しました。無いんです。**投稿の作成・削除に当たるツールは、現時点の公式 MCP サーバーに搭載されていません。** 用意されているのは、閲覧・分析・ブックマーク管理まわりの24ツール。「MCP でツイートを自動投稿する」は、少なくとも今は叶いませんでした。

MCP サーバーが返す全ツール（`tools/list` の結果）を並べると、こうなります。ちょうど24個。`create_` で始まるのはブックマーク系だけで、ツイートを作る・消すツールはどこにもありません。

```text
# X 公式 MCP が返す全24ツール（name / description）

# ユーザー情報
get_users_me                      Get Users Me
get_users_by_id                   Get Users by ID
get_users_by_username             Get Users by Username
get_users_by_usernames            Get Users by Usernames
search_users                      Search Users

# 投稿・タイムライン（すべて読み取り）
get_users_posts                   Get Users Posts
get_users_timeline                Get Users Timeline
get_users_mentions                Get Users Mentions
get_posts_by_id                   Get Posts by ID
get_posts_by_ids                  Get Posts by IDs
get_posts_counts_recent           Get Posts Counts Recent

# エンゲージメント
get_posts_liking_users            Get Posts Liking Users
get_posts_reposted_by             Get Posts Reposted by
get_posts_quoted_posts            Get Posts Quoted Posts

# 検索・トレンド・ニュース
search_posts_all                  Search Posts All
get_trends_by_woeid               Get Trends by Woeid
search_news                       Search News
get_news                          Get News

# ブックマーク
get_users_bookmarks               Get Users Bookmarks
create_users_bookmark             Create Users Bookmark
delete_users_bookmark             Delete Bookmark
get_users_bookmark_folders        Get Users Bookmark Folders
create_users_bookmark_folder      Create Bookmark Folder
get_users_bookmarks_by_folder_id  Get Users Bookmarks by Folder ID
```

投稿を取得・分析する read 系のツールは、description の末尾に `Each returned post includes a url deeplinking to it on x.com`（各結果に x.com へのリンクが付く）が添えられます。

がっかりして終わるのももったいないので、繋いだついでに全24ツールを叩いて「では何ができるのか」を確かめました。そして最後に、本命の投稿自動化を MCP の外でどう実現するかまで書きます。

## 繋ぐ：OAuth 2.0 でつまずく1点

ツールを試すにも、まず接続です。ここで1か所だけ確実に詰まるポイントがあるので先に潰しておきます。

### Client ID が見つからない（OAuth 1.0a と 2.0 の混同）

xurl は OAuth 2.0 の PKCE フローで認証します。必要なのは **Client ID** と **Client Secret** の2つです。

ところが X Developer Portal でアプリを作ると、最初に出てくるのは **Consumer Key・API Key Secret・Bearer Token** の3点セットでした。これは **OAuth 1.0a** 用の認証情報で、xurl が使う OAuth 2.0 とは別物です。「Client ID がどこにも無い」で、ここで一度手が止まりました。

正解は、Developer Portal の **User Authentication Settings** を設定すること。これを保存して初めて OAuth 2.0 の Client ID / Secret が発行されます。設定を触る前は存在すらしないのが引っかかりどころでした。順序が直感に反しているので、同じところで詰まる人はきっといます。実際の画面で順に追っていきます。

**1. Developer Console でアプリを作る**

[X Developer Console](https://developer.x.com/) の左メニュー「アプリ」を開き、右上の作成ボタンをクリックします。

![Developer Console の「アプリ」画面。右上の作成ボタンから新規アプリを作る](/images/x-official-mcp-claude-code/step1-apps-create.png)

アプリケーション名を入力し、Environment（ここでは Production）を選んで「新しいクライアントアプリケーションを作成」をクリックします。

![アプリ作成ダイアログ。名前と Environment を決めて作成する](/images/x-official-mcp-claude-code/step2-create-dialog.png)

**2. 最初に出る鍵は「OAuth 1.0a」だと知っておく**

作成直後、コンシューマーキー・Secret Key・ベアラートークンが表示されます。「これを xurl に使うのかな」と思いがちですが、**これは OAuth 1.0a の認証情報で、xurl には使いません。** 一度きりの表示なので、保存だけして×で閉じて構いません。

![作成直後に出る OAuth 1.0a の3点セット。xurl では使わない](/images/x-official-mcp-claude-code/step3-oauth1-keys.png)

**3. 「ユーザー認証設定」をセットアップする**

アプリの「Keys & Tokens」タブを開き、一番下の「ユーザー認証設定」にある「セットアップ」をクリックします。ここが OAuth 2.0 の入口です。

![Keys & Tokens タブ下部の「ユーザー認証設定」→「セットアップ」](/images/x-official-mcp-claude-code/step4-user-auth-setup.png)

**4. 認証設定フォームを埋める（必須項目に注意）**

必須項目を設定します。ポイントは3つです。

- **アプリの権限**：閲覧だけなら「読む」で十分ですが、**xurl から投稿もしたいなら「読み取りと書き込み」**を選びます（この権限を後から変えたら、OAuth ログインをやり直してトークンを取り直します）。
- **アプリの種類**：「ウェブアプリ、自動化アプリまたはボット（機密クライアント）」を選ぶと OAuth 2.0 が有効になります。
- **アプリ情報（必須）**：
  - コールバックURI / リダイレクトURL → `http://localhost:8080/callback`（後で `xurl auth apps add` に渡す値と完全一致させます）
  - ウェブサイトURL → 自分が持つ有効な公開URL（`localhost` は不可。認証の動作には影響しないメタ情報なので、Zenn やブログの URL でよい）

入力したら「変更を保存する」をクリックします。

![認証設定フォーム。権限・アプリの種類・コールバックURI・ウェブサイトURL を設定して保存](/images/x-official-mcp-claude-code/step5-auth-settings.png)

**5. OAuth 2.0 の Client ID / Secret を受け取る**

保存すると、ようやく **OAuth 2.0 のクライアントID・クライアントシークレット**が表示されます。これが xurl に渡す値です。こちらも一度きりの表示なので、必ずコピーして安全な場所に保存します。

![保存後に表示される OAuth 2.0 のクライアントID / クライアントシークレット。これが xurl に渡す値](/images/x-official-mcp-claude-code/step6-oauth2-clientid.png)

最初に迷った「Client ID がどこにも無い」は、これで解決です。Client ID は User Authentication Settings を保存して初めて生まれる、という順序でした。

### 認証を通して `.mcp.json` に登録する

取得した認証情報でアプリを登録し、OAuth ログインを済ませます。

```bash
# アプリ登録（Client ID / Secret は環境変数から。ソースにはコミットしない）
xurl auth apps add prd-mcp \
  --client-id "$CLIENT_ID" \
  --client-secret "$CLIENT_SECRET" \
  --redirect-uri "http://localhost:8080/callback"

# OAuth ログイン（ブラウザで「許可」→ localhost:8080/callback に戻る）
xurl auth oauth2 <あなたのハンドル> --app prd-mcp

# 疎通確認
xurl auth status
xurl whoami --app prd-mcp
```

認証が通れば、素の Claude Code なら登録はこれだけです。

```json
// .mcp.json（接続制限のない環境）
{
  "mcpServers": {
    "x-api": {
      "command": "xurl",
      "args": ["mcp", "https://api.x.com/mcp", "--app", "prd-mcp"]
    }
  }
}
```

:::message
devcontainer 内で OAuth する場合は、ポート `8080` を転送しておかないとブラウザの「許可」がコールバックに届きません。また、接続できる MCP が許可リストで制限された環境では、この stdio 登録が弾かれることがあります。その場合の対処法（supergateway で localhost の HTTP に変換する）は[別記事](https://zenn.dev/akaringo1109/articles/mcp-supergateway-allowlist)にまとめました。
:::

## 投稿はできない。では、何ができるのか（全24ツール）

繋がったので、全24ツールを実際に叩いて確かめました。カテゴリごとに、機能・必要ティア（目安）・実測（クレジット枯渇時）を並べます。

実測列の記号は次の意味です。

- ✅：クレジット無しでも動作
- ❌ 402：クレジット切れ（credits depleted）。クレジットまたは上位プランが必要
- ❌ 403：認証タイプ不一致
- ❌ 503：サービス停止中
- ⚠️：応答は返るが空・要追試

### ユーザー情報（5ツール）

| ツール | できること | 必要ティア | 実測 |
|---|---|---|---|
| `get_users_me` | 認証ユーザー自身のプロフィール取得 | Free | ✅ |
| `get_users_by_id` | ユーザー ID でプロフィール取得 | Basic | ❌ 402 |
| `get_users_by_username` | @handle でプロフィール取得 | Basic | ❌ 402 |
| `get_users_by_usernames` | 複数 @handle で一括取得 | Basic | ❌ 402 |
| `search_users` | 名前・プロフィール文で検索 | Basic | ❌ 402 |

Free で唯一通った `get_users_me` の実際のやりとりはこうです。MCP ツールが内部で叩くのと同じエンドポイントを、xurl で直接叩いた例です。

```bash
xurl "/2/users/me?user.fields=created_at,description,public_metrics" --app prd-mcp
```

```json
{
  "data": {
    "id": "0000000000000000000",
    "name": "あかりんご｜外資系コンサルPM × FDE",
    "username": "akaringo_ai",
    "created_at": "2023-01-25T00:09:05.000Z",
    "description": "PM/PMO・コンサルへ「FDE × 業務AI導入」を発信 🔧 外資系コンサル AI Lead。要件定義の上流から顧客に伴走し、AIを業務に溶け込ませる実践者。ものづくりの現場が好き。Zenn → https://t.co/cbdLfLkQKm",
    "public_metrics": {
      "followers_count": 0,
      "following_count": 0,
      "tweet_count": 0,
      "listed_count": 0,
      "like_count": 0
    }
  }
}
```

`id` とフォロワー数などの数値はマスクしています。`user.fields` を付けないと `id`・`name`・`username` の3つだけが返ります。

### 投稿・タイムライン（6ツール）

| ツール | できること | 必要ティア | 実測 |
|---|---|---|---|
| `get_users_posts` | 指定ユーザーの投稿一覧 | Basic | ❌ 402 |
| `get_users_timeline` | フォロー中のホームタイムライン | Basic | ❌ 402 |
| `get_users_mentions` | 自分へのメンション一覧 | Basic | ❌ 402 |
| `get_posts_by_id` | 投稿 ID でツイート取得 | Basic | ❌ 402 |
| `get_posts_by_ids` | 複数投稿 ID で一括取得 | Basic | ❌ 402 |
| `get_posts_counts_recent` | 直近7日のツイート数推移 | Pro | ❌ 402 |

「投稿・タイムライン」カテゴリにあるのは、あくまで**読み取り**のツールです。ここに投稿を作るツールがある、と期待しがちですが、ありません。

### エンゲージメント（3ツール）

| ツール | できること | 必要ティア | 実測 |
|---|---|---|---|
| `get_posts_liking_users` | いいねしたユーザー一覧 | Basic | ⚠️ 200（0件） |
| `get_posts_reposted_by` | リポストしたユーザー一覧 | Basic | ❌ 402 |
| `get_posts_quoted_posts` | 引用した投稿一覧 | Basic | ❌ 402 |

### 検索・トレンド・ニュース（4ツール）

| ツール | できること | 必要ティア | 実測 |
|---|---|---|---|
| `search_posts_all` | 全期間のツイート全文検索 | Pro | ❌ 403 |
| `get_trends_by_woeid` | 地域別トレンドランキング | Basic | ❌ 402 |
| `search_news` | ニュース記事のキーワード検索 | Basic | ❌ 402 |
| `get_news` | ニュース記事を ID で取得 | 不明 | ❌ 503 |

`search_posts_all` だけは実測が 403 です。他の23ツールと認証タイプが違うためで、後述の「仕様のはまりどころ」で扱います。

### ブックマーク（6ツール）

| ツール | できること | 必要ティア | 実測 |
|---|---|---|---|
| `get_users_bookmarks` | 自分のブックマーク一覧 | Basic | ❌ 402 |
| `create_users_bookmark` | ブックマークに追加 | Basic | ❌ 402 |
| `delete_users_bookmark` | ブックマーク削除 | Basic | ❌ 402 |
| `get_users_bookmark_folders` | フォルダ一覧取得 | Free? | ⚠️ 200（フォルダ0） |
| `create_users_bookmark_folder` | フォルダ作成 | Basic（要 Project 登録） | ❌ enrollment |
| `get_users_bookmarks_by_folder_id` | フォルダ別ブックマーク取得 | Basic | ❌ 402 |

書き込み系はこのブックマーク周辺だけで、しかも大半が Basic 以上。投稿（ツイート）の書き込みは、24ツールのどこにも入っていません。

## 料金：読み取りは高い。でも投稿は1件2円ほど

X API は 2026 年 2 月に従量課金（Pay Per Use・クレジット制）が既定になり、無料枠は廃止されました（旧 Basic / Pro のサブスクは新規受付終了、既存も従量へ移行中）。私のアカウントもこれで、クレジットが尽きていたので `get_users_me` 以外はほぼ `402 credits depleted` でした。主な単価はこうです（公式ドキュメント基準・執筆時点）。

| 操作 | 単価（従量課金） |
|---|---|
| 投稿（通常のテキスト・メディア） | **$0.015 / 件**（約2円） |
| 投稿（リンクを含む） | **$0.20 / 件**（約30円） |
| 投稿の読み取り | $0.005 / 件 |
| ユーザー情報の読み取り | $0.010 / 件 |

ここで効いてくる非対称が2つあります。ひとつは、**投稿はむしろ激安**だということ。1件 $0.015、日本円で2円ほどで、クレジットを数ドル入れれば数百〜数千回投稿できる計算です（今回の 402 は、単に残高がゼロだっただけ）。もうひとつは、**リンクを含む投稿だけ $0.20 と13倍に跳ねる**こと。記事の URL を添えて宣伝したい、という使い方はコストが変わるので頭の隅に置いておくとよいです。なお読み取りは月200万件まで、全期間検索（`search_posts_all`）は Enterprise（月 $42,000〜）が必要。「大量に読む」は高く、「たまに投稿する」はほぼ無料、という構図です。

もうひとつ大事なのは、**このクレジットは X Developer Portal 側の財布**だということ。同じ「X」でも、Grok（xAI）の API（`api.x.ai`、課金は console.x.ai）とはまったく別勘定です。Grok 側にいくらクレジットを足しても、xurl（`api.x.com`）の `402` は 1 円も解消しません。

## 本命の投稿自動化は、MCP の外にあった

投稿ツールが無い、で諦める必要はありません。xurl はもともと X API を直に叩ける CLI です。MCP のツールとして露出していないだけで、投稿エンドポイントそのものは叩けます。

```bash
xurl -X POST /2/tweets -d '{"text":"投稿内容"}' --app prd-mcp
```

まず Claude Code から叩いてみたら、返ってきたのは成功ではなくこれでした。

```json
{
  "title": "Payment Required",
  "detail": "credits depleted",
  "status": 402,
  "type": "https://api.x.com/2/problems/credits-depleted"
}
```

ここで一度手が止まります。ただ、大事なのは `403`（権限が無い）ではなく `402`（クレジット切れ）だったこと。**書き込みの経路自体は通っていて、止めているのは権限ではなく残高**でした。X API は従量課金なので、クレジットがゼロだと投稿も弾かれます（前章のとおり、これは X Developer Portal 側の財布。Grok の xAI クレジットとは別勘定です）。

そこで X Developer Portal 側にクレジットを数ドル入れて、同じコマンドをもう一度。今度は投稿の `data.id` が返ってきました。**Claude Code の会話から、実際に X へ投稿できた**わけです。1件あたり約2円。前章の「投稿はむしろ激安」が、そのまま効いてきます。

試しに、公開済みの別記事の告知を1本流してみました。文面は文体ルールで整えて、投稿は xurl。実際にタイムラインへ出たのがこれです。

![Claude Code から xurl で投稿した実際のツイート。記事カード付きでタイムラインに表示されている](/images/x-official-mcp-claude-code/xurl-post-result.png)

MCP の「ツール一覧」には投稿ツールが出てこないので、私は投稿の手順を専用の手順ファイル（スキル）にまとめて、「X へ投稿するときはこのコマンド」を固定化しました。口頭で毎回説明する指示は消えますが、ファイルに書いたルールは会話のたびに読まれるので、「これを X に出しといて」で自然に呼べるようになります。

つまり、やりたかったことは「MCP でやる」と「MCP の外の CLI でやる」に分けると成立しました。閲覧・分析は MCP のツールで、投稿は xurl の直叩きで。公式 MCP に投稿ツールが無い、という一見の行き止まりは、隣の道が開いていただけでした。

## 仕様のはまりどころ

### 認証は2種類ある（User Context と App-Only）

X の API には2つの認証タイプがあります。ツールごとにどちらを要求するかが決まっていて、ここを取り違えると 403 になります。

- **User Context（OAuth 2.0 PKCE）**：「あなた（ユーザー）として」操作する認証。xurl の `auth oauth2` で通すのはこちら。全24ツール中23ツールがこの方式です。
- **App-Only（Bearer Token）**：「アプリとして」読み取る認証。`search_posts_all` だけがこれを要求します。

先ほど `search_posts_all` だけ 403 だったのは、xurl が User Context で認証しているのに対し、このツールだけ App-Only を求めるためでした。ティアの問題（402）ではなく、認証タイプの不一致（403）です。同じ「使えない」でも原因がまったく違うので、エラーコードで切り分けます。

### エラーコードの読み方

実測で返ってきたエラーは、そのまま切り分けの手がかりになります。

- `402`：credits depleted。ティア不足。上位プランなら使える見込み。
- `403`：認証タイプ不一致。App-Only が必要なツールを User Context で叩いたケース。
- `503`：サービス停止中。エンドポイント未公開か一時障害。
- `client-not-enrolled`：OAuth アプリが Developer Portal の Project に紐付いていない状態。`create_users_bookmark_folder` はこれが返り、書き込み系の一部は Project 登録が別途要るとわかりました。

たとえば Free で `get_users_by_username` 相当を叩くと、`402` はこんな形で返ってきます。

```json
{
  "title": "Payment Required",
  "detail": "credits depleted",
  "status": 402,
  "type": "https://api.x.com/2/problems/credits-depleted"
}
```

## FAQ

**Q. 結局、MCP で X に投稿できますか？**
いいえ。投稿の作成・削除ツールは現時点の公式 MCP に非搭載です。投稿したいときは、MCP を経由せず xurl で `POST /2/tweets` を直に叩きます。閲覧・分析・ブックマーク管理までが MCP ツールの範囲です。

**Q. 無料（Free）ティアで実質何ができますか？**
`get_users_me`（自分のプロフィール取得）くらいです。他の読み取り系はほぼ Basic 以上で、Free だと `402` が返ります。まず自分のアカウント情報を取るところから、と割り切るのが現実的です。

**Q. `search_posts_all` だけ 403 になります。**
このツールだけ App-Only（Bearer Token）認証を要求するためです。xurl の User Context 認証では通りません。ティアではなく認証タイプの問題なので、Pro にしても xurl 経由のままでは 403 のままです。

## おわりに

本命の「X への投稿を Claude Code から自動化する」は、公式 MCP のツールとしては叶いませんでした。でも xurl の直叩きで、投稿という行為自体は Claude Code の会話から出せるようになりました。「MCP でやる」と「MCP の外の CLI でやる」を分けて考えたら、やりたかったことの半分は今日から動きます。

残り半分、たとえば「何を・いつ投稿するか」の判断まで委ねる自動化は、また別の設計が要ります。あなたなら、投稿の自動化はどこまで AI に任せますか。

---

**関連記事**

@[card](https://zenn.dev/akaringo1109/articles/mcp-supergateway-allowlist)

- Smart Connections MCP を supergateway 経由で動かした話（近日公開予定）
