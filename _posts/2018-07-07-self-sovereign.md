---
layout: default
title: Self-Sovereign
category: post
author: Jae
comments: true
---

# Self-Sovereign

> 본 포스트는 개인적인 공부 목적으로 아래 white-paper를 번역하고 의견을 추가한 것입니다.
> 
> * https://sovrin.org/wp-content/uploads/2018/03/Sovrin-Protocol-and-Token-White-Paper.pdf


## Glossary
* Digital Credential = Verifiable Claims
    * Banking account
    * Education qualification
    * Healthcare data
* Owner
    * 내가 내가 맞다고 증명하고 싶은 사람 또는 단체
    * Issuer로부터 발행된 Digital Credential을 소유하고 있다.
* Issuer
    * Owner의 Claim에 자신 의 Private key로 만든 서명을 붙여 Digital Credential(Verifiable Claim, Signed Claim)을 발행해주는 사람 또는 단체
    * Private-Public key를 발행한 사람
* Verifier
    * Issuer의 Public key를 이용해서 Owner의 Digital Credential을 검증하고 싶은 사람 또는 단체


## The Problem

### Page.4 인터넷에서는 자신을 증명하는것이 어렵다.

* 확인 할 수 없으니, 신뢰할 수 없다.

![On the Internet, nobody knows you're a dog](/images/posts/self-sovereign/dog.png)

* Offline 에서는 자신을 증명하기 위해서 지갑에 있는 물리적인 자격증을 사용한다.
    * 물리적인 증명서는 사람이 보고 판단하니까 상대적으로 검증하는 것이 쉽다.
*  Online 에서는 왜 그렇게 못하는거지?
    * 표준이 없다.


### Page.5 디지털 증명서(Digital Credential)을 검증하기 위한 두가지 방법

* Physical document와 다르기 Digital Document는 위변조가 가능하다.
* 디지털 증명서를 검증하기 위해서는 두가지 문제를 해결해야 한다.
    * First. 기계가 읽을 수 있도록 표준화된 포맷이 필요하다.
        * 여권은 이미 표준화 되어 기계가 읽을 수 있다.
    * Second. Digital Credential가 위변조 되지 않았음을 증명 할 표준화된 방법이 필요하다.
        * 전 세계적으로 디지털 서명은 이미 합법적으로 유효하다.
            * 공개키 암호화 방식에 사용되는 두 개의 키를 사용해서 그것을 구현할 수 있다.
                * Private key를 Signing key로
                * Public key를 Verification key로
        * 그런데 이 방법에는 한가지 보안 홀이 존재한다.
            * Sign을 검증한 Verification key가 Issuer의 Public key라는 것을 어떻게 믿을 수 있지?


### Page.6 첫 번째 문제에 대한 기존 해법: Digital Credential의 표준화

* 현재 Claims (Personal Data) 들은 3rd party에 의해서 verify 되고 이것들은 web 에서 표현(사용)하기 어렵다.
    * 모두 제각각
* W3C의 Verifiable Claims Working Group 표준화를 진행중
    * Claim들을 표현하고, 교환하고, 검증하기 쉽고 더 안전하도록
* 표준화는 진행하지만, 결국 Verifiable Claims(Digital Credential)을 신뢰할 수 있는지는 Verifier와 Issuer 사이의 Existing Trust Relationship에서 나온다.

![verifiable-claim](/images/posts/self-sovereign/verifiable-claim.png)

```
Issuer   -     Owner      - Verifier
 은행     -       나        - 상인
 CA      -    web site    - 나
```

### Page.7 두 번째 문제에 대한 기존 해법: PKI

* 다음으로 Credential issuer(Digital Credential을 발행한 기관)의 디지털 서명을 확인하는 방법을 표준화 해야한다.
    * 네트워크를 통해서 Digital Credential을 발급 받거나, 검증을 위한 Issuer의 Public key를 전달하게 되면 언제나 위변조의 위험이 존재한다.
* 이 문제를 해결하기 위해서 PKI(Public Key Infrastructure)를 이용할 수 있다.
    * Public key 암호화의 전제는 public key를 가질 수 있다면, 어떤 사람이든 digital signature를 검증 할 수 있다는 것이다.
    * Private key는 오직 하나의 Public key를 가진다. 그 역도 성립한다.
    * 문제는 내가 가지고 있는 Public key가 정말 Credential Issuer의 것인지를 믿을 수 있느냐는 것이다.
        * 은행이 서명 했다는데, 검증하기 위한 Public key가 정말 은행 것이 맞아?
        * CA가 서명 했다는데, 검증하기 위한 Public key가 정말 CA의 것이 맞아?
* PKI를 활용한 Https
    * Web site는 private key owner로 자신의 public key(claims)를 CA에게 주고(전용선을 통해서), CA는 자신의 Private key로 sign해서 public key certificate(verifiable clams)를 발행한다.
    * 그리고 이 Certificate(Verifiable claims, Digital Credential)를 증명하기 위한 CA(Issuer)의 Certificate(public key)는 browser 에 내장되어 있다.
    * 그럼 Verifier인 나는 Local에 있는 CA의 Certificate를 이용해서 전용선을 통해 발행된 Web site(Owner)의 Certificate를 검증할 수 있는것이다.
* PKI의 문제는 가격이 비싸고, 중앙 집중적이라는 것이다.
    * 가격
        * 평판 좋은 CA의 인증서는 비싸다.
        * Root CA의 Certificate는 브라우저 및 기타 소프트웨어에 내장되어있기 때문에, CA가 된다는 것은 돈을 많이 벌 수 있다는 것을 의미한다.
        * 이것이 디지털 인증서를 개인이 아닌 회사가 구매하는 이유이다.
        * 디지털 인증서는 대다수의 사람들이 다루기 너무 어렵다. (Self-sovereign 이 불가능)
    * 단일 실패 지점
        * 더 나쁜것은 중개인을 디지털 신뢰 인프라에 삽입하는 것이 하나의 취약점이라는 것이다.
        * CA가 디지털 인증에 실수를 하거나, 서비스가 중단되거나 보안이 지체되거나 가격이 고르면 전체 시스템이 다 허물어져 버린다.


## The Solution

### Page.9 CA는 쓰기 싫다. Blockchain으로 대체한다.

* Public Blockchain을 Decentralized Root CA로 사용할 수 있다. (누구도 소유하지 않지만, 누구나 사용할 수 있다.)
* 블록체인은 인간에 대한 신뢰를 수학에 대한 신뢰로 대체한다.
    * 각각의 트랜잭션은 원작자에 의해서 signed 되어져  있다.
    * 각각의 트랜잭션은 Digital hash로 연결되어 있다.
    * 검증된 트랜잭션은 합의 알고리즘에의해 모든 머신으로 복제된다.
* 블록체인은 Public key를 위한 Decentralized self-service registry(등기소)로 사용되는데 이상적이다.
    * 블록체인 안의 모든 트랜잭션은 Private key가 필요한 디지털 서명을 가지고 있다.
    * 따라서 연관된 Public key 혹은 다른 자신의 것 임을 증명하기 위한 암호화 키를 보관하기 위한 저장소로 사용할 수 있다.


### Page.10 DID(address), DID Document(Verifiable Claims Storage), DID Method(SmartContract)

* 모든 Public key는 address를 가질 수 있다.
    * 이 address는 DID(Decentralized Identifier)라고 부른다.
    * DID는 Identity 소유자의 Private key를 통해서 통제되고, 검증할 수 있다.
* DID는 등록기관이 필요하지 않은 전세계적으로 유일한 검증가능한 identifier이다.
    * DID는 DID를 위한 public key, 소유자가 자신의 것이라고 밝히고 싶은 다른 개인적인 자격들, network address등을 포함하는 DID Document와 함께 블록체인에 저장된다. 
    * 신원(Identity) 소유자는 private key로 DID Document를 컨트롤 한다.
    * DID는 공개 표준이므로 어떤 블록체인이든 해당 블록 체인에서 DID를 등록(wirte) 및 해석(read) 할 수 있는 방법을 정의하는 DID method(SmartContract)를 정의할 수 있다.
    * DID는 블록체인 안에서 전자 서명된 트랜잭션에 의해서 등록되어지기 때문에, DID를 등록, 추적, 관리하기 위한 어떤 중앙 기관도 필요하지 않다.
* DID는 self-sovereign identity를 가능한게 한다. 그것은 소유권을 주장하고 싶은 것, 사람, 조직을 위한 일생 휴대용 디지털 신원이다.
    * DID는 DIgital Identity를 바꿨다.
        * Identity owner들은 유일한 ID를 얻기 위해 더이상 external provider에게 의존하지 않아도 된다.
        * 적절한 블록체인의 경제성을 고려할 때 DID는 저렴하므로, 사람들의 개인 정보를 보호하는데 필요한 만큼 정보를 생성할 수 있다.
        * 인터넷에 접속된 모든 사람들은 Public key의 소유권을 증명함으로써, 그들의 Claim을 검증할 수 있게 한다.


### Page.11 DID가 구현된 블록체인 위에서 누구나 digital signed credential을 발행할 수 있고 누구나 검증할 수 있다.

* DID를 위한 Public 블록체인과 함께, 누구나 digitally signed credential을 발행 할 수 있고, 누구나 그것을 검증할 수 있다.
    * 모든 DID에는 관련된 public-private key가 있기 때문에, DID를 가진 누구든 검증 가능한 클레임 및 기타 문서를 디지털 방식으로 발행하고 서명할 수 있다.
    * Verifier가 Issuer의 DID를 가지고 있는 한, 블록체인에서 Issuer의 public key를 찾는 것과 그 클레임의 서명을 검증하는것은 간단한 일이다.
* Digital Credential의 Issuer와 verifier은 ID를 공유(연계) 하거나 같은 조직에 속할 필요가 없다.
    * Issuer와 Verifier는 같은 조직에나 신원 연합에 속하지 않아도, DID 스펙에 의해서 필요한 Public key를 Public 블록체인에서 찾을수 있다.

![pki-with-blockchain](/images/posts/self-sovereign/pki-with-blockchain.png)


> **여기까지 내용을 정리하면,**
>
> 1. 내 DID(Address)에 저장된 카드번호 (Digital Credential)은 내 것이다. (내 DID에 접근 할 수 있는 사람은 Private key를 가진 나 뿐이므로)
> 2. 내 DID Document에 저장된 카드번호(Digital Credential)는 현대카드가 서명, 발행한 것이다. (블록체인 상에서 내 DID로 직접 전송할 수 있다.)
> 3. 서명을 검증하기 위한 위변조 되지 않은 현대카드의 Public key를 어떻게 구할 수 있지?
> 4. 과거에는 PKI로 그것을 보장했다. (Browser나 software에 내장하는 방식으로)
> 5. 현대카드의 DID에 있는 들어있는 Public key는 현대카드 것이다. (현대카드의 DID에 접근 할 수 있는 사람은 Private key를 가진 현대카드 뿐이므로)
> 6. 현대카드의 DID를 알고 있다면 블록체인에서 현대카드의 Public key는 항상 얻을 수 있다.
> 7. 현대카드의 Public key로 내 카드번호(Digital Credential)를 검증 할 수 있으므로 내 카드번호는 유효하다.
> 8. 내 카드번호는 현대카드의 public key로 검증되었고, 내 카드번호는 내 DID 안에 있다.
> 9. 따라서 내 카드번호는 내 것이다.

**To be Continue..**
