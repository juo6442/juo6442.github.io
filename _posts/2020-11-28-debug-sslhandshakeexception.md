---
layout: post
title: "SSLHandshakeException 이슈 보는 이야기"
categories: study
published: true
---

## 0

주의: 이 글은 SSL과 보안에 대한 심도 깊은 지식을 제공하고 있지 않습니다. 그냥 회사에서 있었던 작은 이슈에 대해 간단히 기록한 내용입니다.

## 1

회사 일로 서버에서 HTTP로 데이터를 받아오는 background 앱을 개발한 적이 있다. 우여곡절이 많았던 초기 개발 단계를 지나 몇 년이 흐른 지금. 앱은 이런저런 요구사항 변경으로 수정이 있었지만 HTTP request를 보내고 받는 부분은 평화로운 상태를 유지하고 있었다.

가끔 데이터가 안 받아와진다는 리포트가 올라왔지만 서버 쪽에서 필요한 작업이 안 되어 있던 경우가 대부분이었다. 이는 메일 한두 통이면 해결될 간단한 문제였다. 아니면 "일시적인 서버 문제"니 나중에 다시 시도하거나 - 입사 이래 계속된 인원 감축으로 모두들 바쁜 상태였으므로 주 업무가 아닌 앱의 이슈는 대충 확인하는 경향이 있었다. 그래도 결국 앱에 문제는 없었으며 이슈는 3일 안에 닫혔다.

## 2

매시간 쌓이는 메일을 정리하던 중 새 이슈가 열렸다는 메일을 봤다. 그 앱이었다. 이번에도 똑같겠지 하는 생각이 먼저 들었고 가벼운 마음으로 로그를 받아 열었다. 두 가지 다른 원인으로 동작에 실패하고 있었다. 우선 서버에서 확인이 필요한 에러 코드를 받고 있어 기계적으로 문의 메일을 한 통 썼다.

그리고 아래와 같은 로그에 눈을 돌렸다.

```
javax.net.ssl.SSLHandshakeException: java.security.cert.CertPathValidatorException: Trust anchor for certification path not found.
```

그동안 아무 문제 없었다가 갑자기 인증서 오류가? 내가 확인해 봤을 땐 이런 문제가 없었다. 재현 빈도를 알아보기 위해 재시험 요청을 했고, 두 번째 재현 로그와 함께 다시 열린 이슈를 보며 이 문제의 원인을 좀 제대로 알아봐야 마음의 평화를 찾을 수 있겠다는 생각이 들었다.

### 3

우선 서버에서 사용하는 인증서 정보를 알아보았다. 이 글에서는 Amazing Company라고 하자. 루트 인증서로 Entrust Root Certification Authority - G2를 사용하고 있다.

```
openssl s_client -connect some.amazing.domain.com:443

CONNECTED(00000003)
depth=2 C = US, O = "Entrust, Inc.", OU = See www.entrust.net/legal-terms, OU = "(c) 2009 Entrust, Inc. - for authorized use only", CN = Entrust Root Certification Authority - G2
verify return:1
depth=1 C = US, O = "Entrust, Inc.", OU = See www.entrust.net/legal-terms, OU = "(c) 2012 Entrust, Inc. - for authorized use only", CN = Entrust Certification Authority - L1K
verify return:1
depth=0 C = US, ST = Washington, L = Bellevue, O = "Amzaing Company USA, Inc.", CN = some.amazing.domain.com
verify return:1
...
```

앱이 실행되는 안드로이드 환경에는 이미 이 credential이 기본적으로 설치되어 있다. Lock screen & security - Encryption & credentials - Trusted credentials의 System 탭을 보면 확인할 수 있다.

그렇다면 테스터가 시험한 기기에서만 알 수 없는 이유로 이 credential이 설치되어 있지 않단 말인가? 하지만 로그에는 SSL 인증을 통과한 시점도 있다. 중간에 제거될 리도, 테스터가 내게 일감을 주기 위해 credential을 비활성화했을 리도 없다.

다음은 패킷 로그를 열어 무슨 일이 일어나고 있는지 살펴볼 차례다.

### 4

패킷 로그 내 DNS query를 뒤져서 테스트 당시 `some.amazing.domain.com`의 IP 주소를 찾아내 여기로 오가는 TLS 패킷만 필터링해 보았다. 실제 주소를 쓰기는 좀 그러니 있을 수 없는 주소 `111.222.333.444`라고 하자.

```
39  2020-11-23  00:54:31.346623  192.168.1.29     111.222.333.444  TLSv1.2  585   Client Hello
42  2020-11-23  00:54:31.353785  111.222.333.444  192.168.1.29     TLSv1.2  1516  Server Hello
44  2020-11-23  00:54:31.353960  111.222.333.444  192.168.1.29     TLSv1.2  1124  Certificate, Server Key Exchange, Server Hello Done
46  2020-11-23  00:54:31.358305  192.168.1.29     111.222.333.444  TLSv1.2  75    Alert (Level: Fatal, Description: Certificate Unknown)
```

46번째 패킷에서 단말이 certificate unknown의 사유로 handshaking을 거부하였다. 그럼 44번 패킷을 열어 서버에서는 대체 무슨 certificate를 보냈는지 살펴보자.

```
Certificate: 2082032730420290a00302010202085fa42f8b15862f9d300d06092a86488ff70d01010b… ([id-at-commonName=](http://id-at-commonname%3Dsome.amazing.domain.com/)some.amazing.domain.com,id-at-organizationName=Amzaing Company USA, Inc.,id-at-localityName=Bellevue,id-at-stateOrProvinceName=Washington,id-
Certificate: 208204d0309203b8a003020102020a642cad1100000000006e300d05092a863886f60d01… ([pkcs-9-at-emailAddress=some.guy@ourcompany.com](mailto:pkcs-9-at-emailAddress=some.guy@ourcompany.com),[id-at-commonName=ourcompany.com](http://id-at-commonname%3Dourcompany.com/),id-at-organizationalUnitName=Our Company,id-at-organizationName=Our Company,id-at-localityName=
```

아까 확인해 봤던 것과 다르다. 우선 3개가 아니라 2개인 것부터 이상하다. 첫 번째는 Amzaing Company USA, Inc.의 그것이 맞는데 두 번째는 Entrust, Inc.의 것이 아닌 Our Company - 우리 회사의 이름이 떡하니 박혀 있었다.

### 5

자, 정리하면 이렇다.

회사에서는 사람들이 SSL을 통하여 사내 기밀을 유출하는지 살펴볼 필요가 있다. 그래서 사내망과 사외망 사이에 프록시를 하나 두고 주고받는 패킷을 검사한다.

그런데 클라이언트와 서버가 SSL로 통신하는 경우, 클라이언트에서 서버의 공개키로 세션 키를 암호화해 서버로 보낸다. 이는 서버의 비밀키로만 복호화가 가능하므로 프록시는 세션 키를 알 수가 없다. 내용을 훔쳐볼래도 볼 수가 없는 것이다.

그래서 프록시는 중간에서 자신이 대신 서버와의 handshaking에 참여하고, 클라이언트와는 자신의 사설 인증서를 사용해 handshaking을 한다. 이 과정에서 클라이언트가 받아보는 certificate가 위처럼 바뀌게 된다. 클라이언트에는 Our Company의 사설 인증서에 대한 정보가 있을 리 없으니 handshaking에 실패한다.

그럼 이슈가 때에 따라 나왔다 안 나왔다 하는 이유는 뭐였을까? 아마 프록시에서 몇몇 패킷을 샘플링해서 이런 짓을 하는 것 같다고 추측만 할 뿐이다.

분석 결과와 함께 테스트에 되도록 사내망을 사용하지 말라는 코멘트 후 이슈를 닫았다. 원인은 알아냈지만 썩 유쾌한 기분은 아니다.
