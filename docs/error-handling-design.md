# エラーハンドリング設計

## 概要
eBPF L7 Rate Limiterにおける包括的なエラーハンドリング戦略とエラー分類

## エラー分類

### 1. システムレベルエラー

#### 1.1 eBPFプログラム関連
```rust
#[derive(Debug, Clone)]
pub enum EbpfError {
    // プログラムロード失敗
    ProgramLoadFailed {
        program_name: String,
        verifier_log: String,
        error_code: i32,
    },
    
    // Map作成失敗
    MapCreationFailed {
        map_name: String,
        map_type: String,
        error_code: i32,
    },
    
    // Map操作失敗
    MapOperationFailed {
        map_name: String,
        operation: String, // lookup, update, delete
        key: String,
        error_code: i32,
    },
    
    // プログラムアタッチ失敗
    AttachFailed {
        program_name: String,
        interface: String,
        hook_type: String, // tc_ingress, tc_egress, xdp
        error_code: i32,
    },
    
    // 権限不足
    PermissionDenied {
        operation: String,
        required_capability: String,
    },
}
```

#### 1.2 ネットワーク関連
```rust
#[derive(Debug, Clone)]
pub enum NetworkError {
    // インターフェース不存在
    InterfaceNotFound {
        interface_name: String,
    },
    
    // インターフェース状態異常
    InterfaceDown {
        interface_name: String,
    },
    
    // ネットワーク設定エラー
    NetworkConfigError {
        interface_name: String,
        config_type: String,
        details: String,
    },
    
    // パケット処理エラー
    PacketProcessingError {
        error_type: String,
        packet_info: String,
    },
}
```

#### 1.3 リソース関連
```rust
#[derive(Debug, Clone)]
pub enum ResourceError {
    // メモリ不足
    OutOfMemory {
        requested_size: usize,
        available_size: usize,
    },
    
    // ファイルディスクリプタ不足
    OutOfFileDescriptors {
        current_count: u32,
        max_count: u32,
    },
    
    // CPU使用率過多
    CpuOverload {
        current_usage: f64,
        threshold: f64,
    },
    
    // Map容量不足
    MapCapacityExceeded {
        map_name: String,
        current_size: u32,
        max_size: u32,
    },
}
```

### 2. アプリケーションレベルエラー

#### 2.1 設定関連
```rust
#[derive(Debug, Clone)]
pub enum ConfigError {
    // 設定ファイル読み込み失敗
    FileReadError {
        file_path: String,
        io_error: String,
    },
    
    // 設定パース失敗
    ParseError {
        file_path: String,
        line_number: u32,
        column: u32,
        message: String,
    },
    
    // 設定検証失敗
    ValidationError {
        field_path: String,
        expected: String,
        actual: String,
    },
    
    // 必須フィールド不足
    MissingRequiredField {
        field_name: String,
        section: String,
    },
    
    // 設定値範囲外
    ValueOutOfRange {
        field_name: String,
        value: String,
        min: String,
        max: String,
    },
}
```

#### 2.2 クライアント管理関連
```rust
#[derive(Debug, Clone)]
pub enum ClientError {
    // クライアント不存在
    ClientNotFound {
        client_id: String,
    },
    
    // クライアント重複
    ClientAlreadyExists {
        client_id: String,
    },
    
    // 無効なクライアントID
    InvalidClientId {
        client_id: String,
        reason: String,
    },
    
    // クライアント設定無効
    InvalidClientConfig {
        client_id: String,
        field: String,
        reason: String,
    },
    
    // クライアント数上限
    TooManyClients {
        current_count: u32,
        max_count: u32,
    },
}
```

#### 2.3 API関連
```rust
#[derive(Debug, Clone)]
pub enum ApiError {
    // 認証失敗
    AuthenticationFailed {
        reason: String,
    },
    
    // 認可失敗
    AuthorizationFailed {
        required_permission: String,
        user_permissions: Vec<String>,
    },
    
    // リクエスト形式不正
    InvalidRequest {
        field: String,
        message: String,
    },
    
    // レート制限
    RateLimitExceeded {
        client_id: String,
        current_rate: u32,
        limit: u32,
    },
    
    // サーバー内部エラー
    InternalServerError {
        error_id: String,
        message: String,
    },
}
```

### 3. ランタイムエラー

#### 3.1 パケット処理関連
```rust
#[derive(Debug, Clone)]
pub enum PacketError {
    // パケット形式不正
    MalformedPacket {
        packet_type: String,
        reason: String,
    },
    
    // ヘッダー解析失敗
    HeaderParsingFailed {
        header_type: String,
        offset: u32,
        reason: String,
    },
    
    // プロトコル未対応
    UnsupportedProtocol {
        protocol: String,
    },
    
    // パケットサイズ異常
    InvalidPacketSize {
        actual_size: u32,
        expected_range: String,
    },
}
```

## エラー処理戦略

### 1. エラー回復戦略

#### 自動回復可能エラー
```rust
pub enum RecoveryAction {
    // 再試行
    Retry {
        max_attempts: u32,
        backoff_ms: u64,
    },
    
    // フォールバック
    Fallback {
        fallback_action: String,
    },
    
    // 部分的継続
    PartialContinue {
        skip_failed_operation: bool,
    },
    
    // 設定リロード
    ReloadConfig,
    
    // プログラム再起動
    RestartProgram,
}
```

#### 回復不可能エラー
```rust
pub enum FatalAction {
    // 緊急停止
    EmergencyShutdown {
        reason: String,
    },
    
    // セーフモード移行
    SafeMode {
        limited_functionality: Vec<String>,
    },
    
    // 管理者通知
    NotifyAdmin {
        severity: String,
        message: String,
    },
}
```

### 2. エラーログ戦略

#### ログレベル定義
```rust
pub enum LogLevel {
    Trace,   // 詳細なデバッグ情報
    Debug,   // デバッグ情報
    Info,    // 一般的な情報
    Warn,    // 警告（処理は継続）
    Error,   // エラー（処理に影響）
    Fatal,   // 致命的エラー（システム停止）
}
```

#### 構造化ログ形式
```json
{
  "timestamp": "2024-08-10T06:30:00.123Z",
  "level": "ERROR",
  "component": "ebpf_loader",
  "error_type": "ProgramLoadFailed",
  "error_code": "EBPF_001",
  "message": "Failed to load TC ingress program",
  "context": {
    "program_name": "rate_limiter_tc",
    "interface": "eth0",
    "verifier_log": "invalid memory access at instruction 42"
  },
  "trace_id": "abc123def456",
  "recovery_action": "retry_with_fallback"
}
```

### 3. エラー監視・アラート

#### メトリクス定義
```rust
pub struct ErrorMetrics {
    // エラー発生回数
    pub error_count_total: Counter,
    
    // エラー種別別カウント
    pub error_count_by_type: CounterVec,
    
    // エラー発生率
    pub error_rate: Gauge,
    
    // 回復成功率
    pub recovery_success_rate: Gauge,
    
    // 平均回復時間
    pub recovery_time_seconds: Histogram,
}
```

#### アラート条件
```yaml
alerts:
  - name: "high_error_rate"
    condition: "error_rate > 0.05"
    duration: "5m"
    severity: "warning"
    
  - name: "ebpf_program_load_failure"
    condition: "error_count_by_type{type='ProgramLoadFailed'} > 0"
    duration: "0s"
    severity: "critical"
    
  - name: "memory_exhaustion"
    condition: "error_count_by_type{type='OutOfMemory'} > 0"
    duration: "0s"
    severity: "critical"
```

## エラーハンドリング実装

### 1. エラー型統合
```rust
#[derive(Debug, Clone)]
pub enum RateLimiterError {
    Ebpf(EbpfError),
    Network(NetworkError),
    Resource(ResourceError),
    Config(ConfigError),
    Client(ClientError),
    Api(ApiError),
    Packet(PacketError),
}

impl std::error::Error for RateLimiterError {}

impl std::fmt::Display for RateLimiterError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            RateLimiterError::Ebpf(e) => write!(f, "eBPF error: {}", e),
            RateLimiterError::Network(e) => write!(f, "Network error: {}", e),
            // ... 他のエラー型
        }
    }
}
```

### 2. エラーハンドラー
```rust
pub struct ErrorHandler {
    logger: Logger,
    metrics: ErrorMetrics,
    recovery_strategies: HashMap<String, RecoveryAction>,
}

impl ErrorHandler {
    pub async fn handle_error(&self, error: RateLimiterError) -> Result<(), FatalAction> {
        // エラーログ記録
        self.log_error(&error).await;
        
        // メトリクス更新
        self.update_metrics(&error).await;
        
        // 回復戦略実行
        match self.get_recovery_strategy(&error) {
            Some(action) => self.execute_recovery(action).await,
            None => Err(FatalAction::EmergencyShutdown {
                reason: format!("No recovery strategy for error: {}", error),
            }),
        }
    }
    
    async fn execute_recovery(&self, action: RecoveryAction) -> Result<(), FatalAction> {
        match action {
            RecoveryAction::Retry { max_attempts, backoff_ms } => {
                // 再試行ロジック
                for attempt in 1..=max_attempts {
                    tokio::time::sleep(Duration::from_millis(backoff_ms * attempt)).await;
                    // 再試行実行
                }
                Ok(())
            },
            RecoveryAction::Fallback { fallback_action } => {
                // フォールバック実行
                Ok(())
            },
            // ... 他の回復アクション
        }
    }
}
```

### 3. eBPFプログラム内エラーハンドリング
```rust
// eBPFプログラム内でのエラーハンドリング
#[inline(always)]
fn handle_packet_error(ctx: &TcContext, error_code: u32) -> i32 {
    // エラー統計更新
    unsafe {
        let key = 0u32;
        if let Some(stats) = ERROR_STATS.get_ptr_mut(&key) {
            (*stats).error_count += 1;
            (*stats).last_error_code = error_code;
            (*stats).last_error_time = bpf_ktime_get_ns();
        }
    }
    
    // エラーログ出力（デバッグ時のみ）
    #[cfg(debug_assertions)]
    info!(ctx, "Packet processing error: {}", error_code);
    
    // デフォルトアクション（パケット通過）
    TC_ACT_OK
}
```

## テスト戦略

### 1. エラー注入テスト
```rust
#[cfg(test)]
mod error_injection_tests {
    use super::*;
    
    #[tokio::test]
    async fn test_ebpf_program_load_failure() {
        let mut handler = ErrorHandler::new();
        let error = RateLimiterError::Ebpf(EbpfError::ProgramLoadFailed {
            program_name: "test_program".to_string(),
            verifier_log: "test error".to_string(),
            error_code: -1,
        });
        
        let result = handler.handle_error(error).await;
        assert!(result.is_ok());
    }
}
```

### 2. 回復テスト
```rust
#[tokio::test]
async fn test_automatic_recovery() {
    // 一時的なエラーからの自動回復をテスト
}

#[tokio::test]
async fn test_fallback_mechanism() {
    // フォールバック機構のテスト
}
```

## 運用時の考慮事項

### 1. エラー分析
- エラーパターンの定期的な分析
- 根本原因分析（RCA）の実施
- 予防策の策定

### 2. 監視ダッシュボード
- エラー発生率の可視化
- エラー種別の分布
- 回復時間の追跡
- システム健全性の総合指標

### 3. 運用手順書
- 各エラー種別の対応手順
- エスカレーション基準
- 緊急時の対応フロー

## 次のステップ
1. エラーハンドリング実装
2. テストケース作成
3. 監視システム構築
4. 運用手順書作成
