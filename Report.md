
## システム概要
このプラットフォームは、ライブオークション向けのAI自動出品生成システムです。  
出品者が商品画像をアップロードすると、AIが類似商品データベースを検索し、過去の販売実績に基づいた最適なタイトル・説明文・価格を自動生成します。

---

## アーキテクチャ

```plaintext
┌─────────────────────────────────────────────────────────────────┐
│                        フロントエンド                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │ 入力フォーム   │→│ 進捗表示      │→│ プレビュー/編集        │   │
│  └──────────────┘  └──────────────┘  └──────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │ API Call
┌────────────────────────────▼────────────────────────────────────┐
│                     バックエンド API                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              /api/generate-listing                        │   │
│  │  1. 画像分析 → 2. ベクトル検索 → 3. AI生成 → 4. 検証      │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                      データ層                                    │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────────┐   │
│  │ ベクトルDB       │  │ 商品DB (Oracle) │  │ AI Model       │   │
│  │ (pgvector)      │  │ 販売履歴         │  │ (OpenAI)       │   │
│  └─────────────────┘  └─────────────────┘  └────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
````

---

## コア機能の詳細

### 1. ベクトル検索エンジン (`/lib/vector-search.ts`)

```typescript
// 類似商品検索の流れ
1. 入力テキスト/画像 → 埋め込みベクトル生成
2. ベクトルDB内でコサイン類似度検索
3. カテゴリ・状態でフィルタリング
4. 類似度スコア付きで結果返却

// 主要機能
- searchSimilarProducts() - テキストベースの類似検索
- searchByImageEmbedding() - 画像埋め込みベースの検索
- hybridSearch() - テキスト+画像のハイブリッド検索
- calculatePriceRange() - 類似商品の価格統計計算
```

### 2. AI生成パイプライン (`/lib/ai-pipeline.ts`)

4段階のパイプラインで出品情報を生成:

| ステップ   | 機能   | 出力             |
| ------ | ---- | -------------- |
| Step 1 | 画像分析 | 商品特徴、状態、ブランド推定 |
| Step 2 | 類似検索 | 過去の類似商品リスト     |
| Step 3 | 出品生成 | タイトル、説明文、価格    |
| Step 4 | 品質検証 | ハルシネーションチェック   |

**ハルシネーション防止機構:**

```typescript
// 生成内容が類似商品データに基づいているか検証
validateListingQuality(listing, similarProducts) {
  // タイトルに存在しない機能が含まれていないか
  // 価格が類似商品の範囲内か
  // 説明文に虚偽の情報がないか
}
```

### 3. 価格最適化アルゴリズム

```typescript
calculateOptimalPrice(similarProducts, condition) {
  // 1. 状態による調整係数
  const conditionMultiplier = {
    'new': 1.0,
    'like-new': 0.9,
    'good': 0.75,
    'fair': 0.6,
    'poor': 0.4
  }

  // 2. 類似商品の加重平均（類似度で重み付け）
  const weightedAvg = Σ(price × similarity) / Σ(similarity)

  // 3. 最終価格 = 加重平均 × 状態係数
  return Math.round(weightedAvg × conditionMultiplier[condition])
}
```

---

## フロントエンド構成

### コンポーネント階層

```plaintext
ListingGeneratorPage (page.tsx)
├── Header (統計情報、ユーザーメニュー)
├── ListingGenerator
│   ├── ListingInputForm (画像/キーワード入力)
│   ├── GenerationProgress (4ステップ進捗)
│   ├── SimilarProducts (類似商品カード)
│   └── ListingPreview (生成結果プレビュー)
└── ListingEditor (編集モーダル)
    ├── 画像ギャラリー
    ├── タイトル/説明文エディタ
    ├── 価格調整スライダー
    └── 公開アクション
```

### 状態管理フロー

```typescript
// メインの状態遷移
idle → uploading → analyzing → searching → generating → validating → complete

// 各ステップで更新されるデータ
{
  progress: { step, message, percentage },
  similarProducts: SimilarProduct[],
  generatedListing: GeneratedListing,
  isEditing: boolean
}
```

---

## API エンドポイント

### POST `/api/generate-listing`

**リクエスト:**

```typescript
{
  images: string[],     // Base64画像データ
  keywords: string,     // 商品キーワード
  category: string,     // カテゴリID
  condition: string     // 商品状態
}
```

**レスポンス (ストリーミング):**

```typescript
// 進捗イベント
{ type: 'progress', step: 1-4, message: string }

// 類似商品
{ type: 'similar_products', products: SimilarProduct[] }

// 最終結果
{ type: 'result', listing: GeneratedListing }
```

### POST `/api/search-similar`

キーワードのみで類似商品を検索するシンプルなエンドポイント。

---

## データモデル

### GeneratedListing

```typescript
interface GeneratedListing {
  title: string           // AI生成タイトル
  description: string     // AI生成説明文
  suggestedPrice: number  // 推奨開始価格
  priceRange: { min: number, max: number, average: number }
  confidence: number      // AI信頼度スコア
  category: string
  condition: string
  specifications: Record<string, string>
  keywords: string[]
  similarProducts: SimilarProduct[]
}
```

### SimilarProduct

```typescript
interface SimilarProduct {
  id: string
  title: string
  soldPrice: number
  soldDate: string
  similarity: number      // 0-1の類似度スコア
  condition: string
  imageUrl: string
}
```

---

## 拡張ポイント

1. **ベクトルDB統合**: 現在モックデータですが、`VectorSearchEngine` クラスを実装すれば pgvector / Pinecone 等に接続可能
2. **AI モデル切替**: `ai-pipeline.ts` のモデル指定を変更するだけで異なるLLMに対応
3. **データベース統合**: `mock-data.ts` を Oracle / MySQL 接続に置き換え可能
4. **リアルタイム価格更新**: WebSocket でマーケット価格の変動を反映

```
