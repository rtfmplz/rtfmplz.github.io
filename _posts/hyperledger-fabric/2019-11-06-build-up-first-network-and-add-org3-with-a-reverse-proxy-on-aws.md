---
layout: post
title: Build up first-network and add-org3 w/ a reverse proxy on AWS
tags: [hyperledger-fabric, AWS, Terraform, troubleshooting]
author: Jae
comments: true
---

# Build up first-network and add-org3 w/ a reverse proxy on AWS

> * 본 포스트의 내용은 개인의 공부 및 정리를 목적으로 작성한 것 입니다. 혹시 사실과 다르거나 잘못된 내용이 있으면 댓글로 남겨주시면 수정하겠습니다.

hyperledger-fabric 공식 기본 예제인 `first-network`을 aws상에 구축하고, `add-org3`를 수행한 후, `chaincode_example02`를 설치, 데이터가 정상적으로 복제되는 것을 확인하기 까지의 여정을 소개한다.

AWS 서비스를 이용해서 구축한 아키텍처는 아래에서 확인 할 수 있으며, 관련 Terraform 코드 및 docker-compose.yaml, shell-scripts는 [여기](https://github.com/rtfmplz/fabric-playground/tree/master/aws)에서 확인 할 수 있다.  

`first-network`, `add-org3` 에 대한 설명 및 관련 코드는 [hyperledger-fabric readthedocs](https://hyperledger-fabric.readthedocs.io/en/release-1.4/) 및 fabirc-samples의 [first-network](https://github.com/hyperledger/fabric-samples/tree/release-1.4/first-network) 코드를 참조하면 된다.

## Network Architecture

![network-architecture](/images/posts/hyperledger-fabric-aws/network-arch.png)

위의 아키텍처는 AWS 서비스를 이용해서 구축한 infra 위에서 Hyperledger-fabric 관련 서비스와 reverse proxy 등이 서비스 되고 있는 상태를 도식화 하고 있으며, 가능하면 AWS 서비스와 Fabric의 서비스명을 그대로 사용하였다. 화살표는 주요 통신 흐름을 나타낸다.  

아키텍처는 다음과 같은 주요 특징을 가진다. 이번 포스팅에서는 아래 주요 특징들을 설계, 구현하면서 경험한 것들을 간략히 공유해 보려한다.  

* Terraform 을 이용한 infra 및 서비스 자동 배포
* public-subnet과 private-subnet을 나누고, reverse proxy를 이용하여 보안성을 강화
* Bootnode를 통한 On-Premise의 Legacy 서비스로의 연계 및 public-subnet의 Fabric 모니터링

---
이하 업데이트 예정

## Terraform 을 이용한 infra 및 서비스 자동 배포

## public-subnet과 private-subnet을 나누고, reverse proxy를 이용하여 보안성을 강화

## Bootnode를 통한 On-Premise의 Legacy 서비스로의 연계 및 public-subnet의 Fabric 모니터링