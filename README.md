# dustalk-api — Dustalk 連携 API 仕様

PhoneAgent（電話）・DustalkChat（チャット）・ImageModel（画像認識）が **Dustalk 本体**や
相互に対して呼ぶ API 契約（OpenAPI）を**集約・公開**するリポジトリです。実装コードは含みません。

> **契約の正本は各 OpenAPI ファイル**（機械可読）。エンドポイント・認証は **暫定**
> （Dustalk バックエンド確定後に更新）。ImageModel は**実装済みの稼働サービス**。

## 公開ドキュメント（URL）

`main` へ push すると GitHub Actions が Redoc をビルドし GitHub Pages に公開します。

- 一覧: https://developerunion.github.io/dustalk-api/
- 電話: https://developerunion.github.io/dustalk-api/phone/
- チャット: https://developerunion.github.io/dustalk-api/chat/
- 画像認識: https://developerunion.github.io/dustalk-api/imagemodel/

> 初回のみ: **Settings → Pages → Source = GitHub Actions** を有効化。

## 構成

```
.
├── README.md                     ← このファイル（索引）
├── redocly.yaml                  ← 3 サービスの定義（lint / build-docs）
├── .github/workflows/pages.yml   ← Redoc ビルド → Pages 公開
├── phone/                        ← PhoneAgent（電話）→ Dustalk 本体（暫定・自動生成）
│   ├── openapi.yaml              ← OpenAPI 3.1（paths へ $ref）
│   ├── paths/{intakes,requests,customers}.yaml
│   └── components/schemas.yaml
├── chat/                         ← DustalkChat（チャット）→ Dustalk 本体（暫定）
│   └── openapi.chat.yaml         ← OpenAPI 3.0.3
└── imagemodel/                   ← DustalkChat → ImageModel（画像認識・実装済）
    └── openapi.yaml              ← OpenAPI 3.0.3
```

## サービス一覧

| 経路 | 仕様 | 状態 | 真実源 / 管理 |
|---|---|---|---|
| **PhoneAgent（電話）** → Dustalk 本体 | [`phone/openapi.yaml`](phone/openapi.yaml)（3.1） | 暫定 | PhoneAgent `server/models.py`（Pydantic）から自動生成。`python -m scripts.gen_api_models`。手動編集不可 |
| **DustalkChat（チャット）** → Dustalk 本体 | [`chat/openapi.chat.yaml`](chat/openapi.chat.yaml)（3.0.3） | 暫定 | DustalkChat `src/lib/slots/types.ts` の `Slots` 型を起点に手動 |
| **DustalkChat** → ImageModel（画像認識） | [`imagemodel/openapi.yaml`](imagemodel/openapi.yaml)（3.0.3） | 実装済 | `ImageModel/image-model` 実装を起点に手動 |

PhoneAgent 経路と DustalkChat 経路は同一ドメイン（回収申し込み）の別表現。チャット経路の方が
情報量が多い（品目ごとの依頼先・希望時間帯・構造化住所）。**将来は単一契約への収斂を想定**。

> 補足ドキュメント（API契約ではないため本リポには含めない）:
> 電話経路のフィールド表（`data-models.md`）と暫定 DB 設計（`db-schema-provisional.md`）は
> PhoneAgent リポの `api/phone/` にあります。

## 認証（暫定）

- PhoneAgent（`phone/openapi.yaml`）: API キー認証 `X-API-Key: <key>`（暫定・未確定）。
- DustalkChat → Dustalk 本体（`chat/openapi.chat.yaml`）: Bearer トークン（暫定・未確定）。
- DustalkChat → ImageModel（`imagemodel/openapi.yaml`）: **認証なし**（パブリック、CORS で Origin 制限）。

## エンドポイント一覧（暫定）

### PhoneAgent 経路（[`phone/openapi.yaml`](phone/openapi.yaml)）

| メソッド | パス | 用途 | リクエスト | レスポンス | PhoneAgent ツール |
|---|---|---|---|---|---|
| `POST` | `/intakes` | 新規スポット/定期回収の受付（見積もり依頼） | `SpotIntake` | `IntakeAccepted` | `submit_intake` |
| `POST` | `/requests` | 変更/問い合わせの記録 | `ChangeRequest` | `RequestAccepted` | `submit_request` |
| `GET` | `/customers/{phone}` | 電話番号による顧客照会 | — | `LookupResult` | `look_up` |
| `PUT` | `/customers/{phone}` | 顧客マスタの作成/更新（受付後の同期） | `CustomerRecord` | `CustomerRecord` | `upsert_customer` |

### DustalkChat 経路（[`chat/openapi.chat.yaml`](chat/openapi.chat.yaml)）

| メソッド | パス | 用途 | リクエスト | レスポンス |
|---|---|---|---|---|
| `GET` | `/customers/lookup?phone=` | 顧客データ取得（プレフィル用） | — | `Customer` |
| `POST` | `/applications` | 申し込み依頼送信（Slots ベース） | `ApplicationRequest` | `ApplicationAccepted` |

### ImageModel 経路（[`imagemodel/openapi.yaml`](imagemodel/openapi.yaml)）

| メソッド | パス | 用途 | リクエスト | レスポンス |
|---|---|---|---|---|
| `POST` | `/api/detect` | 画像から品目検出（品目名・属性・bbox） | `multipart/form-data`（`file`） | `DetectResponse` |

エラーは各仕様の共通エラー（`ApiError` / `Error`、`code`/`message`/`detail`）。

## ローカルでプレビュー / 検証

```sh
npx @redocly/cli preview-docs phone@v1   # ブラウザ表示（phone/chat/imagemodel を指定可）
npx @redocly/cli lint                    # 3 サービスを検証
```

## 未確定事項

- エンドポイント URL・バージョニング・採番方式
- 認証方式の確定（経路間で統一するか）
- PhoneAgent 経路とチャット経路の契約を **単一契約に収斂** させるか
- `/customers/{phone}` で定期回収の進行中スケジュール（次回回収日・契約状態）を返すか
- `/intakes` で概算見積もり（`estimated_cost` / `proposed_dates`）を同期返却するか、非同期か
