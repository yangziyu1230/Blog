---
title: Protect the docker daemon socket
date: 2022-10-21 13:00:15
tags: docker
categories: docker
---

## 背景

前两天折腾docker服务器被黑了，发现是docker的remote API暴露在公网被攻击了，后来查询相关资料发现可以进行CA认证来使用；为了以后省事那现在就费点事折腾下吧。

## 使用OpenSSL创建ca、服务端和客户端秘钥

{% note::需将下面内容中所有`$HOST`替换为docker服务器的IP %}

首先，在docker机器上生成CA私钥和公钥：

```shell
$ openssl genrsa -aes256 -out ca-key.pem 4096
Generating RSA private key, 4096 bit long modulus (2 primes)
...........................................................................................................................................++++
....................................................................................................................................++++
e is 65537 (0x010001)
Enter pass phrase for ca-key.pem:
Verifying - Enter pass phrase for ca-key.pem:

$ openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem
Enter pass phrase for ca-key.pem:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:
State or Province Name (full name) [Some-State]:   
Locality Name (eg, city) []:
Organization Name (eg, company) [Internet Widgits Pty Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:$HOST
Email Address []:
```

现在你有一个CA，你可以创建服务器秘钥和证书签名请求：

```shell
$ openssl genrsa -out server-key.pem 4096
Generating RSA private key, 4096 bit long modulus (2 primes)
....................++++
..............................++++
e is 65537 (0x010001)
$ openssl req -subj "/CN=43.155.61.172" -sha256 -new -key server-key.pem -out server.csr
```

接下来，我们使用CA签署公钥：

```shell
$ echo subjectAltName = DNS:your DNS,IP:$HOST,IP:0.0.0.0 >> extfile.cnf
```

设置仅用于服务器身份验证：

```shell
$ echo extendedKeyUsage = serverAuth >> extfile.cnf
```

生成签名证书：

```shell
$ openssl x509 -req -days 365 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem \
 -CAcreateserial -out server-cert.pem -extfile extfile.cnf
Signature ok
subject=CN = 43.155.61.172
Getting CA Private Key
```

创建客户端密钥和证书签名请求：

```shell
$ openssl genrsa -out key.pem 4096
Generating RSA private key, 4096 bit long modulus (2 primes)
.....++++
.....................++++
e is 65537 (0x010001)
$ openssl req -subj '/CN=client' -new -key key.pem -out client.csr
```

创建新的扩展配置文件用于客户端认证：

```shell
echo extendedKeyUsage = clientAuth > extfile-client.cnf
```

生成签名证书：

```shell
$ openssl x509 -req -days 365 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem \
> -CAcreateserial -out cert.pem -extfile extfile-client.cnf
Signature ok
subject=CN = client
Getting CA Private Key
Enter pass phrase for ca-key.pem:
```

在生成cert.pem和server-cert之后。pem您可以安全地删除两个证书签名请求和扩展配置文件：

```shell
$ rm -v client.csr server.csr extfile.cnf extfile-client.cnf
```

更改文件权限

```shell
$ chmod -v 0400 ca-key.pem key.pem server-key.pem
$ chmod -v 0444 ca.pem server-cert.pem cert.pem
```

归集服务器证书：

```shell
$ cp -v ~/server-*.pem /etc/docker/ && cp -v ~/ca.pem /etc/docker/
```

配置Docker支持TLS：

```shell
vi /usr/lib/systemd/system/docker.service
ExecStart=/usr/bin/dockerd -H fd:// --tlsverify --tlscacert=/etc/docker/ca.pem --tlscert=/etc/docker/server-cert.pem --tlskey=/etc/docker/server-key.pem -H tcp://0.0.0.0:2376 --containerd=/run/containerd/containerd.sock

systemctl daemon-reload
service docker restart
```