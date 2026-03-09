---
title: "OpenClawでiMessageを安定させる：3つの問題と解決策"
emoji: "📱"
type: "tech"
topics: ["imessage", "macos", "opensource", "automation"]
published: true
---


OpenClawはiMessageを通信チャンネルとして利用できる——テキストをAIエージェントに送れば、エージェントが返信してくれる。シンプルに聞こえるが、Mac miniで24/7稼働させたところ、診断に数週間かかった3つの信頼性問題が発覚した。何が起きて、どう修正したかを解説する。

## 構成の概要

OpenClawのiMessageプラグインは、FSEventsを使って`~/Library/Messages/chat.db`を監視している。新しいメッセージが届くと、macOSが`chat.db`に書き込み、ウォッチャーが変更を検出し、ゲートウェイがメッセージを処理する。

理論上は即時に動作する。実際には、3つの異なる方法で壊れる。


## 問題1：アイドル時にメッセージが最大5分遅延する

**症状**：メッセージを送ると、スマホには「配信済み」と表示されるが、エージェントは3〜5分間応答しない。その後、突然すべてを一度に処理する。

**根本原因**：macOSの電源管理は、バックグラウンドプロセスに対してFSEventsを集約する。LaunchAgentのplistに`ProcessType=Interactive`を設定し、`caffeinate`を実行していても、カーネルは低活動時に`chat.db`のvnodeイベントをバッチ処理する。`imsg rpc`サブプロセスはファイルを監視しているが、macOSは「このプロセスはしばらく動いていない、ファイル通知をまとめよう」と判断してしまう。

**診断が難しい理由**：メッセージはすでに`chat.db`に書き込まれている——遅れているのはメッセージ自体ではなく、*通知*だ。そのため、アクティブな使用中は完全に正常動作するが、マシンがアイドル状態になると静かに失敗する。

**修正策**：15秒ごとに`chat.db`をチェックし、新しい行が検出されたときにファイルを`touch`して新鮮なFSEventを生成するポーリングスクリプト：

```javascript
#!/usr/bin/env node
// imsg-poller.mjs — Polls chat.db for new messages and wakes FSEvents watcher


const CHATDB = join(homedir(), 'Library/Messages/chat.db');
const INTERVAL = 15000; // 15 seconds

function getMaxRowid() {
  try {
    return execSync(
      `/usr/bin/sqlite3 "${CHATDB}" "SELECT MAX(ROWID) FROM message;"`,
      { timeout: 5000, encoding: 'utf8' }
    ).trim() || '0';
  } catch { return '0'; }
}

let lastRowid = getMaxRowid();
if (lastRowid === '0') {
  console.error('ERROR: Cannot read chat.db — check Full Disk Access');
  process.exit(1);
}

console.log(`imsg-poller started. ROWID: ${lastRowid}, interval: ${INTERVAL}ms`);

setInterval(() => {
  const current = getMaxRowid();
  if (current !== '0' && current !== lastRowid) {
    console.log(`New message (ROWID ${lastRowid} -> ${current}), touching chat.db`);
    try {
      const now = new Date();
      utimesSync(CHATDB, now, now);
    } catch (e) {
      console.error(`touch failed: ${e.message}`);
    }
    lastRowid = current;
  }
}, INTERVAL);
```

**なぜbashではなくNode.js？** まずbash版を試したが、launchdが起動した`/bin/bash`プロセスはフルディスクアクセス（TCC）を継承しない。`stat`コマンドは動くが、`sqlite3`は「認証が拒否されました」を返す。`/opt/homebrew/bin/node`を使うと、ゲートウェイと同じTCA付与からFDAを継承するため動作する。

**デプロイ方法**：`KeepAlive: true`のLaunchAgentとして実行：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>ai.openclaw.imsg-poller</string>
  <key>ProgramArguments</key>
  <array>
    <string>/opt/homebrew/bin/node</string>
    <string>/path/to/imsg-poller.mjs</string>
  </array>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>EnvironmentVariables</key>
  <dict>
    <key>HOME</key><string>/Users/youruser</string>
  </dict>
  <key>ThrottleInterval</key><integer>10</integer>
</dict>
</plist>
```

**OpenClawのアップデート後も有効？** はい——独立したlaunchdジョブのため。


## 問題2：iMessage経由の画像送信が「パスが許可されていない」で失敗する

**症状**：エージェントがiMessageで受信した画像を送ろうとすると、「ローカルメディアパスが許可されたディレクトリ下にありません」エラーが発生する。画像は`~/Library/Messages/Attachments/...`に存在するが、OpenClawのメディアサンドボックスがブロックする。

**根本原因**：OpenClawの`buildMediaLocalRoots()`関数が、メディアファイルアクセスを許可するディレクトリを定義している。ワークスペース、一時ディレクトリ、サンドボックスは含まれているが、`~/Library/Messages/Attachments/`は含まれていない。エージェントがiMessageで受信した画像を転送・処理しようとすると、パスが拒否される。

**修正策**：Messagesの添付ファイルディレクトリを許可ルートに追加するパッチスクリプト：

```bash
#!/usr/bin/env bash
# patch-imessage-attachments.sh
# Adds ~/Library/Messages/Attachments to allowed media roots
# Re-run after every `npm update -g openclaw`

DIST="/opt/homebrew/lib/node_modules/openclaw/dist"

patched=0
for f in "$DIST"/ir-*.js; do
  [ -f "$f" ] || continue
  if grep -q "buildMediaLocalRoots" "$f" && \
     ! grep -q "Messages/Attachments" "$f"; then
    sed -i '' 's|path.join(resolvedStateDir, "sandboxes")|path.join(resolvedStateDir, "sandboxes"),\n\t\tpath.join(os.homedir(), "Library/Messages/Attachments")|' "$f"
    echo "Patched: $(basename $f)"
    patched=$((patched + 1))
  fi
done

echo "Done. Patched: $patched files"
echo "Run: openclaw gateway restart"
```

**OpenClawのアップデート後も有効？** いいえ——コンパイル済みJSファイルは上書きされる。**アップデートのたびに再実行が必要。**


## 問題3：macOSアップデート後にフルディスクアクセスが静かに取り消される

**症状**：iMessageが完全に動作しなくなる。メッセージが受信されず、ゲートウェイログに意味のあるエラーも出ない。エージェントはオンラインに見えるが、何も聞こえていない。

**根本原因**：macOSのシステムアップデート（時にはマイナーなセキュリティパッチ）により、TCC（透明性、同意、制御）の権限がリセットされることがある。これが起きると、`imsg`バイナリがフルディスクアクセスを失い、`~/Library/Messages/chat.db`が読めなくなる。ゲートウェイログには以下が表示される：

```
permissionDenied(path: "~/Library/Messages/chat.db",
  underlying: authorization denied (code: 23))
```

私たちのログでは、**2026年2月13日**と**2月24日**に発生し、どちらもmacOSのアップデートと時期が一致した。

**修正策**：残念ながら手動対応のみ。

1. ゲートウェイのエラーログを確認：
   ```bash
   grep "permissionDenied" ~/.openclaw/logs/gateway.err.log | tail -5
   ```

2. `code: 23`が見えたら：
   **システム設定 → プライバシーとセキュリティ → フルディスクアクセス**

   `imsg`（またはゲートウェイを実行しているTerminal / iTerm）にFDAが有効になっているか確認する。正しく見えるのに動かない場合は、一度オフにしてから再度オンにする。

3. 確認：
   ```bash
   /opt/homebrew/bin/imsg chats --limit 1
   # エラーではなく、最近のチャットが返ってくるはず
   ```

4. 再起動：
   ```bash
   openclaw gateway restart
   ```

**OpenClawのアップデート後も有効？** はい——TCCの権限はシステムレベル。ただしmacOSのアップデートでリセットされることがある。


## アップデート後のチェックリスト

`npm update -g openclaw`を実行するたびに、以下を行う：

```bash
# 1. パッチを再適用（アップデートで上書きされたため）
bash ~/.openclaw/autopatch/patch-imessage-attachments.sh

# 2. ゲートウェイを再起動
openclaw gateway restart

# 3. iMessageが動作するか確認
/opt/homebrew/bin/imsg chats --limit 1
```

macOSのアップデート後は、フルディスクアクセスの権限も確認すること。


## これらはOpenClawがアップストリームで修正すべき？

**問題1**（FSEventsの集約）はmacOSカーネルの動作——OpenClaw自体での修正は難しい。ポーリングが正しい回避策であり、OpenClawはオプションコンポーネントとして提供することを検討できる。

**問題2**（添付ファイルパス）は明らかなバグ/見落とし。iMessageプラグインが有効な場合、`~/Library/Messages/Attachments/`はデフォルトで許可ルートに含まれるべきだ。アップストリームで1行修正すれば済む。

**問題3**（TCCリセット）はAppleの問題だ。OpenClawにできることは、それを検知してより明確なエラーメッセージを出力することくらい。


## 学んだこと

1. **「自分のマシンで動く」は常時稼働エージェントには不十分。** これらのバグは、数日間の継続稼働やシステムアップデートの後にしか現れない。発見するには、エージェントを数週間24/7稼働させる必要がある。

2. **macOSはヘッドレスサーバー向けに設計されていない。** 電源管理、TCC、FSEventsの集約——すべてが人間がスクリーンの前に座っていることを前提としている。Mac miniでAIエージェントを動かすには、あらゆるレベルでOSと戦う必要がある。

3. **パッチディレクトリを維持する。** `~/.openclaw/autopatch/`にスクリプトとREADMEを置き、すべてのパッチを記録している。アップデートが来たら全部実行する。エレガントではないが、確実だ。

4. **すべてをログに残す。** ポーラーは毎回の`touch`をログに記録し、ゲートウェイはすべての権限エラーをログに残す。これがなければ、「なぜメッセージが届かなかったのか」を今でもデバッグしていただろう。

---

*この記事は [claw-stack.com](https://claw-stack.com/ja/blog/imessage-reliability) で最初に公開されました。OpenClawはオープンソースのAIエージェントランタイムです。*
