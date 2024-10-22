---
title: "Go 1.22でTLS通信が失敗した話"
emoji: "🙉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

## イントロダクション

とある外部システムAPIとのHTTPS通信を試みた際に次のエラーが発生しました。

```
remote error: tls: handshake failure
```

解決するのに少しハマったので、備忘録としてまとめます。

## 環境

- Go 1.22
- Echo v4.12.0

## エラーの概要と原因

調査を進めたところ、Go 1.22からのセキュリティ強化が原因で、TLS 1.2以前のRSA暗号スイートがデフォルトから削除されたことがわかりました。
(正確にはECDHEをサポートしない暗号スイートが削除されています)
これにより、古いサーバーや互換性のないサーバーとの通信でエラーが発生するようになったのです。

## Go 1.22 での仕様変更

Go 1.22 の[リリースノート](https://tip.golang.org/doc/go1.22)を見ると以下のような記載があります。

> crypto/tls
> ConnectionState.ExportKeyingMaterial will now return an error unless TLS 1.3 is in use, or the extended_master_secret extension is supported > by both the server and client. crypto/tls has supported this extension since Go 1.20. This can be disabled with the tlsunsafeekm=1 GODEBUG 
> setting.
> 
> By default, the minimum version offered by crypto/tls servers is now TLS 1.2 if not specified with config.MinimumVersion, matching the 
> behavior of crypto/tls clients. This change can be reverted with the tls10server=1 GODEBUG setting.
> 
> By default, cipher suites without ECDHE support are no longer offered by either clients or servers during pre-TLS 1.3 handshakes. This change 
> can be reverted with the tlsrsakex=1 GODEBUG setting.

Go 1.22では、ECDHEをサポートしない暗号スイートがデフォルトから削除されました。
これは前方秘匿性（Forward Secrecy）を提供してしないために、過去の通信データが後から解読されるリスクを回避するためのセキュリティ強化です。

具体的には以下のような変更がおこなわれました。

https://github.com/golang/go/issues/63413

https://go-review.googlesource.com/c/go/+/544336

※TLS鍵交換方式については以下のサイトが大変参考になりました。
https://www.serotoninpower.club/archives/360/#rsa%E9%8D%B5%E4%BA%A4%E6%8F%9B


## 接続先サーバーの設定確認（OpenSSL）

接続先サーバーがTLS 1.2の非推奨暗号スイートを使用しているかどうか確認するには、OpenSSLを使います。

```
openssl s_client -connect example.com:443
```

実行結果(一部抜粋)

```
New, TLSv1.2, Cipher is AES256-SHA256
```

サーバーはTLS 1.2と「AES256-SHA256」という暗号スイートを使用していることがわかりました。

## 一時的な回避策

:::message alert
この方法はあくまで一時的な回避策であり、セキュリティ上のリスクがあります。
できるだけ早く接続先サーバー側でより安全な暗号スイートをサポートするように設定を更新することが望ましいです。
:::

サーバー側の更新がすぐにできない場合、クライアント側で以下のように一時的な対応をすることが可能です。

```go
package main

import (
    "crypto/tls"
    "net/http"
)

func main() {
    client := &http.Client{
        Transport: &http.Transport{
            TLSClientConfig: &tls.Config{
                MinVersion:   tls.VersionTLS12,
                CipherSuites: []uint16{tls.TLS_RSA_WITH_AES_256_CBC_SHA256},
            },
        },
    }

    resp, err := client.Get("https://example.com")
    if err != nil {
        panic(err)
    }
    defer resp.Body.Close()
    println("Status:", resp.Status)
}
```

tls.ConfigでCipherSuitesを明示的に指定することで、TLSハンドシェイクが成功し、通信が可能になりました。

## まとめ

Go 1.22へのアップデートにより、デフォルトのTLS 1.2のRSA暗号スイートが削除されたため、外部サーバーと通信できなくなっていました。
OpenSSLを使ってサーバーのTLS設定を確認し、クライアント側で対応する暗号スイートを指定することで一時的に問題を回避することができました。

しかし、セキュリティ上のリスクがあるため接続先サーバー側に依頼して、根本的な対応をすることが望ましいと思います。
