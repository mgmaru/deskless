# VMware Workstation Pro + Ubuntu Server 開発VM構成

## 1. このドキュメントの目的

自宅のWindowsデスクトップPCを開発環境の中心にし、外出先のMacBookからTailscale + SSHで接続して開発するために、Linux開発環境をVMとして構築します。

今回、VM構成として以下を採用します。

```text
Windows 11 Home
        ↓
VMware Workstation Pro 26H1
        ↓
Ubuntu Server 24.04 LTS
```

このUbuntu Server VMを、日常的なアプリ開発の**メイン開発サーバー**として利用します。

---

# 2. 採用する構成

## ホストPC

```text
OS   : Windows 11 Home
CPU  : Intel Core i7-13700KF
RAM  : 32GB
GPU  : NVIDIA GeForce RTX 3080 10GB
```

## VM管理ソフト

```text
VMware Workstation Pro 26H1
```

## ゲストOS

```text
Ubuntu Server 24.04 LTS
```

GUI付きのUbuntu Desktopではなく、Ubuntu Serverを使用します。

理由：

- GUIが不要
- SSH中心で利用する
- RAM消費を抑えられる
- CPU負荷を抑えられる
- サーバー用途として管理しやすい
- Dockerや開発ツールを動かす用途に向いている

---

# 3. 全体構成

```text
┌─────────────────────────────────────────────┐
│ Windows 11 Home                             │
│                                             │
│ ├─ Tailscale                               │
│ ├─ OpenSSH Server                          │
│ ├─ Ollama                                  │
│ ├─ ComfyUI                                 │
│ ├─ RTX 3080                                │
│ │                                           │
│ └─ VMware Workstation Pro 26H1             │
│      │                                      │
│      └─ Ubuntu Server 24.04 LTS             │
│          ├─ Tailscale                       │
│          ├─ OpenSSH                         │
│          ├─ Git                             │
│          ├─ Docker                          │
│          ├─ Python                          │
│          ├─ Node.js                         │
│          ├─ Go / Rust                       │
│          ├─ DB                              │
│          ├─ tmux                            │
│          ├─ AI Agent                        │
│          └─ 開発中のアプリ                  │
└─────────────────────────────────────────────┘
```

---

# 4. 役割分担

| コンポーネント | 役割 |
|---|---|
| Windows 11 Home | ホストOS |
| VMware Workstation Pro | VMの作成・起動・停止・管理 |
| Ubuntu Server | メイン開発環境 |
| Windows OpenSSH | MacBookからWindows管理 |
| Ubuntu OpenSSH | MacBookから開発VMへ接続 |
| Tailscale | 外出先と自宅PCを安全に接続 |
| Ollama | Windows側GPUを利用したLLM推論 |
| ComfyUI | Windows側GPUを利用した画像生成 |
| RTX 3080 | AI推論・画像生成用GPU |
| tmux | SSH切断後も処理を継続 |
| Docker | アプリやミドルウェアの実行 |

---

# 5. Ubuntu VMの初期リソース

現在のホストPCは、

```text
CPU : Intel Core i7-13700KF
RAM : 32GB
```

なので、Ubuntu VMは控えめな設定から開始します。

## 初期設定

| 項目 | 初期値 |
|---|---:|
| vCPU | 4 |
| RAM | 8GB |
| 仮想ディスク | 100GB |
| OS | Ubuntu Server 24.04 LTS |
| GUI | なし |
| ネットワーク | Bridgedを第一候補 |

```text
Ubuntu Server VM
├─ 4 vCPU
├─ 8GB RAM
└─ 100GB Virtual Disk
```

---

# 6. 最初から大きく割り当てない理由

VMのリソースは後から変更できます。

そのため、

```text
最初
4 vCPU / 8GB
      ↓
不足したら
6 vCPU / 12GB
      ↓
さらに必要なら
8 vCPU / 16GB
```

のように段階的に増やします。

Windows側では、

- Ollama
- ComfyUI
- ブラウザ
- その他Windowsアプリ

も動かすため、最初からRAM 16GBなどをVMへ固定しない方が安全です。

---

# 7. RAMは後から変更可能

VMwareではVMを停止した状態で、Ubuntu VMへ割り当てるRAMを変更できます。

例：

```text
8GB
 ↓
12GB
 ↓
16GB
```

必要に応じて増減できます。

---

# 8. CPUも後から変更可能

vCPUも後から変更できます。

例：

```text
4 vCPU
   ↓
6 vCPU
   ↓
8 vCPU
```

vCPUを4つ割り当てても、Ubuntuがアイドル状態であれば4 CPU分を常時100%使用するわけではありません。

CPU負荷は、

- Docker build
- npm build
- pytest
- コンパイル
- AI Agentの並列処理

などの処理を実行したときに増えます。

---

# 9. 仮想ディスクも拡張可能

仮想ディスクも後から、

```text
100GB
  ↓
150GB
  ↓
200GB
```

のように拡張できます。

ただし、RAMやCPUとは異なり、

1. VMware側で仮想ディスクを拡張
2. Ubuntu側でパーティション/ファイルシステムを拡張

という作業が必要になる場合があります。

そのため、ディスク容量を**増やすことは比較的容易**ですが、縮小は慎重に扱います。

---

# 10. 仮想ディスクは可変サイズを利用

100GBの仮想ディスクを設定しても、最初から物理SSDを100GB消費する方式にはしません。

```text
Virtual Disk 最大容量 : 100GB
実際の使用量          : 20GB
```

であれば、物理SSD側の消費量も使用分を中心に増加していく構成にします。

---

# 11. Ubuntu Serverを選ぶ理由

Ubuntu ServerはUbuntu Desktopより軽量です。

```text
Ubuntu Desktop
├─ GNOME
├─ Display Server
├─ GUIアプリ
└─ 各種デスクトップサービス

Ubuntu Server
├─ SSH
├─ systemd
├─ Docker
├─ Git
└─ 開発ツール
```

今回MacBookからVS Code Remote SSHやTerminalを利用するため、Ubuntu側のGUIは必要ありません。

---

# 12. WSL2との役割分担

既存PCにはWSL2上のUbuntuがあります。

```text
Windows 11 Home
└─ WSL2
   └─ Ubuntu
```

ただし、新構成ではメイン開発環境を、

```text
VMware
└─ Ubuntu Server
```

へ移します。

そのため、将来的にはWSL2の用途を減らし、

```text
Linux開発
    ↓
Ubuntu Server VM

GPU処理
    ↓
Windows
```

と役割を明確に分離します。

WSL2は、Linux上でCUDA/PyTorchを直接扱う必要が出た場合などに再利用を検討します。

---

# 13. ネットワーク構成

VMwareのネットワークは、まず**Bridged Networking**を第一候補とします。

```text
                  自宅ルーター
                       │
                    LAN
                       │
          ┌────────────┴────────────┐
          │                         │
    Windows Host               Ubuntu VM
    192.168.x.x                192.168.x.y
          │                         │
      Tailscale                  Tailscale
```

Ubuntu VMがLAN上で独立した1台のPCのように動作します。

---

# 14. Tailscale構成

Tailscaleは次の3か所に配置します。

```text
MacBook
Windows Host
Ubuntu VM
```

外出先からは、

```text
MacBook
   │
   │ Tailscale
   │
   ├──── SSH ────→ Ubuntu VM
   │               メイン開発
   │
   ├──── SSH ────→ Windows
   │               Windows管理
   │
   └──── HTTP ───→ Windows
                   Ollama / ComfyUI
```

という経路を使います。

---

# 15. MacBookからUbuntu VMへ接続

メインの開発経路です。

```text
MacBook
   │
   │ Tailscale
   │
   │ SSH
   ▼
Ubuntu Server VM
```

VS Codeでは、

```text
VS Code
   ↓
Remote - SSH
   ↓
Ubuntu Server
   ↓
~/projects
```

として利用します。

---

# 16. MacBookからWindowsへ接続

WindowsにはOpenSSH Serverを有効にします。

用途：

- PowerShell操作
- Ollamaの起動・停止
- ComfyUIの起動・停止
- ログ確認
- Windows側のトラブルシューティング

```text
MacBook
   │
   │ Tailscale
   │
   │ SSH
   ▼
Windows
```

---

# 17. GPU処理はWindows側に集約

Ubuntu VMからRTX 3080を直接利用する構成にはしません。

GPU処理はWindows側へ集約します。

```text
Windows
├─ Ollama
│    └─ RTX 3080
│
└─ ComfyUI
     └─ RTX 3080
```

Ubuntu VMからGPU処理が必要な場合はHTTP APIを利用します。

---

# 18. Ubuntu VMからOllamaを利用

```text
Ubuntu VM
   │
   │ HTTP API
   ▼
Windows
   │
   ▼
Ollama
   │
   ▼
RTX 3080
```

Ubuntu VM自身にGPUが見えている必要はありません。

---

# 19. Ubuntu VMからComfyUIを利用

```text
Ubuntu VM
   │
   │ HTTP / WebSocket
   ▼
Windows
   │
   ▼
ComfyUI
   │
   ▼
RTX 3080
```

画像生成処理もWindows側へ委譲します。

---

# 20. SSH切断対策

Ubuntu VMでは `tmux` を利用します。

```bash
tmux new -s dev
```

例：

```text
tmux
├─ AI Agent
├─ Web Server
├─ Tests
└─ Shell
```

MacBookのWi-Fi切断やスリープでSSHが切れても、Ubuntu VM上の処理は継続します。

再接続後：

```bash
tmux attach -t dev
```

---

# 21. VMの常時運用

今回のUbuntu VMは、

> 一時的な検証環境

ではなく、

> 自宅のメイン開発サーバー

として使用します。

そのため、

```text
Windows起動
   ↓
VMware
   ↓
Ubuntu Server VM
   ↓
systemd
   ├─ sshd
   ├─ tailscaled
   ├─ Docker
   └─ その他サービス
```

という運用を目指します。

VMwareの自動起動機能も利用する予定です。

---

# 22. 開発データの置き場所

開発中のGitリポジトリは、原則Ubuntu VM内へ集約します。

```text
/home/<user>/projects/
├─ project-a
├─ project-b
├─ project-c
└─ ...
```

MacBook側には別のWorking Treeを持たず、Remote SSH経由で直接Ubuntu VM上のファイルを編集します。

これにより、

```text
MacBookだけに未commit差分がある
Windowsだけに未commit差分がある
```

という問題をなくします。

---

# 23. バックアップは別途必要

Ubuntu VMが唯一のWorking Treeになるため、VM障害時の未コミットデータ消失を防ぐ必要があります。

候補：

- VMのバックアップ
- 仮想ディスクのバックアップ
- restic
- BorgBackup
- 別ストレージへの定期バックアップ

```text
Ubuntu VM
├─ GitHub
│    └─ commit済みデータ
│
└─ Backup
     └─ 未commitデータも保護
```

---

# 24. 今回決定した初期構成

```text
Windows 11 Home
│
├─ Windows側
│   ├─ Tailscale
│   ├─ OpenSSH Server
│   ├─ Ollama
│   ├─ ComfyUI
│   └─ RTX 3080
│
└─ VMware Workstation Pro 26H1
    │
    └─ Ubuntu Server 24.04 LTS
         │
         ├─ 4 vCPU
         ├─ 8GB RAM
         ├─ 100GB可変仮想ディスク
         ├─ Bridged Networking
         │
         ├─ Tailscale
         ├─ OpenSSH
         ├─ Git
         ├─ Docker
         ├─ Python
         ├─ Node.js
         ├─ Go / Rust
         ├─ DB
         ├─ tmux
         └─ AI Agent
```

---

# 25. 将来の拡張方針

負荷を見ながらVMリソースを調整します。

```text
初期
4 vCPU / 8GB RAM
        ↓
必要に応じて
6 vCPU / 12GB RAM
        ↓
さらに必要なら
8 vCPU / 16GB RAM
```

ただし、Windows側ではOllamaやComfyUIも使用するため、ホスト側RAMを十分残すことを優先します。

---

# 26. 最終方針

今回のVM構成は、

```text
VMware Workstation Pro 26H1
            +
Ubuntu Server 24.04 LTS
```

とします。

目的は、

> WindowsデスクトップPCの中に、安定して常時稼働できる独立Linux開発サーバーを構築すること

です。

MacBookは、

```text
開発端末
   ↓
Tailscale
   ↓
SSH
   ↓
Ubuntu Server VM
```

として利用します。

GPU処理は、

```text
Ubuntu VM
   ↓
API
   ↓
Windows
   ↓
Ollama / ComfyUI
   ↓
RTX 3080
```

と分離します。

この構成により、

- 開発環境の一元化
- 未コミット差分の分散防止
- Linuxサーバーとしての独立性
- Windows側GPU資源の活用
- MacBookからの安定したリモート開発

を両立します。
