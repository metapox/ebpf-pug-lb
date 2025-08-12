# パケット処理フロー設計

## TC Hook Point での処理フロー

```
Incoming Packet
       ↓
┌─────────────────┐
│  1. Packet      │ ← Ethernet/IP/TCP header validation
│     Validation  │
└─────────────────┘
       ↓
┌─────────────────┐
│  2. Protocol    │ ← HTTP/HTTPS detection
│     Detection   │
└─────────────────┘
       ↓
┌─────────────────┐
│  3. HTTP Header │ ← Client-ID, API-Key extraction
│     Parsing     │
└─────────────────┘
       ↓
┌─────────────────┐
│  4. Client      │ ← CLIENT_LOOKUP_MAP query
│     Lookup      │
└─────────────────┘
       ↓
┌─────────────────┐
│  5. Rate Limit  │ ← RATE_BUCKETS_MAP update
│     Check       │
└─────────────────┘
       ↓
┌─────────────────┐
│  6. Action      │ ← PASS/DROP/THROTTLE decision
│     Decision    │
└─────────────────┘
       ↓
┌─────────────────┐
│  7. Statistics  │ ← STATISTICS_MAP update
│     Update      │
└─────────────────┘
       ↓
    TC_ACT_OK / TC_ACT_SHOT
```

## 各ステップの詳細

### 1. Packet Validation
```rust
// パケット境界チェック
if data_end - data < ETH_HLEN + IP_HLEN + TCP_HLEN {
    return TC_ACT_OK; // パススルー
}

// TCP port check (HTTP: 80, HTTPS: 443)
if tcp_hdr.dest != 80 && tcp_hdr.dest != 443 {
    return TC_ACT_OK; // 対象外
}
```

### 2. Protocol Detection
```rust
// HTTP request detection
if tcp_payload.starts_with(b"GET ") || 
   tcp_payload.starts_with(b"POST ") ||
   tcp_payload.starts_with(b"PUT ") {
    protocol = HTTP;
} else {
    return TC_ACT_OK; // 非HTTP
}
```

### 3. HTTP Header Parsing
```rust
// Client-ID header extraction
let client_id = extract_header_value(tcp_payload, b"Client-ID: ");
let api_key = extract_header_value(tcp_payload, b"X-API-Key: ");

// Fallback to IP address if no header
if client_id.is_none() && api_key.is_none() {
    client_id = ip_hdr.saddr; // Source IP as fallback
}
```

### 4. Client Lookup
```rust
// Header hash calculation
let header_hash = hash_client_identifier(client_id);

// CLIENT_LOOKUP_MAP query
let client_id = CLIENT_LOOKUP_MAP.lookup(&header_hash)?;

// CLIENT_CONFIG_MAP query
let config = CLIENT_CONFIG_MAP.lookup(&client_id)?;
```

### 5. Rate Limit Check
```rust
// Current timestamp
let now = bpf_ktime_get_ns();

// RATE_BUCKETS_MAP lookup
let mut bucket = RATE_BUCKETS_MAP.lookup(&client_id)?;

// Token bucket algorithm
let time_passed = now - bucket.last_refill;
let tokens_to_add = (time_passed * config.rate_limit) / 1_000_000_000;

bucket.tokens = min(bucket.tokens + tokens_to_add, config.burst_size);
bucket.last_refill = now;

// Rate limit decision
if bucket.tokens > 0 {
    bucket.tokens -= 1;
    bucket.total_requests += 1;
    action = TC_ACT_OK; // PASS
} else {
    bucket.dropped_requests += 1;
    action = TC_ACT_SHOT; // DROP
}

// Update bucket state
RATE_BUCKETS_MAP.update(&client_id, &bucket);
```

### 6. Action Decision
```rust
match action {
    TC_ACT_OK => {
        // パケット通過
        log_debug!("PASS: client_id={}, tokens={}", client_id, bucket.tokens);
    }
    TC_ACT_SHOT => {
        // パケット破棄
        log_debug!("DROP: client_id={}, rate_limited", client_id);
    }
}
```

### 7. Statistics Update
```rust
// STATISTICS_MAP update (every N requests)
if bucket.total_requests % STATS_UPDATE_INTERVAL == 0 {
    let mut stats = STATISTICS_MAP.lookup(&client_id).unwrap_or_default();
    
    stats.requests_per_sec = calculate_rps(&bucket, now);
    stats.last_request_time = now;
    stats.status = if action == TC_ACT_SHOT { LIMITED } else { NORMAL };
    
    STATISTICS_MAP.update(&client_id, &stats);
}
```

## パフォーマンス最適化

### Fast Path Optimization
```rust
// 高頻度クライアントのキャッシュ
static mut HOT_CLIENTS: [ClientCache; 16] = [ClientCache::empty(); 16];

// LRU cache lookup first
let cache_idx = client_id as usize % 16;
if HOT_CLIENTS[cache_idx].client_id == client_id {
    // Cache hit - skip map lookup
    config = HOT_CLIENTS[cache_idx].config;
    bucket = &mut HOT_CLIENTS[cache_idx].bucket;
}
```

### Batch Processing
```rust
// 統計更新のバッチ化
if stats_batch_count >= BATCH_SIZE {
    flush_stats_batch();
    stats_batch_count = 0;
}
```

## エラーハンドリング

### Map Lookup Failures
```rust
// Graceful degradation
let config = CLIENT_CONFIG_MAP.lookup(&client_id)
    .unwrap_or(&DEFAULT_CONFIG);
```

### Memory Constraints
```rust
// Map full handling
if CLIENT_LOOKUP_MAP.is_full() {
    // LRU eviction or fallback to IP-based limiting
    evict_oldest_client();
}
```

## 処理時間目標
- **Total processing time**: < 10μs
- **Map lookups**: < 2μs each
- **Header parsing**: < 3μs
- **Rate calculation**: < 1μs
