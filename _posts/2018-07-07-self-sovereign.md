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

### Page.4 디지털 신원은 인터넷에서 가장 오래되고 힘든 문제 중 하나입니다.
* 확인 할 수 없으니, 신뢰할 수 없다.

![On the Internet, nobody knows you're a dog](/images/posts/self-sovereign/dog.png)

* Offline 에서는 자신을 증명하기 위해서 지갑에 있는 물리적인 자격증을 사용한다.
    * 물리적인 증명서는 사람이 보고 판단하니까 상대적으로 검증하는 것이 쉽다.
*  Online 에서는 왜 그렇게 못하는거지?
    * 표준이 없다.


### Page.5 문제의 핵심은 digital credentials을 확인하는 표준 방법이 없다는 것입니다.
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


### Page.6 W3C(World Wide Web Consortium)는 마침내 digital credentials을 표준화하고 있습니다.
* 현재 Claims (Personal Data) 들은 3rd party에 의해서 verify 되고 이것들은 web 에서 표현(사용)하기 어렵다.
    * 모두 제각각
* W3C의 Verifiable Claims Working Group 표준화를 진행중
    * Claim들을 표현하고, 교환하고, 검증하기 쉽고 더 안전하도록
* 표준화는 진행하지만, 결국 Verifiable Claims(Digital Credential)을 신뢰할 수 있는지는 Verifier와 Issuer 사이의 Existing Trust Relationship에서 나온다.

![verifiable-claim](/images/posts/self-sovereign/verifiable-claim.png)

```scala
Issuer   -     Owner      - Verifier
 은행     -       나        - 상인
 CA      -    web site    - 나
```

### Page.7 그러나 이것은 두 번째 문제를 남깁니다: credential 발급자의 디지털 서명을 검증하는 방법을 표준화하는 것
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

### Page.9 블록 체인 기술로 마침내이 문제를 해결할 수 있습니다.
* Public Blockchain을 Decentralized Root CA로 사용할 수 있다. (누구도 소유하지 않지만, 누구나 사용할 수 있다.)
* 블록체인은 인간에 대한 신뢰를 수학에 대한 신뢰로 대체한다.
    * 각각의 트랜잭션은 원작자에 의해서 signed 되어져  있다.
    * 각각의 트랜잭션은 Digital hash로 연결되어 있다.
    * 검증된 트랜잭션은 합의 알고리즘에의해 모든 머신으로 복제된다.
* 블록체인은 Public key를 위한 Decentralized self-service registry(등기소)로 사용되는데 이상적이다.
    * 블록체인 안의 모든 트랜잭션은 Private key가 필요한 디지털 서명을 가지고 있다.
    * 따라서 연관된 Public key 혹은 다른 자신의 것 임을 증명하기 위한 암호화 키를 보관하기 위한 저장소로 사용할 수 있다.


### Page.10 사실, 블록 체인을 사용하면 모든 공개 키가 이제 자신의 주소를 가질 수 있습니다
* 모든 Public key는 address를 가질 수 있다.
    * 이 address는 DID(Decentralized Identifier)라고 부른다.
    * DID는 Identity 소유자의 Private key를 통해서 통제되고, 검증할 수 있다.
* DID는 등록기관이 필요하지 않은 전세계적으로 유일한 검증가능한 identifier이다.
    * DID(address)는 DID를 위한 public key, 소유자가 자신의 것이라고 밝히고 싶은 다른 개인적인 자격들, network address등을 포함하는 DID Document(Verifiable Claims Storage)와 함께 블록체인에 저장된다. 
    * 신원(Identity) 소유자는 private key로 DID Document를 컨트롤 한다.
    * DID는 공개 표준이므로 어떤 블록체인이든 해당 블록 체인에서 DID Document(Verifiable Claims Storage)를 등록(wirte) 및 해석(read) 할 수 있는 방법을 정의하는 DID method(SmartContract)를 정의할 수 있다.
    * DID는 블록체인 안에서 전자 서명된 트랜잭션에 의해서 등록되어지기 때문에, DID를 등록, 추적, 관리하기 위한 어떤 중앙 기관도 필요하지 않다.
* DID는 self-sovereign identity를 가능한게 한다. 그것은 소유권을 주장하고 싶은 것, 사람, 조직을 위한 일생 휴대용 디지털 신원이다.
    * DID는 DIgital Identity를 바꿨다.
        * Identity owner들은 유일한 ID를 얻기 위해 더이상 external provider에게 의존하지 않아도 된다.
        * 적절한 블록체인의 경제성을 고려할 때 DID는 저렴하므로, 사람들의 개인 정보를 보호하는데 필요한 만큼 정보를 생성할 수 있다.
        * 인터넷에 접속된 모든 사람들은 Public key의 소유권을 증명함으로써, 그들의 Claim을 검증할 수 있게 한다.


### Page.11 DID에 대한 공용 블록 체인을 사용하면 누구나 디지털 서명 된 자격증 명을 발급 할 수 있으며 누구나 이를 확인 할 수 있습니다
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

## The Token: Sovrin token을 이용하면 Digital Credential을 누구나 발행하고, 안전하게 교환할 수 있는 Market place를 구현할 수 있다.

### Page.31 경제적 이익을 실현하기 위해서는 Digital Credential의 가치를 교환하는 새로운 방법이 필요하다.

* Verifiable Claim의 교환은 Owner 및 Verifier에게 실질적인 경제적 이득을 준다.
    * Verifier: 위험 감소시켜 비즈니스 수행 비용을 낮춘다.
    * Owner: Verifier와의 마찰을 제거하고 Verifier가 제공하는 제품 가격을 낮출 수 있다. (e.g., “Good Driver discount”)
    * 즉, Credential Owner와 Verifier는 Credential을 발급한 Issuer로부터 실질적인 이득을 얻는다.

![value-exchange](/images/posts/self-sovereign/value-exchange.png)

* Verifier가 취하는 위험이 클수록 verifiable claims의 가치는 커진다.
    * 낮은 수준(low-end)에서는 웹 사이트에 로그인 한 대상이 봇 (bot)이 아닌지 확인하는데 도움을 줌으로서 위험을 낮춘다.
    * 높은 수준(high-end)에서는 신용 보고서 또는 백그라운드 체크가 수천 달러의 위험을 줄일 수 있다.
    * 간단히 말해 모든 verifiable claims의 교환은 Owner 및 Verifier에게 일정 수준의 이득을 준다.

### Page.32 현재, verifiable claim을 구매하기위한 유일한 방법은 전통적인 지불 시스템이다.

* 이 구식 방법에는 세 가지 문제가 있다.
* 첫째, verifiable claim을 거래하는 시장은 높은 가치의 자격 증명 만을 거래하도록 제한된다.
    * Verifier가 Credential 구매를 위해 Issuer 각각에 대한 거래 계좌를 만들어야 한다면, 고 위험 산업을 위한 중요한 자격 증명을 거래하기 위한 시장만이 존재할 것이다.
        * 개인이 대형 Issuer 로부터 직접 Digital Credential을 발급 구매하는게 어렵다는 의미인듯.
        * 신용 보고서, 은행 업무 등? (왜...?, 거래 계좌 만드는게 어려운가? 비싼가? 그래서 기업만 하나?)
    * 그럼, 주소 확인, 소셜 네트워크 회원 가입, 동료 추천 등 낮은 가치의 자격 증명을위한 long-tail market의 막대한 잠재력을 놓치게 된다.

> **Server side certificate**
> 
> 7 page에서 이야기 한 것 처럼, 보통의 경우 인증서를 구매하는 주체는 Naver 같은 대형 사업자이다. 개인이 인증서를 구매하기는 너무 비싸고, 절차도 어렵다. 은행을 이용할 때 발급받는 공인 인증서도 결국 은행에서 우리 은행을 이용해 주십사 발급받아 주는 것이다. 결국 개인이 공인 인증서, 즉, Digital Credential을 구입하는 것은 매우 어렵다.

* 둘째, 데이터 유출의 가장 큰 대상 인 대형 공급자를 선호한다. (e.g. Equifax)
    * 시장에서 마찰이 증가할수록 규모의 경제가 대형 공급자에게 이익을 준다. (??)
    * 결국, 대형 사업자들이 독점하게 되고 대형 사업자들은 공격목표가 되어 치명적인 데이터 유출이 발생할 수 있다.
        * Equifax, Yahoo
* 셋째, 개인 정보를 보호 할 수 있는 선택적 공개(selective disclosure)와 같은 강력하고 새로운 개인 정보 보호 기술의 사용을 막는다.  
	* 선택적 공개의 주체는 나인데, 기존 결재 시스템에서 나의 Digital Credential은 내 통제하에 있지 않다.

> **Long-tail**
> 
> * 파레토 법칙의 반대되는 개념으로 하위 80%의 상품이나 고객들이 상위 20%에 비해 더 큰 매출을 가져온다는 것이다. 이러한 현상은 주로 온라인 비즈니스 시장에서 자주 나타난다. 온라인에서는 장소, 시간적 제약이 없고 수 많은 콘텐츠가 제공되기 때문에 그 꼬리가 엄청나게 길어진다. 즉, 롱테일이 매우 큰 힘을 가지게 된다는 것이다.
> 
> e.g. 
> 
> * 음악 콘텐츠: 20%: Top 100, 80%: 7080
> * 1인 미디어: 20%: 공중파, 80%: 1인 미디어

### Page.33 해결책은 Sovrin 프로토콜의 기본 요소인 새로운 디지털 토큰이다.

* Sovrin 토큰은 디지털 자격 증명의 개인 정보 보호 가치 교환에 인센티브를 제공하여 세 가지 문제를 모두 해결한다.
    * Sovrin 토큰으로 Verifiable claims을 교환 함으로써 Sovrin 프로토콜은 신뢰를 거래하는 디지털 시장이 된다.
* 첫째, 모든 Digital Credential을 거래할 수 있다.
    * 기존 시장에서는 높은 가치의 Digital Credential 만이 교환 됐다면, 가치를 효율적으로 양도 할 수 있는 디지털 토큰 및 프로토콜을 통해 모든 종류의 자격 증명, 약한 평판 신호 (계정 연령, 커뮤니티 등급, 후원)에 대한 시장이 형성된다. 
    * 신뢰의 척도가 될 수있는 모든 것은 sovrin 토큰으로 교환 될 수 있다.
* 둘째, 모든 유형 및 규모의 credential issuers가 시장에 진입 할 수 있습니다.
    * 누구나 Sovrin network 안에서 credential issuers가 될 수 있다. 
    * 그들은 digital credential을 제공하고, sovrin token을 받을 것이다. (이 Issuer를 어떻게 믿을 수 있지? Insurer를 이용할 수 있다.)
    * 소액 융자, 임대 자격, 취업 자격, 온라인 추천, 뉴스 검증 등 
 * 셋째, 이 새로운 마켓 플레이스는 GDPR(General Data Protection Regulation) 준수 개인 정보 보호를 지원할 수 있다.
    * Owner의 동의없이 Credential을 공유하면 법적 규제와 소유자의 신뢰에 위배된다.
    * Verifiable claims 교환과 연결된 Sovrin 토큰을 사용하면, 공유 된 자격 증명의 가치를 실현하는 것이 안전하게 소유자의 동의하에서 수행 될 수 있다.

> GDPR ?
> 
> * 2018년 5월 25일부터 시행되는 EU(유럽연합)의 개인정보보호 법령이며, 법령 위반시 과징금 등 행정처분이 부과될 수 있어 EU와 거래하는 우리나라 기업도 이 법에 위반되지 않도록 주의할 필요가 있다.

### Page.34 Sovrin 토큰은 디지털 자격 증명을위한 글로벌 마켓 플레이스를 가능하게한다.

* 인터넷은 많은 시장을 평평하게 만들고 세계화 시켰다. 하지만, Digital credentials을 교환할 수 있는 시장은 아직 아니다.
    * 소매업. 경매. 분류. 소프트웨어. 이 모든 것들이 인터넷과 웹에 의해 다시 만들어져 중개자를 제거하고 도달 범위를 넓혔다.
    * 그러나 위에 설명 된 지불 및 개인 정보 보호의 어려움이 Digital Credentials 이 인터넷에서 쉽게 교환되는것을 막고 있다.
* SSI(Self Sovereign Identity)를 위한 글로벌 공익사업은 다양한 신뢰를 거래하는 시장을 개방하고 다양한 Issuer가 credential 품질과 비용을 경쟁하게 함으로써 선순환을 창출 할 수 있다.
    * Sovrin 토큰과 Sovrin 프로토콜을 사용하면 Verifiable Claim의 가치가 verifiers에서 issuer로 또는 verifiers에서 owner를 통해 issuer로 Claim이 교환 될 때마다 이동한다.
        * 첫 번째 사례는 Sovrin zero-knowledge payment protocol을 사용한다. 그러면, Issuer에게는 요청한 가격의 sovrin token이 지불될 뿐, credential을 누가, 어디서 사용 중인지는 알 수 없다.
        * 두 번째 경우에 Verifier는 Owner에게 직접 자격 증명을 얻기위해 요금을 지불 할 수 있으며, Owner 역시 자격 증명을 얻기 위해 Issuer에게 요금을 지불한다.

![value-exchange2](/images/posts/self-sovereign/value-exchange2.png)

> 정리
> 
> * Digital Credentials의 거래가 가능하게 됨으로써, Issuer들이 credential 품질과 가격을 놓고 경쟁하게 된다.
> * Verifier-Owner, Issuer-Owner간에 직접 Digital credential을 직접 거래할 수 있다.
> * zero-knowledge payment protocol에 의해서 Issuer에게는 요청한 가격의 sovrin token이 지불될 뿐, credential을 누가, 어디서 사용 중인지는 알 수 없다.

### Page.35 예를 들어, 이동 통신사를 통해 어느 시점에서든 자신의 위치를 증명하고 비용을 지불 할 수 있습니다.

* 최신 스마트 폰을 사용하면 이동 통신사가 스마트 폰의 현재 위치에 대한 verifiable claim을 생산 할 수 있다.
    * 스마트 폰에는 GPS 기능이 포함되어 있으므로 휴대 전화에서 내 위치에 대한 verifiable claim을 쉽게 공유 할 수 있다. 
    * 그러나 Issuer가 개인이면 verifiers은 그 Claim을 신뢰하지 않는다.
    * 모바일 서비스를 제공하기 위해서는 단말의 위치 정보가 필요하기 때문에 이동 통신사는 휴대 전화의 위치를 알고 있다.  
    * 그래서 이동통신사는 단말의 위치에 대한 Verifiable Claim을 발행 할 수 있으며, Verifier는 이동통신사를 신뢰할 수 있기 때문에 Owner의 위치 정보가 담긴 Verifiable Claim을 신뢰할 수 있다.
* 비공개로 Owner의 허락하에 만 공유되는 이 위치 정보는 verifiers에게 실질적인 가치가 있다.
    * Owner의 위치는 법으로 보호되는 민감한 개인 정보이다. 
    * 그러나 self-sovereign identity에 의해 이동 통신사로부터 sign 된 나의 위치에 대한 Verifiable Claim을 내가 선택한 Verifiers와 개인적으로 공유 할 수있다.
    * 앱을 통해 쉽게 위조 될 수 있는 GPS 데이터와 달리 이 이동 통신사로부터 Sign된 Verifiable Claim은 매우 신뢰할 수 있다.
    * Verifier는 아래와 같이 Verifiable Claim을 이용 할 수 있다.
        * 귀중한 거래를 요청할 때 집이나 사무실에 있다는 것을 확인 할 수 있다면 은행은 위험이 감소한다.
        * 확인 된 집 또는 사무실 주소에서 새 컴퓨터를 주문하는 것을 확인 할 수 있다면 온라인 판매자의 위험은 감소한다.
        * 십대 청소년이 부모님의 현재 위치를 확인할 수 있다면? 음?????!!
* Owner는 편리함을 얻고, Verifier는 보증을 얻고, Issuer는 새로운 수익을 얻는다.
    * Verifiable Claim의 청구액 당 가치가 몇 센트에 불과하더라도 한 번의 클릭으로 위치 확인을 수행 할 수 있으므로 통신 사업자를위한 새로운 수익원을 창출하는 동시에 가입자에게 새로운 서비스를 제공 할 수 있습니다.

![mobile-carrier](/images/posts/self-sovereign/mobile-carrier.png)


> **정리: 위치 정보 공유 예**
> 
> * Owner: 자신의 위치 정보를 자신이 소유하고, 원하는 경우에만 공유할 수 있다.
> * Verifier: 통신사로부터 서명된(위변조 되지않은) 위치 정보를 얻음으로써 위험을 감소시킬 수 있다.
> * Issuer: Verifier(은행)로부터 서명에 대한 대가를 얻을 수 있다.

### Page.36 Sovrin 토큰은 디지털 신용 증명 보험 시장을 활성화 할 수 있다.

* 디지털 자격 증명은 위험을 관리하기위한 도구이다. 이는 보험과 매우 비슷하다.
    * 신원 거래는 항상 잘못 확인 될 위험을 가지고 있다.
    * 이것은 기술의 문제가 아니라 사람들의 문제이다. 
        * 사람들은 실수를 저지른다. 
        * 사람들은 가짜 신분증을 사용한다. 
        * 사람들은 기록을 섞는다.
* Sovrin 토큰으로 구매한 Digital credentials은 Sovrin 토큰으로 보험에 가입 할 수도 있다.
    * Issuer는 Insurer에게 보험을 직접 구매하여 그들이 발행하는 claim의 가격에 보험 가격을 포함시킬 수 있다.
    * Verifier는 Insurer가 보증하는 Issuer의 Claim에 대해 독립적으로 보험을 구입할 수 있다.
* Insurer는 credential issuer를 위한 사실상 평판좋은 공급자가 될 수있는 잠재력이 있기 때문에, 효율적인 시장을 창출한다.
    * 보험사는 사업이기 때문에 위험 평가에 매우 뛰어나다. 
    * 정확한 재무 인센티브가 있다. 
    * Issuer에 대한 Insurer의 평가는 다른 검증 기관이 참고 할 수 있다. 이것은 Issuer가 더 좋은 평판, 더 낮은 보험료를 위해서 일하도록 유도한다.

![with-insurer](/images/posts/self-sovereign/with-insurer.png)


> **정리: 신원 거래는 항상 잘못 확인 될 위험을 가지고 있다. (사람이 문제)**
>
> * Insurer를 통한 Digital Credential 모델을 생각할 수 있다.
> * Issuer가 Insurer의 보험 가격을 Credential 가격에 포함시킬수도 있고, 반대로 Verifier가 Insurer가 보장하는 Issuer의 Claim에 대한 보험을 구입할 수도 있다.
> * 결국, Insurer는 credential issuer를 위한 사실상 명성높은 공급자가 될 수 있다.
> * 개인적인 생각이지만, Issuer가 다양해지면 믿을수 없는 Issuer도 생길것이고, 그럼 이걸 보장해 줄 수 있는 기관으로 Insurer를 사용할 수 있겠다.
 
### Page.37 예를 들어, 대학은 졸업생들에게 검증 가능한 학위를 보장 할 수 있습니다.

* 2011년에 “학위 공장”이 가짜 졸업 증서로 매년 미화 3 억 달러 이상을 벌어 들이고 있다고 추정된다.
* 대학은 신뢰 할 수 있는 기관이기 때문에 진짜 학위를 위한 신원 보증 보험은 매우 저렴해야한다.
    * 대학은 광범위한 인증 절차를 거치므로 보험사가 실제 인증 절차를 쉽게 확인해야한다.
    * 대학은 발행한 학위의 진위성을 보장 할 강력한 동기를 가진다. 이는 교육 기관으로서의 명성의 핵심이다.
    * 결과적으로 디지털 신용 증명 보험의 비용은 매우 합리적이고 가치는 매우 높아야한다.
* 소수의 보험 회사가 전 세계의 대학에 보험 및 명성을 제공 할 수 있다.
    * 보험 회사가 대학을 위한 신용 보험을 판매하면, 전세계의 고용주(Verifier)들은 더 이상 모든 교육 기관의 진위 여부를 확인할 필요가 없습니다. 
    * 고용주는 보험사를 신뢰 할 수 있으며 보험사는 대학으로 Sovrin 토큰을 지급받는다. 
    * 검증가능한 학위는 보험사의 Verifiable Claim을 동반한다.
    * 보험사의 Verifiable Claim에 의해서 고용주는 그들이 필요로하는 확신을 얻는다. 
    * 결국, 졸업생들은 그들이 원하는 검증을받고, 대학은 평판을 얻고, 보험 회사는 수익을 얻는다.

![university-with-insurer](/images/posts/self-sovereign/university-with-insurer.png)

> **정리: 학위(Digital Credential) 거래에도 보험사를 낄 수있다.**
>
> * 대학의 서무처리하는 사람이 장난질을 칠 수 있다.
> * 가짜 학위에 대한 위험을 보험사가 보장한다.

### Page.38 또한 Sovrin 토큰은 고객 데이터를 위한 윤리적 시장을 열 수 있다.

* 오늘날 유출 된 개인 데이터는 당사자의 동의없이 third-party data brokers에 의해 집계되고 판매된다.
    * 대형 신용 보고 기관 (Equifax, Experian 및 TransUnion)은 매 월 45 억 개의 데이터를 수집하여 소비자의 직접적인 허가없이 신용 보고서를 작성하며, 2016 년 매출액 3 억 달러를 넘는다.
* Sovrin 토큰을 사용하면, 회사는 고객에게 데이터 공유의 대가로 인센티브를 제공할 수 있다.
    * 고객 정보 거래에서 중개인을 제거하면 다음과 같은 세 가지 주요 이점이 있다.
        * 데이터는 고객으로부터 직접 제공되기 때문에 fully permissioned 되어 있다.
        * 데이터는 타사 데이터 또는 추론된 데이터보다 더 최신이고, 가치가 있다.
        * Sovrin 토큰을 이용하여 데이터에 대한 보상을 제 3 자가 아닌 고객에게 직접 전달하면, 거래자들간의 신뢰, 충성심 등이 증가한다.

![direct-sharing](/images/posts/self-sovereign/direct-sharing.png)

> **정리: Direct sharing Digital Credential**
>
> * 대형 신용 보고 기관들이 데이터를 내 수집하고 가공한 후, 판매해서 돈을 벌고 있다.
> * Owner가 직접 데이터를 Verifier와 거래 함으로서 최신의 데이터를 판매하고 보상을 얻을 수 있다.

### Page.39 예를 들어, Sovrin network는 수십 억 달러의 주소 변경 문제를 해결 할 수 있다

* 인터넷을 통해 자동 주소 변경을 수행하는 간단한 방법은 아직 없다.
    * 인터넷 혁신을 감안할 때 주소 변경 (COA)을위한 간단한 방법이 없다니 놀랍다.
    * 미국에서만 매년 4 천만 명이 넘는 사람들이 이사를 하고, 주소 변경을 위한 평균 처리 비용은 6.00 달러로 연간 수십억 달러의 비용이 든다.
* 신뢰 할 수있는 발급자로부터 완전 허가 된 verifiable claim이 마침내 필요한 모든 요구 사항을 충족시킬 수 있다.
    * 기업이 인터넷을 통해 신뢰할 수 없는 주소 변경을 받아 들일 수 없었다.
    * 따라서 기업은 국가 우편 서비스와 같이 신뢰 할 수있는 발급 기관의 인증 된 COA를 필요로한다.
    * Self-sovereign identity과 verifiable claims infrastructure가 이 솔루션을 제공 할 수 있다.
* verifiable digital 주소 변경은 Verifier가 Issuer와 Owner 양쪽 모두에게 비용을 지불 할 만큼 많은 비용이 절감된다.
    * 이 비즈니스는 6.00 달러의 평균 처리 비용 외에도 배송 오류를 줄이고 사기 예방 할 수 있다.
    * COA 발급 기관에 대한 가려져 있기 때문에 Issuer는 Owner와의 관계를 알 수 없다.
    * ID 소유자에게는 Sovrin 토큰이 직접 지불되며, 이는 또한 충성도와 반복 비즈니스를 구축하는 데 도움이된다.

![postal-service](/images/posts/self-sovereign/postal-service.png)

> **정리: 주소 변경 시스템에 적용**
>
> * 이사를 하게되면 우리집으로 우편을 보내는 모든 사업자에게 공지를 해줘야한다.
> * 기업은 인터넷을 통해서 주소 이전을 받아들이기 어렵다. (신뢰하기 어렵기 때문에)
> * 국가 기관이 인증한 주소를 sovin network에서 판매, 교환하면 많은 비용이 절약될 수 있다.


## Conclusion

### Page.40 결론적으로, "신뢰의 토큰"의 궁극적인 가치는 우리 모두가 믿을 수있는 인터넷이다.

* Bitcoin과 Ethereum은 decentralization이 효과가 있음을 입증했다.
    * 그들은 글로벌 cryptocurrency가 돈의 본질을 바꿀수 있음을 증명했고, SmartContract가 정부, 비즈니스, 금융 및 법률의 본질을 바꿀 수 있음을 보여주었다.
* sovrin은 decentralization이 self-sovereign identity와 verifiable claims에 효과가 있음을 증명할 수 있다.
    * Sovrin이 기반을 둔 공개 표준 (DID 및 Verifiable Claim)은 공개 블록 체인의 고유한 암호화 속성을 활용할 수있는 세 번째 방법이다. 
    * 이번 목표는 인터넷상의 두 피어 간의 신뢰를 위해 상호 운용성이있는 디지털 자격 증명을 제공하는 것이다.
* Bitcoin 및 Ethereum 토큰과 마찬가지로 Sovrin 토큰은 이 새로운 네트워크의 본질적인 구성 요소이다.
    * Verifiable Claim의 교환을 위해 Sovrin 프로토콜과 동일한 개인 정보 보호 속성을 사용하여 가치 교환을 할 수 있도록 설계된 Sovrin 토큰은 Issuer, Owner 및 Verifier 에게 인센티브를 줌으로써 가치를 창출한다. 
* 가장 중요한 것은 Sovrin 인프라가 글로벌 경제에 영향을 주고 인간 사회를 연결하기 위해 의존하는 네트워크에 대한 신뢰를 회복하는 데 도움이 될 수 있다는 것이다.
