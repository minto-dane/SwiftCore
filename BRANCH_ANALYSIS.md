# ブランチ比較分析

## 対象ブランチ

| ブランチ | 役割 |
|---|---|
| `claude/security-architecture-review-30RVA` | セキュリティ・アーキテクチャ **レビュー**（文書作成のみ） |
| `claude/security-vulnerability-audit-4Hrsd` | セキュリティ脆弱性 **修正**（レビュー + コード変更3ラウンド） |

---

## 概要

`security-architecture-review-30RVA` は `security-vulnerability-audit-4Hrsd` の**祖先コミット**である。
`security-vulnerability-audit-4Hrsd` はそのレビュー結果を受けて実際のコード修正を 3 ラウンドにわたって積み上げたものである。

```
main (f14f533)
 |
 +-- security-architecture-review-30RVA (20c3075)  ← SECURITY_REVIEW.md を追加
      |
      +-- 6e3b067  fix: セキュリティ脆弱性の全面修正 (Round 1)
           |
           +-- 353b24b  fix: 第2回セキュリティ脆弱性修正 (Round 2)
                |
                +-- 514f748  fix: Round 3 security hardening
                     |
                     security-vulnerability-audit-4Hrsd
```

---

## `claude/security-architecture-review-30RVA` の内容

**コミット数**: 1（ベースブランチからの追加分）

### 追加ファイル
- `SECURITY_REVIEW.md` — カーネル全体の静的解析レポート（744行）

### 文書に記載された脆弱性の概要

| 重大度 | 件数 | 主な内容 |
|---|---|---|
| 🔴 Critical | 5 | ユーザーポインタ無検証、I/Oポート無制限アクセス、fork()メモリ非分離、ELFオーバーフロー、ELFカーネル空間マップ |
| 🟠 High | 6 | fs::read()のUse-after-free競合、カーネル.textページがWRITABLEなど |
| 🟡 Medium | 8 | スタックNX欠如、メモリ順序不正、ASLR未実装など |
| 🔵 Low/Info | 8 | PIC EOI不正送信、物理メモリリーク、デバッグコード残存など |

**コード変更はなし** — 文書作成・分析のみ。

---

## `claude/security-vulnerability-audit-4Hrsd` の追加内容

`security-architecture-review-30RVA` に対して**3ラウンドの修正コミット**を追加。

### Round 1: セキュリティ脆弱性の全面修正 (commit `6e3b067`)

**変更ファイル**: 12ファイル、+232行 / -171行

| 対応ID | 内容 | 変更ファイル |
|---|---|---|
| CRIT-01 | `validate_user_ptr()` を追加し、全syscallでユーザーポインタ検証 | `syscall/mod.rs`, `io.rs`, `ipc.rs`, `process.rs`, `fs.rs`, `exec.rs` |
| CRIT-02 | `caller_has_port_privilege()` でI/OポートをService/Core権限のみに制限 | `syscall/io_port.rs` |
| CRIT-04 | ELFセグメント読み込みで `checked_add` によりオーバーフロー防止 | `syscall/exec.rs` |
| CRIT-05 | ELFの `p_vaddr + p_memsz` がユーザー空間上限を超えないことを検証 | `syscall/exec.rs` |
| HIGH-02 | `fs::read()` でロック解放前に `FileHandle` にアクセスし Use-after-free 防止 | `syscall/fs.rs` |
| HIGH-03 | `brk()` / `mmap()` にユーザー空間アドレス範囲チェックを追加 | `syscall/process.rs` |
| MED-03 | `alloc_user_stack()` に `NO_EXECUTE` フラグを追加 | `mem/user.rs` |
| MED-05 | `Ordering::Relaxed` → `Ordering::SeqCst` でメモリ順序保証 | `syscall/syscall_entry.rs` |
| MED-07 | ELFヘッダの `e_machine == EM_X86_64` 検証を追加 | `elf/loader.rs` |
| MED-08 | ELFプログラムヘッダ走査で `phentsz == 0` チェックとオーバーフロー防止 | `syscall/exec.rs` |
| LOW-01 | `generic_interrupt_handler()` のスレーブPICへの不正EOI送信を修正 | `interrupt/idt.rs` |
| その他 | `task/context.rs` の古い実装を削除（整理） | `task/context.rs` |

---

### Round 2: 第2回セキュリティ脆弱性修正 (commit `353b24b`)

**変更ファイル**: 9ファイル、+94行 / -19行

| 対応ID | 内容 | 変更ファイル |
|---|---|---|
| C-2 | IRQ1(キーボード)ハンドラをIDTベクタ33に登録（未登録で #GP → システム停止していた） | `interrupt/idt.rs` |
| C-3 | CVE-2012-0217対策: SYSRETQ前にユーザーRSPの正規アドレスチェックを追加 | `syscall/syscall_entry.rs` |
| C-7 | EXT2パス解析に `..`/`.` チェックを追加してディレクトリトラバーサルを防止 | `init/fs.rs` |
| H-1 | 重複STAR/LSTAR/FMASK MSR初期化関数を除去し一本化 | `interrupt/mod.rs` |
| H-15 | ATAの `identify` コマンド後DRQ待ちループにタイムアウトを追加（無限ループ防止） | `services/disk/ata.rs` |
| H-16 | ATAのスレーブドライブ選択ビットが常にマスタになっていたバグを修正 | `services/disk/ata.rs` |
| H-17 | `sleep_until()` で `yield_now()` 後に再スケジュールされないバグをビジーウェイトに変更して修正 | `syscall/time.rs` |
| MED-27 | エントリポイントが0のELFを `EINVAL` で拒否 | `syscall/exec.rs` |
| MED-32 | `sti`（割り込み有効化）をPIT/スケジューラ/タイマー初期化の**後**に移動して競合状態を解消 | `init/mod.rs` |
| その他 | `clock_gettime()` に `validate_user_ptr(16バイト)` を追加 | `syscall/time.rs` |
| L-1 | `enable_sse()` でCR4ビット20(SMEP)とビット21(SMAP)をセット | `cpu.rs` |

---

### Round 3: セキュリティ強化 (commit `514f748`)

**変更ファイル**: 12ファイル、+133行 / -4行

| 内容 | 変更ファイル |
|---|---|
| `EFER.NXE` ビットをSSE/FPU初期化**前**に有効化し `NO_EXECUTE` PTEを有効にする | `cpu.rs` |
| カーネルヒープページに `NO_EXECUTE` フラグを追加（W^Xの強制） | `mem/allocator.rs` |
| `inode()` の `inodes_per_group == 0`、`data_block_number()` の `block_size == 0` によるゼロ除算を防止 | `init/fs.rs` |
| IPC `send()` で存在しないスレッドへの送信を拒否（ゴーストメッセージ注入 / IPC DoS防止） | `syscall/ipc.rs` |
| `GetThreadPrivilege` syscall (番号527) を新規追加（Core=0/Service=1/User=2を返す） | `syscall/task.rs`, `syscall/types.rs`, `syscall/mod.rs` |
| ユーザー空間swiftlibに `GetThreadPrivilege` を追加 | `user/sys.rs`, `user/task.rs` |
| diskサービスがIPC要求元のプロセス権限をチェックし、Userプロセスからの要求を拒否 | `services/disk/main.rs` |
| `thread_id_exists(u64) -> bool` を公開（IPC検証で利用） | `task/thread.rs`, `task/mod.rs` |

---

## 変更規模の比較

| ブランチ | ベースからの追加コミット | 変更ファイル数 | 追加行数 | 削除行数 |
|---|---|---|---|---|
| `security-architecture-review-30RVA` | 1 | 1 (新規) | 744 | 0 |
| `security-vulnerability-audit-4Hrsd` | 4 (上記+3) | 26 | +459 | -194 |

---

## まとめ

- **`security-architecture-review-30RVA`**: 脆弱性の**発見・文書化**フェーズ。`SECURITY_REVIEW.md` を通じてカーネル全体のリスクを可視化した。コード変更はない。
- **`security-vulnerability-audit-4Hrsd`**: `security-architecture-review-30RVA` を包含し、さらに**修正実装**フェーズを3ラウンド追加したもの。ユーザーポインタ検証・権限チェック・整数オーバーフロー防止・NXE/SMEP/SMAP有効化・IPC DoS対策など、文書で挙げられた主要脆弱性の大半をコードレベルで解消している。
