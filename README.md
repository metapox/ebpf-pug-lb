# eBPF L7 Rate Limiter

## 目的
高スループット環境において、アプリケーションレベルの識別子（Client-ID, APIキー, ユーザーID）単位で公平・効率的なレート制御を、ユーザー空間に頼らずカーネル内で実現する。

## 概要
アプリ層の情報に基づき、カーネル空間の eBPF プログラムで即時に帯域制御・リクエスト制限・優先度制御を行う L7 レートリミッター。

## 技術スタック
- Rust + Aya (eBPF framework)
- TC/XDP for packet processing
- eBPF Maps for state management
