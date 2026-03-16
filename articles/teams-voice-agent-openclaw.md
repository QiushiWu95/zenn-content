---
title: "AIにTeamsミーティングに参加させる：リアルタイム音声パイプラインの実装"
emoji: "🍊"
type: "tech"
topics: ["ai", "opensource", "automation", "agents"]
published: true
---

Microsoft Teamsには今、多くのAI機能があります。Copilotはミーティング後に会議内容を要約できます。アクションアイテムを生成できます。トランスクライブできます。ただし、それができないことの1つ—少なくとも私たちにとって機能した形式では—ウェイクワードでミーティング中にAIエージェントを呼び出し、AIが全員の声を聞き、話し返し、通話全体から文脈を覚えた状態で実際の会話をすることです。

私たちはそれが欲しかった。だからそれを構築しました。

その結果は、Claude搭載のAIエージェントをライブTeamsミーティングに統合するオープンソースの音声パイプラインです。ウェイクワードを言うと、AIが起動し、文脈の中であなたの質問を聞き、自然な音声で応答します—すべてリアルタイムで、エコーキャンセレーション付きなので、独自の音声を入力と混同しません。

GitHub: [teams-meeting-agent-public](https://github.com/QiushiWu95/teams-meeting-agent-public)

## Teams APIを使わない理由は？

Teamsには通話APIがあります。ボットをミーティングに追加できます。しかし、ミーティングボットAPIは構造化された統合—トランスクリプションサービス、録画ボット、ミーティングノートテーカー用に設計されています。公開される音声パイプラインは、低レイテンシー会話エージェントには適していません。そのようなエージェントは、すべての参加者を聞き、スピーカー識別を行い、2秒以内に応答し、割り込みを処理する必要があります。

また、具体的な制約があります。Mac（Teamsクライアントがここで実行される）とLinux GPUサーバー（推論を実行する場所）で動作します。GPUサーバーが音声認識を実行したい場所です—CUDAでfaster-whisperを実行する方が、Macでリアルタイムに実行できるものより大幅に高速で正確です。つまり、2つのマシン間に橋が必要で、SSH トンネル越しに音声が流れます。

抵抗の最も少ないパスは次のようになりました：Linux側の仮想PulseAudioデバイスを使用してTeams音声をインターセプトし、すべての処理をそこで行い、すべてを調整するカスタムWebSocketリレーを構築します。

## アーキテクチャの概要

```
Mac (Teams client)                    Linux GPU Server
┌─────────────────────┐               ┌──────────────────────────────────┐
│                     │               │                                  │
│  Microsoft Teams    │◄──────────────│  teams_speaker (null-sink)       │
│  (speaker output)   │               │  teams_virtual_mic (null-sink)   │
│  (mic input)        │               │  teams_mic_input (virtual-source)│
│                     │               │                                  │
│  bridge.py          │◄─WebSocket────│  ws_relay.py (port 8765)         │
│  (wake word,        │───speak cmd──►│                                  │
│   transcript buf)   │               │  stt_pipeline.py                 │
│                     │               │  (faster-whisper + VAD)          │
│  OpenClaw Agent     │               │                                  │
│  (Claude + memory)  │               │  tts_pipeline.py                 │
│                     │               │  (Edge-TTS → PulseAudio)         │
└─────────────────────┘               │                                  │
         │                            │  speaker_id.py                   │
         └────────SSH tunnel──────────│  (ECAPA-TDNN voiceprints)        │
              (port 8765)             └──────────────────────────────────┘
```

1回の交換のフロー：

1. Teams音声は`teams_speaker`（PulseAudioのnull-sink）を通じて再生される
2. STTパイプラインは`.monitor`ストリームをキャプチャし、VAD + スピーカーID + whisperを実行
3. トランスクリプトはWebSocket経由でMacブリッジに送信される
4. ブリッジはウェイクワードを検出→文脈をバッファリング→HTTP経由でOpenClawエージェントに送信
5. エージェントが応答を生成→ブリッジがWebSocket経由でバックに`speak`コマンドを送信
6. TTSパイプラインはEdge-TTSで合成→`teams_virtual_mic`にストリーム配信
7. Teamsは`teams_mic_input`を通じてAIが話しているのを聞く

エージェントLLMコール以外はすべてローカルで実行されます。STTはオンデバイスCUDA、TTSは最初のEdge-TTSチャンクから200ms以内にストリーム配信され、ウェイクワードから最初の音声までの往復全体は通常2秒以下です。

## PulseAudioトリック

システム全体は、ほとんどの人が見たことのないPulseAudio設定に依存しています。Linuxサーバーでは、3つの仮想オーディオデバイスを作成します：

```bash
pactl load-module module-null-sink \
    sink_name=teams_speaker \
    sink_properties=device.description=Teams_Speaker

pactl load-module module-null-sink \
    sink_name=teams_virtual_mic \
    sink_properties=device.description=Teams_Virtual_Mic

pactl load-module module-virtual-source \
    source_name=teams_mic_input \
    master=teams_virtual_mic.monitor \
    source_properties=device.description=Teams_Mic_Input
```

`teams_speaker`はnull-sink です—音声が入ると何も再生されません。ただし、PulseAudioのnull-sinksは自動的に`.monitor`ソースを作成し、音声を読み取り可能なストリームとして公開します。Teamsのスピーカー出力を`Teams_Speaker`に設定することで、`teams_speaker.monitor`が得られます—すべてのミーティング参加者を含め、Teamsが再生しているすべての内容のリアルタイムPCMストリームです。STTパイプラインはこれから読み取ります。

`teams_virtual_mic`と`teams_mic_input`は反対方向で同じ方法で機能します。TTSパイプラインは合成音声を`teams_virtual_mic`に書き込みます。モニターはそれを読取可能なソース（`teams_mic_input`）として公開し、これをTeamsのマイク入力に設定します。AIが「話すと」、Teamsはそれをマイク信号として聞きます。

これはTeamsに完全に透過的です。仮想デバイスと通信していることを知りません。API アクセスは不要です。ボット登録は不要です。Linuxサーバーは単に非常にスマートなマイクを備えたミーティング参加者のように見えます。

1つの複雑な点があります：仮想デバイスは、システムPulseAudioではなく、Chrome Remote Desktop（CRD）PulseAudioセッション内に作成される必要があります。CRDは独自の分離されたPulseAudioデーモンを実行し、標準以外のソケットパスを使用します。スタートアップスクリプトは自動的にこれを検出します：

```bash
PULSE_PATH=$(ssh "${SSH_ALIAS}" \
  "cat /proc/\$(pgrep -u \$USER pulseaudio | tail -1)/environ 2>/dev/null \
   | tr '\0' '\n' | grep PULSE_RUNTIME_PATH | cut -d= -f2")
```

実行中のPulseAudioプロセスの環境を読み取ってソケットパスを見つけ、すべての`pactl`コールの前に`PULSE_SERVER=unix:${PULSE_PATH}/native`をエクスポートします。他はすべて機能します。

## STTパイプライン：faster-whisper + VAD + エコーキャンセレーション

STTパイプラインは、最も興味深い信号処理が行われる場所です。Linuxサーバーで実行され、4つの責務があります：音声をキャプチャする、音声の境界を検出する、トランスクライブする、および独自の音声を抑制します。

**音声キャプチャ**はPulseAudioの`parec`を使用して、`teams_speaker.monitor`から16kHz単一チャネル（faster-whisperが期待する形式）でraw PCMをストリーム配信します。音声は連続した30msチャンクで入ってきます。

**音声活動検出**はSilero VADを使用します。生の音声ストリームは512サンプルフレームに分割され、モデルに供給されます。VADは十分に高速に実行され、知覚できるレイテンシーを追加しません。音声セグメントは、沈黙ギャップがトランスクリプションをトリガーするまで累積されます。

**トランスクリプション**は、CUDAで`distil-large-v3`を使用してfaster-whisperを使用します。Distil-large-v3は、Whisper large-v3の蒸留バージョン—同等の精度で、約5倍高速です。中国語と英語が混在したミーティング（私たちの主な用途）の場合、言語ヒントを必要とせずにコード切り替えを処理します。

**エコーキャンセレーション**は最もチューニングが必要だった部分です。それがなければ、AIは独自のTTS出力をトランスクライブし、AIが自分の声を聞いて応答しようとするフィードバックループが作成されます。解決策は、TTS と STT パイプライン間を調整する`SpeakingState`オブジェクトです：

```python
class SpeakingState:
    def __init__(self):
        self._speaking = False
        self._tail_suppress_until = 0.0

    def set_speaking(self, val: bool):
        self._speaking = val
        if not val:
            self._tail_suppress_until = time.time() + ECHO_TAIL_SUPPRESS_SEC

    def is_suppressed(self) -> bool:
        return self._speaking or time.time() < self._tail_suppress_until
```

TTSが再生を開始すると、`set_speaking(True)`が呼ばれます。STTは、VADでトリガーされたセグメントを処理する前に`is_suppressed()`をチェックします。TTS完了後、抑制はPulseAudioバッファを通じてまだ流れている音声をキャッチするために、設定可能なテールウィンドウ（0.8秒を使用）で続きます。

**バージインイン検出**は割り込みストーリーのもう半分です。AIが話している間に人間が話し始めると、AIを途中で止めたいのです。これはVADが遅いため、リアルタイム割り込みトリガーには遅すぎるため、エネルギーベースの検出で行われます：

```python
# Fast 200ms window vs slow 3.2s baseline
fast_energy = rms(audio[-200ms:])
slow_energy = rms(audio[-3200ms:])
if fast_energy > slow_energy * BARGE_IN_RATIO:
    trigger_interrupt()
```

高速ウィンドウのエネルギーが低速ベースラインの比率を上回る場合、バージインを通知します。TTSパイプラインは割り込みコマンドを受け取り、すぐに再生を停止します。

## スピーカー識別：ボイスプリント照合

ミーティングで言われたすべてが、AIに到達する必要があります。ミーティングホストだけがエージェントを呼び出すことができるようにしたい、またはチームの人だけ呼び出せるようにしたいかもしれません。スピーカー識別がこれを解決します。

実装ではspeechbrainのECAPA-TDNNモデルを使用します—192次元スピーカー埋め込みを生成するスピーカー検証モデルです。各音声セグメントについて、埋め込みを抽出し、コサイン類似度を使用して登録済みボイスプリントのセットと照合します：

```python
similarity = cosine_similarity(embedding, voiceprint)
if similarity > SPEAKER_MATCH_THRESHOLD:  # 0.30 by default
    return "matched_speaker_name"
```

0.30のしきい値は意図的に低い—偽陽性（未知のスピーカーを既知として認識）よりも正当なユーザーを見逃すよりは、偽陽性を許容します。エージェントの起動を特定の人だけに制限する信頼の境界の場合、これを上げます。

モデルは言語に依存しません。中国語と英語のスピーカーで同じくらい機能します。これは私たちのミーティングにとって重要です。推論はCUDAで10ms以下で実行されるため、トランスクリプションパイプラインに無視できるレイテンシーを追加します。

スピーカー識別はWebSocketリレーを通じて信頼層として流れます。トランスクリプトは、`verified`（既知のボイスプリントに一致）または`untrusted`（未知のスピーカー）でタグ付けされます。ブリッジはこれを使用して、ウェイクワードを起動できるユーザーをフィルタリングします。

## ウェイクワード検出とブリッジ

ブリッジはMacで実行されます。そのジョブは、LinuxサーバーとOpenClawエージェントの間に位置し、何がどこに送られるかについてのルーティング決定を行うことです。

Linuxサーバーからトランスクリプトが到着すると、ブリッジはregexパターンマッチングを使用してウェイクワードをチェックします。ウェイクワードリストは設定可能です—私たちの設定では「hey claude」「hey agent」および数個の中国語の同等語を含みます。ブリッジはまた「プレゼンテーションモード」をサポートし、ウェイクワード要件が緩和され、すべてのトランスクリプトがフロー配信されます。

ウェイクワードが検出されると、ブリッジは「engaged」モードに移行します：後続のトランスクリプトをバッファリングし、複数のスピーカーの文脈を累積し、自然な一時停止でバッファをOpenClawエージェントのHTTP APIにフラッシュします。つまり、AIは1文を聞くだけではなく、呼び出された時に何が議論されていたかについての複数ターンの文脈ウィンドウを取得します。

ブリッジはまた応答パスを処理します。OpenClawが応答を生成すると、ブリッジはLinuxサーバーのWebSocketリレーに`speak`コマンドを送信します：

```json
{"cmd": "speak", "text": "Here's the answer to your question..."}
```

TTSパイプラインがこれを拾い、合成し、仮想マイク経由で再生します。

接続の回復力はここで重要です—ミーティングは数時間続くことができます。ブリッジは指数バックオフと自動再接続を実装し、ハートビートpingでサイレント切断を検出し、ドロップされたトランスクリプトが発生する前に検出します。

## TTSパイプライン：ストリーミング合成

Edge-TTSは、自然な音声音声を生成する無料の高品質TTSサービスです。制限は、ストリーミングしないことです—完全なテキストが合成されるまで待機します。

チャンク生成でこれを回避します。Edge-TTSは内部的にMP3データをストリーム配信し、`edge-tts` Pythonライブラリはこれを公開します。MP3ストリームをffmpegにパイプして、その場でPCMに変換し、チャンクが到着したときにPulseAudioに書き込みます：

```python
async for chunk in communicate.stream():
    if chunk["type"] == "audio":
        process.stdin.write(chunk["data"])  # ffmpeg stdin
        # ffmpeg is already decoding and writing to PulseAudio
```

結果は、最初に聞こえる音声が`speak`コマンド到着から200～300ms以内に再生されることです。人々がすでに話していて応答を待っているミーティング文脈では、このレイテンシーはほとんど知覚できません。

TTSパイプラインはSTTで使用される同じ`SpeakingState`と統合されます。各チャンクを書き込む前に、割り込み信号をチェックします。オーディオが生成されている間にバージインが検出された場合、再生は途中で停止し、パイプラインはブリッジに確認応答を送信して、エージェントが応答が途中で切られたことを知っています。

## ワンコマンドスタートアップ

システム全体—SSHトンネル、リモートPulseAudio設定、tmuxセッション内の両方のプロセス—単一スクリプトで起動します：

```bash
./start_meeting.sh
```

スクリプトは完全なオーケストレーションを処理します：
1. SSHトンネルがすでにポート8765で実行されているかどうかを確認してください。必要な場合は作成してください
2. LinuxサーバーへのSSH接続、CRD PulseAudioソケットパスを検出
3. 3つの仮想オーディオデバイスをロードします（べき等性—既にロードされている場合はスキップ）
4. リモートにランチャースクリプトを作成し、ソケットパスでのシェルエスケープの問題を避けます
5. リモートtmuxセッション（`teams-voice`）で`main_linux.py`を開始します
6. WebSocketが接続を受け入れるまでポーリングします
7. ローカルtmuxセッション（`teams-bridge`）で`bridge.py`を開始します

起動後、出力は実行中のものと、Teamsで設定する内容を正確に示します：

```
══════════════════════════════════════════════════
  Teams Voice Agent — RUNNING
══════════════════════════════════════════════════
  Session key  : voice-meeting-20260313-1430
  Bridge tmux  : teams-bridge  (local)
  Remote tmux  : teams-voice   (on gpu-server)
  Tunnel       : localhost:8765 → gpu-server:8765

  ⚠️  Set Teams audio: Speaker=Teams_Speaker, Mic=Teams_Mic_Input
══════════════════════════════════════════════════
```

唯一の手動ステップは、Teamsのオーディオデバイスを仮想デバイスに設定することです。その後、すべてはハンズフリーです。

## 実際に

典型的なミーティングでは、パイプラインはアクティブになるまで完全に無音です。STTは継続的に実行され、スピーカーIDは全員にタグ付けしていますが、エージェントには何もフローしません。AIはリッスンしていますが、存在しません。

誰かが「hey claude、何か検索して」と言うと、ブリッジはウェイクワードをキャッチし、次の数文の文脈をバッファリングし、全体をエージェントに送信します。エージェントは2秒以下で応答し、声は全員のスピーカーを通じて来ます（AIは仮想マイクを通じて話しているため）、その後パイプラインは再度静かになります。

主に技術的なディスカッション中の知識検索—ドキュメントの参照、ノートのクロスリファレンス、通話で何が決まったかをまとめることに使用します。エージェントはOpenClawメモリシステムにアクセスでき、同じプロジェクトについて過去のミーティングから文脈を取得できます。これはミーティングでまだ人々を驚かせる部分です：AIは質問に答えるだけでなく、先週の通話からの決定を参照します。

バージインは予想以上にうまく機能します。AIが中盤の応答中に誰かが話し始めると、約半秒以内に停止します。すでにPulseAudioバッファにあった音声からの簡単なアーティファクトがありますが、破壊的ではありません。

主な制限は、Linuxサーバーで Chrome Remote Desktopが実行されている必要があることです。CRDは、仮想デバイスが存在するPulseAudio環境を作成します。それなしに、スタートアップスクリプトを別のPulseAudio設定で機能するように適合させる必要があります。コアパイプラインコードは気にしません—正しいデバイス名が存在する必要があるだけです。

## 別の方法

現在のアーキテクチャは、すべての信号処理をLinuxサーバーに配置します。これは私たちにとって有意ですが、ユニバーサルではありません。GPU対応のMacがあるか、より単純なセットアップが必要な場合、STTパイプラインはMLX-Whisperを使用してローカルで実行できます—Apple Siliconで同等の精度、リモートサーバー不要です。

スピーカー識別は現在、事前登録されたボイスプリントに基づいています。より堅牢なアプローチは、pyannoteのようなものを使用して、登録を必要とせずにダイアライゼーションスタイル「誰がいつ話したか」識別を行うことです。これにより、エージェントは登録されていない人のためにすら会議の貢献をアトリビューションできるようになります。

ウェイクワードシステムはregexベースで、機能しますが、アクセントと音声認識エラーに対してはもろいです。適切なウェイクワードモデル（openWakeWordのような）は、特に非英語のアクティベーションでより信頼性が高いでしょう。

## 結論

ここで興味深いエンジニアリングはAI部分ではありません—Claudeがそれを処理します。オーディオ配管です：正しいオーディオを正しい時間に正しい場所に取得し、AIが独自の音声を入力と混同することなく、会話を気まずくするほど十分なレイテンシーを追加することなく。

PulseAudio仮想デバイスは、この種のオーディオルーティングに本当に強力です。null-sink + monitorパターンは、アプリケーションの内部パイプラインにパッチを当てることなく、オーディオストリームをインターセプトするクリーンな方法です。もっと多くの人がそれが存在することを知るべきです。

完全なソースは[github.com/QiushiWu95/teams-meeting-agent-public](https://github.com/QiushiWu95/teams-meeting-agent-public)にあります。READMEには、使用するハードウェア構成（GPUサーバー + Tailscale経由のMac）のセットアップ手順がありますが、パイプライン自体は適度な努力で他のセットアップに適応する必要があります。

---

*この記事は [claw-stack.com](https://claw-stack.com/ja/blog/teams-voice-agent) で最初に公開されました。OpenClawはオープンソースのAIエージェントランタイムです。*
