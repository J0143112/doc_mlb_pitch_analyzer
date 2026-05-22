# SUP8準拠ソフトウェア開発 — 学習サマリ
## MLB 配球分析ツールを題材とした構成管理プロセス実践

---

> **このドキュメントの目的**
> Honda 社内 SW構成管理ガイドライン（SUP8）を個人学習プロジェクトで実践し、
> 業務適用に向けて「何をやったか・なぜやったか」を発表できる形にまとめたもの。

---

## 1. なぜこのプロジェクトを作ったか

| 課題 | 目標 |
|------|------|
| SUP8 のルールが文書上の話になりがち | 実際に手を動かして体で理解する |
| CI/CD・テスト管理を業務で活かしたい | GitHub Actions でゼロから構築して仕組みを体得 |
| 学習モチベーションを持続させたい | 好きな野球（MLB）を題材にする |

**選んだ題材**：MLB 投手の配球データ（Statcast）を分析するツール
- 球種構成・カウント別傾向・奪三振の決め球などを計算
- 山本由伸・今永昇太・千賀滉大・大谷翔平のデータを対象

---

## 2. SUP8とは何か（3行まとめ）

> **SUP8 = SW 構成管理ガイドライン（DPG環境版）**
>
> ソフトウェアの「何を」「どのバージョンで」「なぜ変えたか」を
> 追跡・管理するためのルールブック。
> ASPICE（自動車業界の開発プロセス標準）SUP.8 を Honda 社内に適用したもの。

---

## 3. V字プロセスと SUP8 の関係

```
 ← 要件定義 ─────────────────────── 検証・確認 →

 SYS.1 システム要件                  SYS.6 受入テスト
   └─ SWE.1 SW 要件仕様 ────────── SWE.6 SW 適格性確認
        └─ SWE.2 アーキ設計 ───── SWE.5 統合テスト
             └─ SWE.3 実装 ──── SWE.4 単体テスト
                         ↑
                      【ここが今回の実践範囲】

 SUP8 は V 字の「全工程を通じた管理ルール」として横断的に機能する
 ┌──────────────────────────────────────────────────────┐
 │  リポジトリ管理 │ ブランチ戦略 │ 変更追跡 │ ツール管理 │
 └──────────────────────────────────────────────────────┘
```

---

## 4. 実施内容と SUP8 への対応一覧

### 4-1. リポジトリ設計（SUP8 §3・§4）

**やったこと**：2つのリポジトリを SUP8 命名規則で作成

```
src_mlb_pitch_analyzer/   ← ソースコード（src_ プレフィックス）
doc_mlb_pitch_analyzer/   ← ドキュメント（doc_ プレフィックス）
```

**なぜ SUP8 に準拠しているか**：

| SUP8 ルール | 今回の対応 | 理由 |
|------------|-----------|------|
| `src_` = ソースコードリポジトリ | `src_mlb_pitch_analyzer` | コードと文書を分離して独立したバージョン管理が可能 |
| `doc_` = ドキュメントリポジトリ | `doc_mlb_pitch_analyzer` | 仕様書の改版がコードリリースと独立して行える |
| GitLFS = バイナリ・大容量ファイル | `*.csv / *.xlsx / *.pdf / *.png` | Gitの差分管理はテキスト向け。バイナリはLFSで効率化 |

**フォルダ構成**：

```
src_mlb_pitch_analyzer/
├── src/              ← プロダクションコード（実装物）
│   ├── analyzer/     ← ビジネスロジック
│   └── fetcher/      ← データ取得
├── test/
│   ├── unittest/     ← SWE.4 単体テスト
│   └── fixtures/     ← テストデータ（GitLFS管理）
├── .github/
│   └── workflows/    ← CI/CD定義
├── pyproject.toml    ← ツール設定（pytest/ruff/coverage）
└── .gitattributes    ← LFSトラッキング設定
```

---

### 4-2. ドキュメント管理（SUP8 SWE.1・SWE.2）

**やったこと**：V字プロセスに沿った仕様書・設計書を作成

#### SWE1-1 ソフトウェア要件仕様書

```
doc_mlb_pitch_analyzer/requirement/SWE1-1_ソフトウェア要件仕様書.md
```

| 含まれる内容 | SUP8 での意義 |
|------------|--------------|
| REQ-001〜005（機能要件） | 「何を作るか」の基準線。テストの合否判断の根拠になる |
| NFR-001〜004（非機能要件） | カバレッジ80%・ZeroDivision禁止など品質ゲートを明文化 |
| トレーサビリティマトリクス | 要件→設計→テストの縦の繋がりを追跡可能にする |
| 境界値テーブル | 投球回=0など異常系を要件段階で定義しておく |

**業務への適用ポイント**：  
要件が曖昧だと「何をテストすれば合格か」がわからない。  
仕様書に NFR（非機能要件）として数値を書くことで、CI の閾値設定の根拠になる。

#### SWE2-1 ソフトウェアアーキテクチャ設計書

```
doc_mlb_pitch_analyzer/design/SWE2-1_ソフトウェアアーキテクチャ設計書.md
```

| 含まれる内容 | SUP8 での意義 |
|------------|--------------|
| モジュール構成図 | 実装前に依存関係を整理。変更の影響範囲を把握しやすくする |
| クラス・メソッド定義 | 設計とコードの対応が明確になり、レビューで確認しやすい |
| 例外クラス設計 | エラー処理の方針を設計段階で統一する |
| トレーサビリティ | 要件↔設計の紐付けで「この機能どの要件から来た？」が即答可能 |

---

### 4-3. コード実装（SUP8 SWE.3）

**やったこと**：設計書に基づきPythonモジュールを実装

```python
# 設計書の通り、例外クラスで「何が起きたか」を明示
class PitcherNotFoundError(Exception):
    """指定した投手が見つからない場合に送出する例外。"""

# NFR-003（ZeroDivision禁止）を全メトリクス関数で実装
def calc_era(earned_runs: int, innings_pitched: float) -> float | None:
    if innings_pitched <= 0:
        return None          # ← None返却でアプリをクラッシュさせない
    return round((earned_runs * 9) / innings_pitched, 2)
```

**なぜ SUP8 に準拠しているか**：

| SUP8 の観点 | 今回の対応 |
|------------|-----------|
| 実装と設計の一致 | クラス名・メソッド名を SWE2-1 の通りに実装 |
| 変更追跡 | すべての変更が Git コミット履歴に残る |
| エラー処理の標準化 | `PitcherNotFoundError` を設計段階から定義し実装に反映 |

---

### 4-4. 単体テスト（SUP8 SWE.4）

**やったこと**：pytest で 69 件のテストを作成

```
test/unittest/
├── test_metrics.py    ─ ERA/WHIP/FIP/K9/BB9/KBB  (17テスト)
├── test_analyzer.py   ─ PitchMixAnalyzer, SequenceAnalyzer (35テスト)
└── test_fetcher.py    ─ StatcastClient (17テスト)
```

**テスト結果**：

```
69 passed  ─  カバレッジ 93.5%（目標 80% 超達成）
```

| モジュール | カバレッジ | 判定 |
|-----------|-----------|------|
| metrics.py | 100% | ✅ |
| pitch_mix.py | 100% | ✅ |
| sequence.py | 100% | ✅ |
| statcast_client.py | 81% | ✅ |
| **合計** | **93.5%** | ✅ |

**なぜ SUP8 に準拠しているか**：

| SUP8 / NFR | 対応したテスト設計 |
|-----------|-----------------|
| NFR-002（カバレッジ ≥ 80%） | pyproject.toml に `fail_under=80` を設定。CI が自動で判定 |
| NFR-003（ZeroDivisionError 禁止） | `test_zero_innings` で投球回=0 → None 返却を全関数でテスト |
| NFR-004（エラー処理） | `PitcherNotFoundError` が正しく送出されるかテスト |
| SWE.4 要求：モックによる独立性 | `unittest.mock` で pybaseball の外部 API 呼び出しを完全モック化 |

**テスト設計のポイント（業務適用に向けて）**：

```python
# 外部依存（API・DB）はモックで切り離す → テストが「速く・安定して・繰り返し実行できる」
with patch.object(client, "_fetch_from_api", return_value=sample_df) as mock_api:
    result = client.get_pitcher_data("Yoshinobu Yamamoto", 2024)
    mock_api.assert_called_once_with(673548, 2024)  # ← 「正しいIDで呼ばれたか」まで検証
```

---

### 4-5. CI/CDパイプライン（SUP8 SWE.5・§7）

**やったこと**：GitHub Actions でPR作成時に自動検証

```yaml
# .github/workflows/ci.yml の処理フロー
Pull Request 作成
    ↓
① ruff lint（コーディング規約チェック）
    ↓ エラーがあれば NG → マージ不可
② pytest（69件のテスト実行）
    ↓ 1件でも失敗すれば NG → マージ不可
③ coverage ≥ 80% チェック
    ↓ 80% 未満なら NG → マージ不可
④ coverage.xml をアーティファクトに保存
    ↓
✅ 全通過 → レビュー可能状態
```

**なぜ SUP8 に準拠しているか**：

| SUP8 §7（ブランチ戦略・品質ゲート） | 今回の対応 |
|----------------------------------|-----------|
| `main` ブランチは保護する | BPR（Branch Protection Rules）でPR必須に設定 |
| 変更は必ずレビューを経て main に入る | `hosa-fea/*` ブランチ → PR → CI → マージの流れ |
| 品質チェックをゲートとして設ける | CI 失敗 = マージ不可（人の判断ではなく自動で強制）|

---

### 4-6. ブランチ戦略（SUP8 §7）

```
main              ← 本番相当。直接 push 禁止（BPR設定）
  └── hosa-fea/swe4-unit-tests-and-ci  ← 今回作業したブランチ
       ↑
       ブランチ名の規則: [4文字]-fea/[作業内容]
       hosa = Hosoyama の識別子
       fea  = feature（新機能追加）
```

**ブランチ種別**：

| ブランチ種別 | 命名例 | 用途 |
|------------|--------|------|
| `[id]-main` | `hosa-main` | チームの統合ブランチ |
| `[id]-fea/xxx` | `hosa-fea/add-era-calc` | 機能追加 |
| `[id]-fix/xxx` | `hosa-fix/zero-div-bug` | バグ修正 |
| `[id]-rel/xxx` | `hosa-rel/v1.0.0` | リリース準備 |

---

## 5. SUP8 準拠チェックリスト

| # | SUP8 要件 | 対応 | エビデンス |
|---|-----------|------|-----------|
| 1 | リポジトリ命名規則（src_/doc_） | ✅ | `src_mlb_pitch_analyzer`, `doc_mlb_pitch_analyzer` |
| 2 | フォルダ構成（src/test/.github）| ✅ | ディレクトリ構造 |
| 3 | ツール使い分け（GitHub/GitLFS） | ✅ | `.gitattributes`（*.csv→LFS） |
| 4 | ドキュメント（SWE.1 要件仕様書） | ✅ | `SWE1-1_ソフトウェア要件仕様書.md` |
| 5 | ドキュメント（SWE.2 設計書） | ✅ | `SWE2-1_ソフトウェアアーキテクチャ設計書.md` |
| 6 | 実装と設計の一致（SWE.3） | ✅ | クラス名・メソッド名が設計書と対応 |
| 7 | 単体テスト（SWE.4）カバレッジ≥80% | ✅ | 69 tests / 93.5% coverage |
| 8 | ブランチ命名規則（§7） | ✅ | `hosa-fea/swe4-unit-tests-and-ci` |
| 9 | PR必須ワークフロー（§7） | ✅ | GitHub Actions CI + BPR |
| 10 | 変更追跡（コミット履歴） | ✅ | `git log` で全変更が追跡可能 |

---

## 6. アーキテクチャ全体図

```
【データフロー】

  pybaseball API
       ↓
  StatcastClient          ← REQ-001: データ取得・キャッシュ管理
  (.cache/statcast/*.csv)
       ↓
  ┌────────────┬──────────────────┬────────────────┐
  │PitchMixAna │SequenceAnalyzer  │  metrics.py    │
  │lyzer       │                  │                │
  │overall_mix │transition_matrix │ calc_era()     │
  │by_count()  │next_pitch_top3() │ calc_whip()    │
  │by_batter   │                  │ calc_fip()     │
  │strikeout_  │                  │ calc_k9/bb9    │
  │pitch()     │                  │ calc_kbb()     │
  └────────────┴──────────────────┴────────────────┘
       ↓               ↓                ↓
  REQ-002            REQ-003          REQ-004
  球種分析          配球シーケンス     メトリクス計算
```

---

## 7. 技術スタックと選定理由

| 技術 | 役割 | 選定理由 |
|------|------|---------|
| Python 3.10+ | 実装言語 | 型ヒント、pandas との親和性 |
| pandas | データ処理 | DataFrameで表形式データを効率的に扱う |
| pybaseball | MLB データ取得 | Statcast データへの公式アクセス手段 |
| pytest | テストフレームワーク | フィクスチャ・パラメトライズが強力 |
| pytest-cov | カバレッジ計測 | CI での自動閾値チェックが可能 |
| ruff | Lint / Formatter | ruff は flake8+isort+pyflakes の高速統合版 |
| GitHub Actions | CI/CD | push/PR に連動して自動実行。サーバー不要 |
| Git LFS | 大容量ファイル管理 | CSVなどバイナリをリポジトリを肥大化させずに管理 |

---

## 8. 学んだこと・業務適用のポイント

### 8-1. 「SUP8は面倒なルール」から「品質を守る仕組み」へ

> SUP8 を文書だけで理解しようとすると「なぜこのルールが必要か」がわかりにくい。
> 実際にCI が「テストカバレッジ80%未満でマージ不可」を自動で跳ね返す体験をすると、
> **「品質ゲートを人の判断ではなく機械に強制させる」ことの意味** が腑に落ちる。

### 8-2. トレーサビリティの重要性

```
要件: NFR-003「投球回=0でZeroDivisionError禁止」
  ↓
設計: 全メトリクス関数に innings_pitched ≤ 0 チェックを明記
  ↓
実装: calc_era / calc_whip / ... の先頭でガード節
  ↓
テスト: TestCalcEra::test_zero_innings など全関数でテスト
  ↓
CI: pytest が自動で検証し、カバレッジレポートに記録
```

この縦の繋がり（トレーサビリティ）があることで：
- バグが出たとき「どの要件から来た設計か」が即わかる
- 変更時に「どのテストが影響を受けるか」が予測できる

### 8-3. CGW/AUTOSAR 業務への適用イメージ

| 今回学んだ手法 | 業務での適用先 |
|--------------|--------------|
| `src_` / `doc_` 分離 | src_cgw_v1（CGW実装）/ doc_cgw_v1（設計書）の分離管理 |
| GitLFS でバイナリ管理 | ARXML ファイル・Excel 仕様書・バイナリイメージ |
| ブランチ保護 + CI | INTEGPROC の成果物を自動品質チェック後にマージ |
| トレーサビリティ | AUTOSAR 要件 → アーキ設計 → 実装コードの追跡 |
| モックテスト | 診断機 HW なしで診断スタックのロジックを単体テスト |

---

## 9. 今後の予定

| ステップ | 内容 | SUP8 対応 |
|---------|------|-----------|
| Step 7 | BPR設定（GitHub UI）| §7 ブランチ保護 |
| Step 8 | ゾーン分析モジュール実装（REQ-004） | SWE.3 |
| Step 9 | 可視化モジュール実装（matplotlib） | SWE.3 |
| Step 10 | 統合テスト作成 | SWE.5 |
| Step 11 | リリースタグ付け（v1.0.0） | SUP8 §5 バージョン管理 |

---

## 10. リポジトリ情報

| 項目 | 値 |
|------|-----|
| ソースリポジトリ | https://github.com/J0143112/src_mlb_pitch_analyzer |
| ドキュメントリポジトリ | doc_mlb_pitch_analyzer（ローカル） |
| 現在のブランチ | `hosa-fea/swe4-unit-tests-and-ci` |
| 最新コミット | `feat(swe4+swe5): ユニットテスト・CI/CDパイプライン追加` |
| テスト件数 | 69 tests |
| カバレッジ | 93.5% |

---

*作成者: 細山 裕樹（yuki_hosoyama@jp.honda）*
*作成日: 2026-05-22*
*プロセス: SUP8 SW構成管理ガイドライン（DPG環境版）準拠*
