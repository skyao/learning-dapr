---
title: "sentry的Proto定义"
linkTitle: "Proto定义"
weight: 20
date: 2022-02-22
description: >
  proto服务定义
---



sentry 模块的 proto 服务定义在文件 `dapr/proto/sentry/v1/sentry.proto` 中。



```protobuf
service CA {
  // A request for a time-bound certificate to be signed.
  //
  // The requesting side must provide an id for both loosely based
  // And strong based identities.
  rpc SignCertificate (SignCertificateRequest) returns (SignCertificateResponse) {}
}
```

SignCertificate() 方法要求签署一个有时间限制的证书。请求方必须提供一个可以同时用于松散型身份和强势型身份的ID。

SignCertificateRequest 的定义：

```protobuf
message SignCertificateRequest {
  string id = 1;
  string token = 2;
  string trust_domain = 3;
  string namespace = 4;
  // A PEM-encoded x509 CSR.
  bytes certificate_signing_request = 5;
}
```

SignCertificateResponse 的定义：

```protobuf
message SignCertificateResponse {
  // A PEM-encoded x509 Certificate.
  bytes workload_certificate = 1;

  // A list of PEM-encoded x509 Certificates that establish the trust chain
  // between the workload certificate and the well-known trust root cert.
  repeated bytes trust_chain_certificates = 2;

  google.protobuf.Timestamp valid_until = 3;
}
```



trust_chain_certificates 是一个 PEM 编码的 x509 证书的列表，这些证书在 workload_certificate 和众所周知的信任根证书（trust root cert）之间建立信任链。
