---
title: "【Kubernetes】如何访问集群etcd"
author: "Tweakzx"
description: 
date: 2023-03-31T10:04:38+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
categories: Kubernetes 
tags: 
    - etcd
---

## 生成客户端证书

etcd 的证书包括了多种类型的证书，其中包括 `peer.crt` 和 `etcd-client.crt` 两种证书。`peer.crt` 证书是用于 etcd 集群中节点之间的身份验证和加密通信的，而 `etcd-client.crt` 则是用于客户端与 etcd 之间的身份验证和加密通信的。

如果你已经生成了 etcd 集群节点之间的证书，那么你可以使用以下命令生成 `etcd-client.crt`：

```shell
$ openssl genrsa -out etcd-client.key 2048
$ openssl req -new -key etcd-client.key -out etcd-client.csr \
    -subj "/CN=etcd-client/O=example.com"
$ openssl x509 -req -in etcd-client.csr -CA ca.crt -CAkey ca.key \
    -CAcreateserial -out etcd-client.crt -days 365

```

其中，`ca.crt` 和 `ca.key` 是用于签名证书的根证书和私钥，需要和 etcd 集群中节点之间的证书一起使用。`etcd-client.key` 是生成的客户端证书的私钥，`etcd-client.csr` 是证书签名请求，`etcd-client.crt` 是签名后的证书。

在生成 `etcd-client.crt` 之后，你可以将其和根证书一起用于访问 etcd，例如

```shell
import etcd3

etcd_host = 'https://192.168.0.10:2379,https://192.168.0.11:2379'
ca_cert = '/path/to/ca.crt'
client_cert = '/path/to/etcd-client.crt'
client_key = '/path/to/etcd-client.key'

# 创建 etcd3 客户端
client = etcd3.client(
    host=etcd_host,
    ca_cert=ca_cert,
    cert_key=client_key,
    cert_cert=client_cert
)

# 访问 etcd 的 API
key = '/registry/services/specs/default/kubernetes'
value, metadata = client.get(key)

# 打印结果
print(value)
```

注意，在使用 `openssl` 生成证书时，应该按照实际情况修改证书的主题和有效期限等信息。此外，生成的证书应该严格保密，避免泄露给未经授权的人员使用