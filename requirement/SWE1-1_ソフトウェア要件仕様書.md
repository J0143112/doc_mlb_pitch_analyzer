# SWE1-1 ソフトウェア要件仕様書
# MLB 投手配球分析ツール

| 項目 | 内容 |
|---|---|
| 文書ID | SWE1-1 |
| バージョン | 1.1 |
| 作成日 | 2026-05-22 |
| 更新日 | 2026-05-25 |
| 変更内容 | REQ-006 球速分析機能を追加 |
| ステータス | Draft |

---

## 1. 目的・スコープ

MLB 投手の Statcast データを取得・分析し、配球傾向をレポートするツールを開発する。

**対象**: 個人学習用（SUP8プロセス体験 + CI/CD実践）

---

## 2. 用語定義

| 用語 | 定義 |
|---|---|
| Statcast | MLB公式の投球追跡データ（球種/球速/コース/結果等） |
| pitch_type | 球種コード（FF=フォーシーム, SL=スライダー, CH=チェンジアップ等） |
| ERA | 防御率 = (自責点 × 9) / 投球回 |
| WHIP | (四球 + 被安打) / 投球回 |
| FIP | 守備無関係防御率 = (13×HR + 3×(BB+HBP) - 2×K) / IP + C |
| K/9 | 9回あたり奪三振 = (K × 9) / IP |
| BB/9 | 9回あたり四球 = (BB × 9) / IP |
| カウント | ボール数-ストライク数（例: 2-2） |
| 配球シーケンス | 投球の連続パターン（前球→今球） |

---

## 3. 機能要件

### REQ-001: データ取得機能
| ID | 要件 | 優先度 |
|---|---|---|
| REQ-001-01 | 投手名とシーズンを指定してStatcastデータを取得できること | Must |
| REQ-001-02 | 取得データをローカルにCSV形式でキャッシュできること | Should |
| REQ-001-03 | キャッシュが存在する場合はAPIを呼ばずキャッシュを使用すること | Should |

### REQ-002: 球種分析機能
| ID | 要件 | 優先度 |
|---|---|---|
| REQ-002-01 | 球種別使用率（割合）を集計できること | Must |
| REQ-002-02 | カウント（B-S）別の球種使用率を集計できること | Must |
| REQ-002-03 | 左打者・右打者別の球種使用率を集計できること | Should |
| REQ-002-04 | 奪三振の決め球（最終球）を集計できること | Must |

### REQ-003: 配球シーケンス分析機能
| ID | 要件 | 優先度 |
|---|---|---|
| REQ-003-01 | 前球の球種→今球の球種の遷移確率行列を計算できること | Must |
| REQ-003-02 | 特定球種の後に何を投げるか上位3球種を返せること | Should |

### REQ-004: ゾーン分析機能
| ID | 要件 | 優先度 |
|---|---|---|
| REQ-004-01 | 球種別のストライクゾーン投球分布をヒートマップで出力できること | Should |

### REQ-005: レポート出力機能
| ID | 要件 | 優先度 |
|---|---|---|
| REQ-005-01 | 分析結果をテキスト形式でコンソール出力できること | Must |
| REQ-005-02 | 分析結果をPNG画像として保存できること | Should |

### REQ-006: 球速分析機能 ★v1.1追加
| ID | 要件 | 優先度 |
|---|---|---|
| REQ-006-01 | 球種別の平均球速・最速・最遅を集計できること | Must |
| REQ-006-02 | 全投球中の最速球（球種と速度）を返せること | Must |
| REQ-006-03 | 球速データが存在しない場合は空dictまたはNoneを返すこと（NFR-003準拠） | Must |

---

## 4. 非機能要件

| ID | 要件 |
|---|---|
| NFR-001 | Python 3.10 以上で動作すること |
| NFR-002 | ユニットテストのカバレッジが80%以上であること |
| NFR-003 | 投球回が0の場合にZeroDivisionErrorを発生させないこと |
| NFR-004 | データが存在しない投手名を指定した場合、適切なエラーメッセージを返すこと |

---

## 5. 境界値・エラーケース

| ケース | 期待動作 |
|---|---|
| 投球回 = 0 | ERA/WHIP/FIPは `None` を返す |
| 奪三振 = 0 | K/9 = 0.0, K/BB = None |
| データなし投手 | `PitcherNotFoundError` を送出 |
| 全球が同一球種 | シーケンス行列は自己遷移100% |
| 球速データなし（release_speed列が全NaN） | by_pitch_type()は空dict, fastest_pitch()はNone |
| 投球データが空DataFrame | 球速分析は空dict / None を返す |

---

## 6. トレーサビリティ

| 要件ID | 対応設計 | 対応テスト |
|---|---|---|
| REQ-001-01 | `fetcher/statcast_client.py` | `test_fetcher.py::test_fetch_by_name` |
| REQ-002-01 | `analyzer/pitch_mix.py` | `test_analyzer.py::test_pitch_mix_ratio` |
| REQ-002-03 | `analyzer/pitch_mix.py` | `test_analyzer.py::test_pitch_mix_by_count` |
| REQ-003-01 | `analyzer/sequence.py` | `test_analyzer.py::test_sequence_matrix` |
| REQ-006-01 | `analyzer/velocity.py` | `test_velocity.py::TestVelocityByPitchType` |
| REQ-006-02 | `analyzer/velocity.py` | `test_velocity.py::TestFastestPitch` |
| REQ-006-03 | `analyzer/velocity.py` | `test_velocity.py::test_empty_returns_none` |
| NFR-003 | `analyzer/metrics.py` | `test_metrics.py::test_era_zero_innings` |
