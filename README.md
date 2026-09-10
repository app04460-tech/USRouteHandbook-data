# USRouteHandbook-data

iOSアプリ「アメリカ国道便覧」が使用する、US Route (U.S. Numbered Highway)
本線208路線の加工済みデータ (`highways.json`) を公開するリポジトリです。

このデータは OpenStreetMap のデータを加工して作成した派生データベースであり、
**Open Database License (ODbL) 1.0** のもとで公開します (`LICENSE` 参照)。

## 出典

- **路線の形状・距離・通過州**: © OpenStreetMap contributors (ODbL)
  https://www.openstreetmap.org/copyright
  - Overpass API から US Route のルートリレーション
    (`type=route, route=road, network=US:US`) を州単位で取得
    (48州 + DC の49クエリ。取得日: 2026年9月9日)
  - 重用区間の補完には同時に取得した Interstate (`network=US:I`) を使用
- **州の帰属判定**: U.S. Census Bureau, Cartographic Boundary Files
  (`cb_2023_us_state_500k`。パブリックドメイン)
- **検証用の公式延長・通過州**: Wikipedia 各路線記事の infobox
  (データそのものには含まれません。生成時の突き合わせにのみ使用)

距離はOSMの道路形状から算出した推定値で、公称値とは算出方法が異なります
(詳細は下記「距離の算出」)。

## 生成手順

生成スクリプトはアプリ本体リポジトリの `scripts/` にあります。

1. `python3 scripts/download_osm.py`
   — Overpass API から48州+DCの生データを `rawdata/osm/<STATE>.json` に取得し、
   取得日時を `rawdata/osm/snapshot.json` に記録
2. `python3 scripts/preprocess_osm.py`
   — リレーションの束ね直し・上下線の重複除去・鎖の構築・重用区間の縫合・
   通過州の判定・Douglas-Peucker 間引き (許容誤差0.0002度) を行い
   `highways.json` を出力

州境界データ (Census の KML) は `rawdata/` に配置します。

### 収録範囲

- US Route **本線のみ**。バナールート (Business / Alternate / Bypass / Truck / Spur)
  と歴史路線は含みません
- US 9W・US 11E のようなサフィックス付きの分岐本線は独立した路線として収録します
- US Route が存在しないアラスカ・ハワイ・プエルトリコは対象外です

### 距離の算出

- 双方向wayは全長、一方通行wayは実測した並走本数 m で按分して合算します
  (通常の上下線分離区間では m=2)
- 海上区間 (フェリー) は距離に含めず、`ferryMi` / `ferryKm` に分けています
- OSM のリレーションに Interstate との重用区間が欠けている箇所は、
  Interstate の形状で最短経路を探索して補完しています
  (経路長が直線距離の1.5倍以内の場合のみ採用)
- 5km未満の途切れは直線で補完しています

## フォーマット概要

```json
{
  "source": "© OpenStreetMap contributors (ODbL 1.0)",
  "osmSnapshotDate": "2026-09-09",
  "schemaVersion": "osm-1.4",
  "highways": [
    {
      "id": "us-421",
      "sign": "US 421",
      "signType": "U",
      "signNumber": "421",
      "states": ["NC", "TN", "VA", "KY", "IN"],
      "originState": "NC",
      "terminalState": "IN",
      "originLat": 33.96013, "originLon": -77.93853,
      "terminalLat": 41.6804, "terminalLon": -86.89406,
      "stateCount": 5,
      "lengthMi": 960.5,
      "lengthKm": 1545.8,
      "isLoop": false,
      "chainCount": 1,
      "polylines": [ [[-77.93853, 33.96013], ...] ]
    }
  ]
}
```

| フィールド | 内容 |
| --- | --- |
| `osmSnapshotDate` | OSM データの取得日 |
| `schemaVersion` | スキーマ版 (`osm-1.4`) |
| `id` / `sign` / `signNumber` | 路線の識別子・表示名・番号 (サフィックスを含む) |
| `signType` | `"U"` (US Route) |
| `states` | 通過州の略号。走行方向 (奇数は南→北 / 偶数は西→東) の順 |
| `originState` / `terminalState` | 起点州・終点州。奇数番号は南端、偶数番号は西端が起点 |
| `originLat` / `originLon` / `terminalLat` / `terminalLon` | 起終点の座標 (環状路線は `null`)。リレーションが行き止まる点のうち最も離れた2点 |
| `stateCount` | 通過州数 |
| `lengthMi` / `lengthKm` | 距離 (海上区間を含まない) |
| `isLoop` | 環状路線かどうか |
| `chainCount` | 連続した区間 (鎖) の本数。2以上なら不連続路線 |
| `discontinuityType` | 不連続の分類 (`chainCount >= 2` のみ)。`official` = 公式の不連続 / `osm` = OSMのリレーション未整備 / `mixed` = 両方を含む |
| `discontinuities` | 途切れている箇所ごとの内訳。`type` (official / osm)、`km` (直線距離)、`note` (内容) |
| `polylines` | 描画用のポリライン。座標は `[経度, 緯度]` の順、小数第5位 |
| `ferryMi` / `ferryKm` / `ferryPolylines` | 海上区間 (US 9 / US 10 のみ) |

## ライセンス

このデータベースは Open Database License (ODbL) 1.0 のもとで提供されます。
元データである OpenStreetMap の著作権表示 (© OpenStreetMap contributors) を
保持してください。

- ODbL 1.0 全文: `LICENSE`
- https://opendatacommons.org/licenses/odbl/1-0/
- https://www.openstreetmap.org/copyright

## データ更新の手順

OSM を再取得して更新するときは、アプリ本体と本リポジトリの両方に反映します。

1. `python3 scripts/download_osm.py --force` — 48州+DC を再取得 (約50分)
2. `python3 scripts/preprocess_osm.py` — `highways.json` を再生成 (約20分)
3. サマリで確認する
   - 公式延長の警告ゼロ (定義差の US 11 を除く)
   - 通過州の不一致ゼロ
   - 「残置承認以外の間隙 0箇所」
   - 縫合・直線補完の箇所数に不自然な増減がないこと
4. `python3 scripts/verify_endpoints.py` — 起終点マーカーの検証
   (全路線がポリラインの端に載っているか / 公式の起終点都市との突き合わせ)
5. 承認リスト (`KNOWN_DISCONTINUOUS` / `ACCEPTED_OSM_GAPS`) を見直す
   — OSM 側の整備で解消した間隙はリストから削除し、新たに出た間隙は位置を
   確認して分類を追加する
6. 本リポジトリに `highways.json` をコピーしてコミットする
7. スキーマを変えた場合は `schemaVersion` を上げ、上記のフォーマット表も更新する
