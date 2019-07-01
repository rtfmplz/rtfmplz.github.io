---
layout: post
title: Chaincode instantiate error w/ private docker registry
tags: [hyperledger-fabric, error, docker local registry, ccenv, rumtime]
author: Jae
comments: true
---

# Chaincode instantiate error w/ private docker registry

> 프라이빗 도커 이미지 저장소를 이용하는 방법은 [여기](https://www.44bits.io/ko/post/running-docker-registry-and-using-s3-storage) 참조

Hyperledger-Fabric 네트워크 구성시, 외부 인터넷 접근이 차단된 기업 환경에서 프라이빗 도커 이미지 저장소를 사용해야 하는 경우 Peer, Orderer 등의 image 경로는 docker-compose 파일의 image의 경로를 변경해서 간단하게 로컬 저장소를 바라보도록 할 수 있다.

하지만 이 경우 chaincode instantiate 시 다음과 같은 Error 를 만나게 된다. 본 Post에서는 아래 Error의 원인과 해결 방법을 알아본다.

```bash
# byfn.sh up in fabric-samples/first-network
Error: could not assemble transaction, err proposal response was not successful, error code 500, msg error starting container: error starting container: Failed to generate platform-specific docker build: Failed to pull hyperledger/fabirc-ccenv:latest: API error (500): Get https://registry-1.docker.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
!!!!!!!!!!!!!!!! Chaincode instantiation on peer0.org2 on channel 'mychannel' failed !!!!!!!!!!!!!!!!
```


# 기

위의 Error를 살펴보면 다음과 같은 내용을 확인할 수 있다.

```bash
Failed to pull hyperledger/fabric-ccenv:latest
```

이유는 Peer가 instantiate 과정에서 hyperledger/fabric-ccenv docker image를 사용해서 chaincode를 빌드하는데, local에 fabric-ccenv image가 없으면 '[https://registry-1.docker.io](https://registry-1.docker.io/)'에 접근해서 다운로드를 시도하기 때문이다.

# 승

Fabric network의 다른 service들이 운영자에 의해서 직접 실행되는 반면 ccenv는 chaincode 를 빌드하기 위한 환경으로 Peer에 의해서 실행된다.

따라서, Peer의 설정 파일(`core.yaml`) 에서 chaincode 빌드에 사용 할 ccenv와 runtime 환경으로 사용 할 docker image의 경로를 변경할 수 있다. (runtime 환경은 chaincode를 작성한 언어에 따라 다르게 설정할 수 있다.)

- core.yaml

```yaml
chaincode:
	# Generic builder environment, suitable for most chaincode types
	builder: $(DOCKER_NS)/fabric-ccenv:latest
	golang:
			# golang will never need more than baseos
			runtime: $(BASE_DOCKER_NS)/fabric-baseos:$(ARCH)-$(BASE_VERSION)
	
			# whether or not golang chaincode should be linked dynamically
			dynamicLink: false
	
		car:
			# car may need more facilities (JVM, etc) in the future as the catalog
			# of platforms are expanded.  For now, we can just use baseos
			runtime: $(BASE_DOCKER_NS)/fabric-baseos:$(ARCH)-$(BASE_VERSION)
	
		java:
			# This is an image based on java:openjdk-8 with addition compiler
			# tools added for java shim layer packaging.
			# This image is packed with shim layer libraries that are necessary
			# for Java chaincode runtime.
			runtime: $(DOCKER_NS)/fabric-javaenv:$(ARCH)-$(PROJECT_VERSION)
	
		node:
			# need node.js engine at runtime, currently available in baseimage
			# but not in baseos
			runtime: $(BASE_DOCKER_NS)/fabric-baseimage:$(ARCH)-$(BASE_VERSION)
```

# 전

다음으로, Docker-compose는 사용 할 이미지의 경로를 다음과 같이 local registry를 바라보거나, docker hub의 개인 registry를 바라보도록 지정할 수 있다.

```yaml
services:

	service1:
		image: local_registry:5000/yourimage

	service2:
		image: your_docker_hub_registry/yourimage
```

# 결

`core.yaml`을 직접 수정한 후, `volume` 을 이용해서 Peer container 구동시 바라보게 할 수도 있지만, 아래와 같이 `environment`를 활용할 수 있다. 단, local_registry에 해당 이미지를 미리 다운로드 해놓아야 한다 :)

```yaml
PRIVATE_DOCKER_REGISTRY='local_registry:5000'
CORE_CHAINCODE_BUILDER=${PRIVATE_DOCKER_REGISTRY}/fabric-ccenv:latest
CORE_CHAINCODE_GOLANG_RUNTIME=${PRIVATE_DOCKER_REGISTRY}/fabric-baseos:latest
```