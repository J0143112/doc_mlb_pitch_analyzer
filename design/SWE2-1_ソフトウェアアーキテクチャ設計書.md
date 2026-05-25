# SWE2-1 ソフトウェアアーキテクチャ設計書
# MLB 投手配球分析ツール

| 項目 | 内容 |
|---|---|
| 文書ID | SWE2-1 |
| バージョン | 1.1 |
| 作成日 | 2026-05-22 |
| 更新日 | 2026-05-25 |
| 変更内容 | REQ-006対応: VelocityAnalyzer クラスを追加 |
| 上位要件 | SWE1-1 ソフトウェア要件仕様書 v1.1 |

---

## 1. モジュール構成

```
src/
├── fetcher/
│   ├── __init__.py
│   └── statcast_client.py   # REQ-001: Statcastデータ取得
├── analyzer/
│   ├── __init__.py
│   ├── pitch_mix.py          # REQ-002: 球種分析
│   ├── sequence.py           # REQ-003: 配球シーケンス分析
│   ├── metrics.py            # ERA/WHIP/FIP等の指標計算
│   └── velocity.py           # REQ-006: 球速分析 ★v1.1追加
└── visualizer/
    ├── __init__.py
    └── zone_chart.py         # REQ-004: ゾーンヒートマップ
```

---

## 2. データフロー

```
[ユーザー入力]
  投手名 / シーズン
      │
      ▼
[fetcher/statcast_client.py]
  pybaseball.statcast_pitcher()
  → キャッシュ確認 → CSVキャッシュ読込 or API取得
      │
      ▼ DataFrame（全投球データ）
      │
      ├──▶ [analyzer/pitch_mix.py]
      │       球種割合 / カウント別 / 左右別 / 決め球
      │
      ├──▶ [analyzer/sequence.py]
      │       前球→今球 遷移行列
      │
      ├──▶ [analyzer/metrics.py]
      │       ERA / WHIP / FIP / K9 / BB9
      │
      ├──▶ [analyzer/velocity.py]          ★v1.1追加
      │       球種別球速統計 / 最速球
      │
      └──▶ [visualizer/zone_chart.py]
              ゾーンヒートマップ PNG出力
```

---

## 3. 各モジュール設計

### 3-1. `fetcher/statcast_client.py`

```python
class StatcastClient:
    def get_pitcher_data(
        self,
        pitcher_name: str,
        season: int,
        use_cache: bool = True
    ) -> pd.DataFrame:
        """
        投手名・シーズンを指定してStatcastデータを返す。
        キャッシュがあれば使用し、なければAPIから取得してキャッシュ保存。

        Raises:
            PitcherNotFoundError: 対象投手が見つからない場合
        """
```

### 3-2. `analyzer/pitch_mix.py`

```python
class PitchMixAnalyzer:
    def __init__(self, data: pd.DataFrame): ...

    def overall_mix(self) -> dict[str, float]:
        """球種別使用率 {pitch_type: ratio}"""

    def by_count(self, balls: int, strikes: int) -> dict[str, float]:
        """カウント別球種使用率"""

    def by_batter_side(self, side: str) -> dict[str, float]:
        """左右打者別球種使用率 side='L' or 'R'"""

    def strikeout_pitch(self) -> dict[str, float]:
        """奪三振の決め球割合"""
```

### 3-3. `analyzer/sequence.py`

```python
class SequenceAnalyzer:
    def __init__(self, data: pd.DataFrame): ...

    def transition_matrix(self) -> pd.DataFrame:
        """前球→今球 遷移確率行列"""

    def next_pitch_top3(self, prev_pitch: str) -> list[tuple[str, float]]:
        """ある球種の後に投げる球種TOP3"""
```

### 3-4. `analyzer/metrics.py`

```python
def calc_era(earned_runs: int, innings_pitched: float) -> float | None:
    """ERA計算。innings_pitched=0の場合はNoneを返す（REQ NFR-003）"""

def calc_whip(walks: int, hits: int, innings_pitched: float) -> float | None:
    """WHIP計算"""

def calc_fip(hr: int, bb: int, hbp: int, k: int,
             innings_pitched: float, fip_constant: float = 3.10) -> float | None:
    """FIP計算"""
```

### 3-5. `analyzer/velocity.py` ★v1.1追加（REQ-006対応）

```python
class VelocityAnalyzer:
    def __init__(self, data: pd.DataFrame) -> None: ...

    def by_pitch_type(self) -> dict[str, dict[str, float]]:
        """
        球種別球速統計を返す。（REQ-006-01）

        Returns:
            {pitch_type: {"mean": float, "max": float, "min": float}}
            release_speed列なし/全NaN/空DataFrameの場合は空dict（REQ-006-03）
        """

    def fastest_pitch(self) -> tuple[str, float] | None:
        """
        全投球中の最速球を返す。（REQ-006-02）

        Returns:
            (pitch_type, speed) のタプル。データなしはNone（REQ-006-03）
        """
```

**設計上の注意点**：
- `release_speed` 列が存在しない場合・全行NaNの場合は安全に空を返す（NFR-003と同様の防御的設計）
- 速度は小数点1桁に丸める（mph単位で0.1精度で十分）

---

## 4. 例外クラス

```python
class PitcherNotFoundError(Exception):
    """指定した投手が見つからない場合"""

class InsufficientDataError(Exception):
    """分析に十分なデータがない場合（投球数が少なすぎる等）"""
```

---

## 5. 設定ファイル（conf/config.yaml）

```yaml
cache:
  enabled: true
  dir: ".cache/statcast"

analysis:
  min_pitches: 100        # 分析に必要な最小投球数
  fip_constant: 3.10      # FIP定数（シーズンにより変動）

target_pitchers:
  - name: "Yoshinobu Yamamoto"
    id: 673548
  - name: "Shota Imanaga"
    id: 681867
```

---

## 6. 要件トレーサビリティ

| 要件ID | 対応モジュール | メソッド |
|---|---|---|
| REQ-001-01 | fetcher/statcast_client.py | `get_pitcher_data()` |
| REQ-001-02 | fetcher/statcast_client.py | キャッシュ保存処理 |
| REQ-002-01 | analyzer/pitch_mix.py | `overall_mix()` |
| REQ-002-02 | analyzer/pitch_mix.py | `by_count()` |
| REQ-002-03 | analyzer/pitch_mix.py | `by_batter_side()` |
| REQ-002-04 | analyzer/pitch_mix.py | `strikeout_pitch()` |
| REQ-003-01 | analyzer/sequence.py | `transition_matrix()` |
| REQ-003-02 | analyzer/sequence.py | `next_pitch_top3()` |
| REQ-004-01 | visualizer/zone_chart.py | `plot_zone()` |
| REQ-006-01 | analyzer/velocity.py | `by_pitch_type()` |
| REQ-006-02 | analyzer/velocity.py | `fastest_pitch()` |
| REQ-006-03 | analyzer/velocity.py | 両メソッドの空データ処理 |
| NFR-003 | analyzer/metrics.py | 全calc_*関数 |
