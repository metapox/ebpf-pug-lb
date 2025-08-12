# eBPF Maps構造設計

## Map一覧

### 1. CLIENT_CONFIG_MAP
**目的**: クライアント別のレート制御設定
**型**: HashMap
**キー**: ClientId (u64)
**値**: ClientConfig

```rust
#[repr(C)]
#[derive(Clone, Copy)]
struct ClientConfig {
    rate_limit: u32,        // requests per second
    current_tokens: u32,    // 現在のトークン数
    last_refill: u64,       // 最後の補充時刻 (nanoseconds)
    burst_size: u32,        // バーストサイズ
    priority: u8,           // 優先度 (0: low, 1: normal, 2: high)
    _padding: [u8; 3],      // アライメント調整
}
```

**注意**: システム全体でToken Bucketアルゴリズムに統一
    burst_size: u32,        // max burst requests
    priority: u8,           // 0-255 (higher = more priority)
    enabled: bool,          // rate limiting enabled
    algorithm: u8,          // 0=TokenBucket, 1=LeakyBucket
}
```

### 2. RATE_BUCKETS_MAP
**目的**: Token/Leaky Bucket状態管理
**型**: HashMap
**キー**: ClientId (u64)
**値**: BucketState

```rust
#[repr(C)]
#[derive(Clone, Copy)]
struct BucketState {
    tokens: u32,            // current token count
    last_refill: u64,       // last refill timestamp (nanoseconds)
    total_requests: u64,    // total request count
    dropped_requests: u64,  // dropped request count
}
```

### 3. STATISTICS_MAP
**目的**: リアルタイム統計情報
**型**: HashMap
**キー**: ClientId (u64)
**値**: ClientStats

```rust
#[repr(C)]
#[derive(Clone, Copy)]
struct ClientStats {
    requests_per_sec: u32,      // current RPS
    avg_response_time: u32,     // average response time (μs)
    last_request_time: u64,     // last request timestamp
    consecutive_drops: u32,     // consecutive dropped requests
    status: u8,                 // 0=Normal, 1=Limited, 2=Blocked
}
```

### 4. GLOBAL_CONFIG_MAP
**目的**: グローバル設定
**型**: Array
**インデックス**: 0 (single entry)
**値**: GlobalConfig

```rust
#[repr(C)]
#[derive(Clone, Copy)]
struct GlobalConfig {
    default_rate_limit: u32,    // default rate limit
    max_clients: u32,           // maximum tracked clients
    cleanup_interval: u32,      // cleanup interval (seconds)
    log_level: u8,              // logging level
    enabled: bool,              // global enable/disable
}
```

### 5. CLIENT_LOOKUP_MAP
**目的**: HTTP HeaderからClientIdへのマッピング
**型**: HashMap
**キー**: HeaderHash (u64) - Client-ID header値のハッシュ
**値**: ClientId (u64)

### 6. PERF_EVENT_MAP
**目的**: ユーザー空間への非同期通知
**型**: PerfEventArray
**用途**: 統計更新、エラー通知、デバッグ情報

## Map設計の考慮事項

### サイズ制限
- CLIENT_CONFIG_MAP: 最大100,000エントリ
- RATE_BUCKETS_MAP: 最大100,000エントリ  
- STATISTICS_MAP: 最大100,000エントリ
- CLIENT_LOOKUP_MAP: 最大100,000エントリ

### パフォーマンス最適化
- **LRU eviction**: 古いエントリの自動削除
- **Pre-allocation**: 初期化時にメモリ確保
- **Batch updates**: 統計情報のバッチ更新

### 同期・一貫性
- **Atomic operations**: カウンタ更新時
- **Lock-free design**: 高スループット維持
- **Eventually consistent**: 統計情報の整合性

## データアクセスパターン

### 高頻度アクセス
1. RATE_BUCKETS_MAP (パケット毎)
2. CLIENT_LOOKUP_MAP (パケット毎)
3. CLIENT_CONFIG_MAP (パケット毎)

### 中頻度アクセス
1. STATISTICS_MAP (統計更新時)
2. GLOBAL_CONFIG_MAP (設定変更時)

### 低頻度アクセス
1. PERF_EVENT_MAP (イベント発生時)
