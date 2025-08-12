# レート制御アルゴリズム設計

## eBPF環境での制約

### 制限事項
- ループ制限（5.3以前は4096命令、以降は100万命令）
- スタックサイズ制限（512バイト）
- 動的メモリ割り当て不可
- 浮動小数点演算不可
- 複雑な制御フロー制限

### 利用可能なリソース
- eBPF Maps（高速なkey-value store）
- 原子操作（atomic operations）
- 高精度タイマー（bpf_ktime_get_ns）
- Per-CPU変数

## アルゴリズム候補比較

### 1. Token Bucket Algorithm
**概要**: 一定レートでトークンを補充し、リクエスト時にトークンを消費

**メリット**:
- バーストトラフィック許可
- 実装が比較的シンプル
- メモリ効率が良い

**デメリット**:
- 時間計算が必要
- 精密な補充タイミング制御が困難

**eBPF実装適性**: ⭐⭐⭐⭐

```rust
struct TokenBucket {
    tokens: u32,           // 現在のトークン数
    capacity: u32,         // バケット容量
    refill_rate: u32,      // 補充レート (tokens/sec)
    last_refill: u64,      // 最後の補充時刻 (ns)
}
```

### 2. Fixed Window Counter
**概要**: 固定時間窓内でのリクエスト数をカウント

**メリット**:
- 実装が最もシンプル
- メモリ使用量最小
- 高速処理

**デメリット**:
- 窓境界でのバースト問題
- 精度が低い

**eBPF実装適性**: ⭐⭐⭐⭐⭐

```rust
struct FixedWindow {
    count: u32,            // 現在窓のカウント
    window_start: u64,     // 窓開始時刻 (ns)
    window_size: u64,      // 窓サイズ (ns)
    limit: u32,            // 制限値
}
```

### 3. Sliding Window Log
**概要**: 各リクエストのタイムスタンプを記録し、時間窓をスライド

**メリット**:
- 高精度
- 真のスライディング窓

**デメリット**:
- メモリ使用量大
- 複雑な実装
- eBPFでの実装困難

**eBPF実装適性**: ⭐⭐

### 4. Sliding Window Counter
**概要**: 複数の小さな窓を組み合わせて近似的なスライディング窓を実現

**メリット**:
- 精度と効率のバランス
- メモリ効率良好

**デメリット**:
- 実装複雑度中程度
- 近似値

**eBPF実装適性**: ⭐⭐⭐

```rust
struct SlidingWindow {
    buckets: [u32; 10],    // 10個の小窓
    current_bucket: u32,   // 現在の窓インデックス
    bucket_size: u64,      // 各窓のサイズ (ns)
    last_update: u64,      // 最後の更新時刻
    limit: u32,            // 制限値
}
```

## 推奨アルゴリズム選定

### Primary: Token Bucket Algorithm
**選定理由**:
1. eBPF環境での実装適性が高い
2. バーストトラフィックに対応
3. 業界標準的なアプローチ
4. メモリ効率が良い

### Secondary: Fixed Window Counter
**選定理由**:
1. 最高のパフォーマンス
2. 実装の単純さ
3. デバッグの容易さ
4. 初期実装に適している

## 実装戦略（修正版）

### 設計方針: システム全体統一
**理由**: 予測可能性・パフォーマンス・運用性を重視し、システム全体で単一アルゴリズムを使用

### Phase 1: Token Bucket統一実装
- システム全体でToken Bucketアルゴリズムを使用
- クライアント別パラメータ（rate_limit, burst_size）のみ個別設定
- 高速・安定した動作を実現

### Phase 2: 運用最適化
- 監視・アラート機能の充実
- パフォーマンスチューニング
- 運用ツールの整備

### Phase 3: 必要に応じた拡張検討
- 運用経験を基に、本当にマルチアルゴリズムが必要かを判断
- 明確な要件がある場合のみ拡張を実装

## パフォーマンス考慮事項

### 最適化ポイント
1. **原子操作の最小化**: 可能な限りper-CPU変数を使用
2. **時間計算の効率化**: ナノ秒単位での高精度計算
3. **Map操作の最適化**: lookup回数の最小化
4. **分岐の最小化**: 予測可能な制御フロー

### メモリレイアウト
```rust
// Cache-friendly構造体設計
#[repr(C, packed)]
struct RateLimitState {
    // Hot path data (frequently accessed)
    current_tokens: u32,
    last_update: u64,
    
    // Cold path data (configuration)
    capacity: u32,
    refill_rate: u32,
    burst_size: u32,
}
```

## 次のステップ
1. Token Bucket詳細設計
2. eBPF Map構造の更新
3. 実装プロトタイプ作成
