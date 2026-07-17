---
title: "許可リストにない stdio の MCP を、supergateway で Claude Code に繋ぐ"
emoji: "🔌"
type: "tech"
topics: ["claudecode", "mcp", "supergateway", "devcontainer", "docker"]
published: true
published_at: 2026-07-19 19:00
---

Claude Code で自分用の仕組みを作りながら、外資系コンサルで AI 導入をリードしているあかりんごです。
`.mcp.json` に MCP サーバーを登録したのに、Claude Code の一覧に出てこない。エラーも出ない。そんな経験はありませんか。

これは、別記事で「X への投稿を MCP で自動化したくて」公式 MCP を Claude Code に繋いだとき、接続でぶつかった話の裏側です。X 公式 MCP で実際に何ができるか（全24ツールの実測と、投稿の自動化）を確かめた本編はこちら。

@[card](https://zenn.dev/akaringo1109/articles/x-official-mcp-claude-code)

接続できる MCP が許可リストで絞られた環境で、リストにない stdio の MCP をどう通したか。supergateway を挟むという定番の一手を、つまずきどころ込みで書きます。

:::message
この記事は次の環境が前提です。

- 接続できる MCP サーバーが許可リストで絞られている Claude Code 環境（会社支給の環境などで見かけます）
- devcontainer（Docker）内で Claude Code を動かしている

接続できる MCP に制限がない環境では、この記事の症状は起きません。stdio の MCP をそのまま `.mcp.json` に書けば繋がります。
:::

## 症状：登録したのに、MCP が一覧に出てこない

繋ぎたかったのは、[X が公式に出した MCP サーバー](https://zenn.dev/akaringo1109/articles/x-official-mcp-claude-code)（`@xdevplatform/xurl`）でした。`.mcp.json` に stdio 型で登録したところ、Claude Code の MCP 一覧に出てきません。エラーも出ません。ただ、一覧に出てこないだけでした。

原因は、この環境が使える MCP サーバーを許可リスト（allowlist）で絞っていたことでした。設定はこんな形です。

```json
{
  "enableAllProjectMcpServers": false,
  "allowedMcpServers": [
    { "serverName": "playwright" },
    // ...許可リスト（xurl は載っていない）
    { "serverUrl": "http://localhost*" }
  ]
}
```

`enableAllProjectMcpServers` が `false` の環境では、許可リストに載っていない MCP は、エラーも出さずにサイレントに無視されます。これが地味に厄介で、自分の設定ミスを疑って `.mcp.json` を何度も見直す時間を溶かしました。

ただ、リストをよく見ると `http://localhost*` が許可されています。ここが突破口でした。名前で登録された stdio の xurl は弾かれます。でも localhost の HTTP エンドポイントなら通ります。だったら、xurl を localhost の HTTP サーバーに化けさせればよいはずです。

## なぜ通らないのか：MCP を「話し方」と「場所」で見る

先に進む前に、なぜ localhost の HTTP なら通るのかを整理しておきます。ここを押さえておくと、次に別の MCP で詰まったときにも同じ考え方で抜けられます。

MCP の接続は、混ざりやすい2つの軸に分けると見通しがよくなります。

- **話し方（トランスポート）**：起動して標準入出力で筆談する **stdio** か、URL に HTTP でアクセスする **HTTP** か
- **場所**：自分のマシンやコンテナ内で動く **ローカル** か、インターネットの先で動く **リモート** か

| | stdio | HTTP |
|---|---|---|
| **ローカル** | playwright、素の xurl | supergateway 出力（localhost:3335） |
| **リモート** | （原理的に存在しない） | ホステッドな MCP（例：Figma） |

この2軸で見ると、許可リストを通す勘所は「stdio か HTTP か」ではないことがわかります。許可リストは、サーバー名か URL で照合しています。名前が載っている stdio（playwright）はそのまま通り、載っていない stdio（xurl）は弾かれます。載っていない stdio を通す道が、「話し方を stdio から HTTP に変え、場所はローカルのまま `http://localhost*` の条件に合わせる」ことでした。

```text
[stdio + ローカル]  ──supergateway──▶  [HTTP + ローカル]
 xurl mcp                             http://localhost:3335/mcp
 （名前が許可リストに無く弾かれる）      （http://localhost* で許可される）
```

## supergateway で stdio を localhost の HTTP に変える

[supergateway](https://github.com/supermachine-ai/supergateway) は、stdio 型の MCP サーバーを HTTP エンドポイントに変換するプロキシです。以前、同じ環境で Smart Connections の MCP を動かしたときにも使いました。今回もそのパターンを流用します。

### 変換の流れと `.mcp.json`

Claude Code からは localhost の HTTP を叩き、supergateway が内部で stdio の xurl に橋渡しします。

```text
Claude Code
  │  Streamable HTTP POST
  │  http://localhost:3335/mcp
  ▼
supergateway（port 3335）
  │  stdio
  ▼
xurl mcp https://api.x.com/mcp
  │  HTTPS
  ▼
https://api.x.com/mcp
```

```json
// .mcp.json
{
  "mcpServers": {
    "x-api": {
      "type": "http",
      "url": "http://localhost:3335/mcp"
    }
  }
}
```

### SSE だと2回目のコールで死ぬ

supergateway の HTTP 出力には SSE と Streamable HTTP の2モードがあります。最初は何となく SSE を選びました。これが失敗でした。

最初のツール呼び出しは成功します。ところが2回目以降が `503: No active SSE connection for session XXX` で落ちます。

原因を追うと、Smart Connections 対応のときに入れた再接続パッチ（`server.close()` を呼ぶ処理）が、内部の stdio パイプごと閉じて xurl を終了させていました。SSE は「1つのサーバーに N クライアントがぶら下がる」ステートフルな設計で、セッションを維持する前提です。そこにパイプを閉じるパッチが噛み合わず、xurl が死ぬと以降の接続も全滅する、という構図でした。

解決は `--outputTransport streamableHttp` への切り替えです。Streamable HTTP はリクエストごとに完結するステートレス設計で、セッション状態を持ちません。だから並列コールでも壊れません。切り替えたあと、3連続リクエストで xurl のプロセス ID が変わらないこと、全レスポンスが正常なことを確認しました。

外部 API をブリッジする用途では、SSE ではなく Streamable HTTP を選ぶ。これは今回いちばんの実感でした。SSE は長期接続を維持したいストリーミング用途には向きますが、xurl のように「1回叩いて結果をもらう」ブリッジには、ステートレスな Streamable HTTP のほうが素直に噛み合います。

### supervisor スクリプト

落ちても勝手に立ち上がるよう、supervisor スクリプトで包みます。

```bash
# sgw-xurl.sh（supervisor）
node supergateway \
  --stdio "xurl mcp https://api.x.com/mcp --app prd-mcp" \
  --outputTransport streamableHttp \
  --streamableHttpPath /mcp \
  --port 3335 \
  --logLevel none
```

## 一度作って、あとは放置できるようにする

ここまでで繋がりました。ただ、繋がっただけだと、常駐プロセスの二重起動や、コンテナを作り直すたびの吹き飛びに毎回悩まされます。「一度作れば意識から消える」状態まで持っていきます。

### `pgrep -f` が自分を検知して自滅する

supervisor を常駐させるとき、二重起動を防ぐガードを書きます。ここで既視感のある罠を踏みかけました。

`pgrep -f "sgw-xurl.sh"` で「もう動いているか」を判定しようとすると、実行中のコマンド文字列に自分自身のスクリプト名が含まれるため、pgrep が自分のシェルを拾って「既に動いている」と誤検知します。結果、起動したいのに起動しません。

これは Smart Connections を組んだときにも踏んだ罠で、「あ、これ前も見た」と気づけたのが唯一の救いでした。対策は、プロセス名ではなくポートの存在で判定することです。

```bash
# ポート 3335 が開いていなければ supervisor を起動
ss -tln 2>/dev/null | grep -q ':3335 ' || setsid /bin/bash sgw-xurl.sh > /dev/null 2>&1 &
```

プロセスを名前で探すと自分を数えてしまいます。ポートは1つしか開かないので、ポートの有無で見るほうが確実です。地味ですが、常駐プロセスのガードを書く人は一度はハマるはずです。

### コンテナを作り直しても壊れないようにする

devcontainer は気軽に作り直せるのが利点です。でも、リビルドのたびに認証もインストールも吹き飛ぶと運用になりません。ここは永続化しておきます。

xurl はトークンを `~/.xurl`（単一ファイル）に置きます。これを名前付き Docker ボリューム上に逃がして、シンボリックリンクを張ります。

```bash
# postCreateCommand で自動セットアップ
if [ ! -L ~/.xurl ]; then
  [ -f ~/.xurl ] && mv ~/.xurl ~/.claude/xurl-store || true
  ln -sf ~/.claude/xurl-store ~/.xurl
fi
```

`~/.claude` を名前付きボリュームにマウントしておけば、この置き場はリビルドを越えて残ります。supergateway 本体も同じボリュームに入れて、supervisor 側に「消えていたら入れ直す」自己修復を仕込んでおきます。

この「トークン置き場」「ツール置き場」を永続ボリュームに寄せる設計は、xurl に限らず他の MCP でもそのまま使い回せます。1回作れば、次からは devcontainer の起動フックが勝手に立ち上げてくれるので、意識から消えます。

## おわりに：制限のある環境では、道具の形のほうを変える

今回の肝は、supergateway そのものより、「繋ぎ方を変えて、既にある入口に合わせる」という発想でした。

`http://localhost*` は、多くの環境でローカル用にあらかじめ開いている入口です。そこに xurl の話し方（stdio → HTTP）のほうを寄せて、素直に収まる形にしました。環境の側をどうこうするのではなく、道具の側の形を変える、というアプローチです。

この「開いている入口に道具を寄せる」発想は、supergateway に限らず、接続できる MCP が絞られた環境で何かを繋ぐときの定石として使い回せます。次に別の stdio ツールで詰まっても、同じ考え方でほぼ通せるはずです。あなたの環境では、繋ぎたい stdio の MCP はもう決まっていますか。

---

**関連記事**

@[card](https://zenn.dev/akaringo1109/articles/x-official-mcp-claude-code)

- Smart Connections MCP を supergateway 経由で動かした話（近日公開予定）
