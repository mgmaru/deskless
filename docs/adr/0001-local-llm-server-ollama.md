# ADR-0001: ローカルLLMの推論サーバーにOllamaを採用する

| 項目 | 内容 |
| --- | --- |
| ステータス | 承認 |
| 日付 | 2026-09-25 |
| 関連 | [vmware_ubuntu_dev_vm_architecture.md](../vmware_ubuntu_dev_vm_architecture.md)（§17 GPU処理はWindows側に集約、§18 Ubuntu VMからOllamaを利用）、[ROADMAP.md](../../ROADMAP.md)（Step 6・7）、[ollama-https-tailscale.md](../reference/ollama-https-tailscale.md)（HTTPS化とTailscaleの設定） |

> 各ソフトウェアの機能・バージョンは2026-09-25時点の調査結果。

---

## 背景

Ubuntu Server VM上で開発するアプリやツールから、Windowsホストに挿したRTX 3080（VRAM 10GB）でLLMを動かし、HTTP APIで呼ぶ構成にしている。

```text
Ubuntu VM ──HTTP（Tailscale）──→ Windows ──→ 推論サーバー ──→ RTX 3080
```

推論サーバーは設計の段階からOllamaを置いていたが、ほかの候補と比べて選んだわけではなかった。構築（ROADMAP Step 6）に入る前に候補を洗い出して比較し、改めて決める。

---

## 判断の基準

この構成の前提から、推論サーバーには次の条件が求められる。

| 条件 | 理由 |
| --- | --- |
| WSL2を使わずにWindowsで動く | GPUはWindows側に集約している。さらに、VMwareを本来の速度で動かすためにWindowsのハイパーバイザーを無効化する案をROADMAPで検討中で、無効化するとWSL2とDocker Desktopが使えなくなる |
| 使っていない間にVRAMを自動で空ける | 10GBのGPUをComfyUIと共有する。LLMがVRAMを持ち続けると画像生成ができない |
| ログインなしで自動起動し、SSHから操作できる | Windows Update後の再起動から、誰も触らずに戻る必要がある（「落ちない」より「戻る」）。操作は外出先から `ssh win` で行う |
| VMからHTTPで呼べ、OpenAI互換APIがある | 対応しているクライアントやライブラリが多い |
| 1人で保守できる | 個人の常時稼働環境であり、構成は単純なほどよい |

---

## 検討した選択肢

1. **Ollama**（当初の構成）
2. **llama.cpp の `llama-server`**
3. **LM Studio（llmster）**
4. **vLLM / SGLang**
5. **TabbyAPI（ExLlamaV3）**

次のものは、条件に明らかに合わないため比較の対象から外した。

| 候補 | 外した理由 |
| --- | --- |
| LocalAI / Docker Model Runner | WindowsではDocker Desktopが必要で、Docker DesktopはWSL2に依存する。READMEの「Docker Desktopは使わない」方針とも合わない |
| KoboldCpp | 中身はllama.cppで、チャットや創作向けの機能が中心。APIサーバーとして使うならllama.cppを直接使うほうが素直 |
| Foundry Local（Microsoft） | Windowsで直接動くが、使えるのはMicrosoftが用意したカタログのモデルが中心 |
| llama-swap | llama.cppの前に置いてモデルを切り替えるプロキシ。llama.cpp本体のルーターモードで代替できるようになった |

---

## 比較

| 選択肢 | WSL2なしでWindowsで動くか | VRAMの自動解放 | 認証 | OpenAI互換 | Anthropic互換 | 手軽さ |
| --- | --- | --- | --- | --- | --- | --- |
| **Ollama** | ○ | ○ 既定で5分使わなければ解放 | **×** | ○ | ○ v0.14.0〜 | ◎ |
| llama.cpp `llama-server` | ○ CUDA版の配布あり | ○ `--sleep-idle-seconds`、ルーターモード | ○ `--api-key` | ○ | ○ | ○ |
| LM Studio（llmster） | ○ | ○ 必要時に読み込み、一定時間後に解放 | ○ | ○ | ○ 0.4.1〜 | ○ |
| vLLM / SGLang | × WSL2・Docker・非公式版のみ | × 起動時にVRAMの大半を確保し続ける | ○ | ○ | － | △ |
| TabbyAPI（ExLlamaV3） | ○ | △ | ○ | ○ | － | △ |

### Ollama

- **良い点**
  - `ollama pull` だけでモデルを管理でき、SSHからの操作が簡単
  - 使っていないモデルは既定で5分後にVRAMから降りる（`keep_alive`）。ComfyUIとGPUを共有しやすい
  - OpenAI互換APIに加え、v0.14.0（2026年1月）からAnthropic互換APIも使える
- **弱い点**
  - **認証がない**。届く相手なら誰でも推論でき、`/api/pull` でのモデル取得や `/api/delete` での削除もできる
  - **VRAMが24GB未満だと、コンテキスト長が既定で4kトークン**。コーディングやエージェント用途には足りない（公式ドキュメントは64k以上を推奨）
  - モデルの一部をCPU側で動かすといった細かい調整は、llama.cppほど自由にできない。新しいモデルへの対応がllama.cpp本体より遅れることがある
  - インストーラー版はログイン後に起動するアプリ。ログインなしで動かすには、公式が用意しているzip版（CLIとGPUライブラリのみ）をサービスやタスクとして登録する

### llama.cpp `llama-server`

- **良い点**
  - OllamaやLM Studioの中で動いている推論エンジンの本家。新しいモデルへの対応が一番早い
  - `--api-key` で認証をかけられる
  - 2025年12月に入ったルーターモードで、複数モデルの自動読み込み・切り替え・古いものからの退避ができる。`--sleep-idle-seconds` で、使っていない間のVRAM解放もできる
  - `--n-cpu-moe` などで、10GBに収まらないMoEモデル（gpt-oss-20bなど）の一部をCPU側に置いて動かせる
  - OpenAI互換（Chat Completions / Responses）とAnthropic互換（`/v1/messages`）の両方がある
- **弱い点**
  - モデルの入手や、コンテキスト長・GPUに載せる層の数などの設定を自分で書く必要がある
  - ほぼ毎日リリースされていて、安定版の区別がない。更新は自分で行う
  - サービスとしての登録はOllamaと同じく自分で行う

### LM Studio（llmster）

- **良い点**
  - 0.4.0（2026年1月）で、GUIなしで動くデーモン `llmster` が登場した。`lms` コマンドでSSHから操作できる
  - 必要になったときにモデルを読み込み、一定時間使わなければ解放する
  - 認証（permission keys）とAnthropic互換APIがある
  - 2025年7月から、仕事での利用も無料
- **弱い点**
  - 本体のソースコードは公開されていない
  - llmsterをWindowsでログイン前から動かす方法が、公式ドキュメントに見当たらない（アプリ版の自動起動は「ログイン時」）
  - 機能がOllamaとほぼ重なり、乗り換える決め手がない

### vLLM / SGLang

- **良い点**
  - 同時にたくさんのリクエストをさばく性能が最も高い
- **弱い点**
  - Windowsは公式にはサポートされていない。動かすにはWSL2、Docker Model Runner（Docker Desktop 4.54以降、WSL2バックエンド）、有志による非公式版のどれかになる
  - 起動時にVRAMの大半（既定で約90%）を確保し続けるため、ComfyUIと共存できない
  - 1人で使う用途では、得意な同時処理の性能を生かせない。10GBでは載せられるモデルも小さい

### TabbyAPI（ExLlamaV3）

- **良い点**
  - モデルがVRAMに全部収まるなら最速クラス。Windowsで動き、APIキーによる認証も標準で使える
- **弱い点**
  - EXL3形式のモデルしか使えず（GGUFは不可）、量子化済みのモデルの選択肢が少ない
  - モデルの一部をCPU側で動かす手段が見当たらず、10GBに全部収まるモデルに限られる
  - Python環境を自分で保守する必要がある

---

## 決定

**ローカルLLMの推論サーバーにOllamaを採用する。**

あわせて、VM側のアプリからはOllama独自の `/api/*` ではなく、**OpenAI互換の `/v1/*`（例：`/v1/chat/completions`）で呼ぶ**。接続先のURLとモデル名はアプリに直接書かず、環境変数で渡す。

---

## 決定の理由

### 1. 条件をすべて満たす候補の中で、手間が最も少ない

WSL2なしで動く、VRAMを自動で空ける、ログインなしで動かせる、OpenAI互換APIがある、という条件を満たすのは Ollama・llama.cpp・LM Studio の3つ。このうち、モデルの管理から設定、SSHからの操作まで最も手間が少ないのがOllamaである。「落ちても勝手に戻る」状態を1人で保つには、構成が単純であることの価値が大きい。

### 2. ComfyUIとのGPU共有に、既定の動きがそのまま合う

Ollamaは、使っていないモデルを既定で5分後にVRAMから降ろす。何も設定しなくても、LLMを使っていない間にComfyUIがVRAMを使える。

### 3. 他の候補の強みは、今の使い方では効いてこない

| 候補 | 強み | 今の使い方で効かない理由 |
| --- | --- | --- |
| llama.cpp | 認証 | 届く相手はTailscaleとWindowsファイアウォールで絞れる |
| llama.cpp | VRAMの細かい調整 | 10GBに収まるモデルを使っている間は必要ない |
| llama.cpp | 新しいモデルへの対応の速さ | 公開直後のモデルを追いかける使い方はしていない |
| LM Studio | 認証、GUIでのモデル選び | 機能はOllamaとほぼ同じ。操作はSSHからなのでGUIの出番が少ない。ソースが非公開で、ログイン前の起動方法も確認できない |
| vLLM / SGLang | 同時処理の性能 | 使うのは1人だけ。WSL2への依存とVRAMの確保し続けが条件に合わない |
| TabbyAPI | VRAMに収まるときの速さ | 使えるモデルの形式が限られ、10GBを超えるモデルに対応できない |

### 4. 後から替えやすい

VM側からOpenAI互換APIで呼んでおけば、llama.cppやLM Studioに乗り換えるときもURLとモデル名の変更で済む。この決定を後から覆す費用は小さい。また、ROADMAPのStep 6・7はすでにOllama前提で書かれており、今替える場合は手順の書き直しが必要になる。

---

## 結果

### 受け入れる弱点と対策

Ollamaを採用することで次の弱点を受け入れる。ROADMAPのStep 6で対策する。

| 弱点 | 対策 |
| --- | --- |
| 認証がない | Windowsファイアウォールで11434/TCPを受け付ける相手を、tailnet全体（`100.64.0.0/10`）から**VMとMacのTailscale IPだけ**に絞る（MacからOllamaを直接使う経路は設計書の§14にある） |
| コンテキスト長の既定が4kトークン | `OLLAMA_CONTEXT_LENGTH` を設定する（まず `32768` から始め、VRAMに収まっているかを `ollama ps` で確認する） |
| 長いコンテキストではVRAMが足りなくなりやすい | `OLLAMA_KV_CACHE_TYPE=q8_0` にして、KVキャッシュが使うメモリを半分にする |
| ComfyUIがVRAMを使ったままだと、入りきらない分がCPUで動いて遅くなる | `ollama ps` の表示が `100% GPU` になっているかを確認する。ComfyUIのVRAMは `POST /free` で解放できる |
| インストーラー版はログイン後に起動するアプリで、タスクスケジューラと二重に起動するとポートが衝突する | インストーラー版ではなくzip版を使い、タスクスケジューラから `ollama serve` を起動する |

### 未対応のこと

- ROADMAPのStep 6には、上の対策がまだ反映されていない

---

## 見直す条件

次のどれかが必要になったら、llama.cppの `llama-server`（ルーターモード）への乗り換えを検討する。

- APIキーで、呼べる相手を制御したい（例：VM上のエージェントのDockerコンテナからは呼ばせない）
- 10GBに収まらないモデルを、一部をCPUで動かして実用的な速さで使いたい
- 公開されたばかりのモデルを試したいが、Ollamaがまだ対応していない

---

## 参考

- Ollama
  - [Windows](https://docs.ollama.com/windows)
  - [FAQ](https://docs.ollama.com/faq)
  - [Context length](https://docs.ollama.com/context-length)
  - [Anthropic compatibility](https://docs.ollama.com/api/anthropic-compatibility)
- llama.cpp
  - [server README](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md)
  - [New in llama.cpp: Model Management](https://huggingface.co/blog/ggml-org/model-management-in-llamacpp)
  - [New in llama.cpp: Anthropic Messages API](https://huggingface.co/blog/ggml-org/anthropic-messages-api-in-llamacpp)
- LM Studio
  - [Run LM Studio as a service (headless)](https://lmstudio.ai/docs/developer/core/headless)
  - [Introducing LM Studio 0.4.0](https://lmstudio.ai/blog/0.4.0)
  - [lms server start](https://lmstudio.ai/docs/cli/serve/server-start)
  - [Anthropic Compatibility Endpoints](https://lmstudio.ai/docs/developer/anthropic-compat)
  - [LM Studio is free for use at work](https://lmstudio.ai/blog/free-for-work)
- vLLM
  - [vLLM on Windows in 2026](https://fazm.ai/t/vllm-windows-support-2026)
  - [Docker Model Runner Adds vLLM Support on Windows](https://www.docker.com/blog/docker-model-runner-vllm-windows/)
  - [Docker Model Runner](https://docs.docker.com/ai/model-runner/)
- TabbyAPI / ExLlamaV3
  - [TabbyAPI Getting Started](https://github.com/theroyallab/tabbyAPI/wiki/01.-Getting-Started)
  - [ExLlamaV3](https://github.com/turboderp-org/exllamav3)
- Foundry Local
  - [Run and install AI models on your Windows PC with Microsoft Foundry Local](https://4sysops.com/archives/run-and-install-ai-models-on-your-windows-pc-with-microsoft-foundry-local/)
