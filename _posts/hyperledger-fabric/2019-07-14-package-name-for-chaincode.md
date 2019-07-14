---
layout: post
title: Package name for Chaincode
tags: [hyperledger-fabric, Chaincode, go package, troubleshooting]
author: Jae
comments: true
---


# Package name for Chaincode

chaincode instantiate 시 아래와 같은 Error 가 발생하는 원인과 해결책을 기록한다.

```bash
starting container process caused "exec: \"chaincode\": executable file not found in $PATH": unknown
```

Error 가 발생한 환경은 다음과 같다.

* Hyperledger-Fabric : release-1.4
* Chaincode : golang

## 기

개인적으로 Fabric을 테스트하고 공부하기 위한 [fabric-playground](https://github.com/rtfmplz/fabric-playground)에 [chaincode_example02](https://github.com/rtfmplz/fabric-playground/tree/master/chaincode/chaincode_example02/go)의 Unittest를 추가하고 테스트가 잘 되는것을 확인 한 후, commit 전에 확인을 위해서  `chaincode_example02`를 다시 설치하고 instantiate 하는 과정에서 아래와 같은 Error 가 발생했다.

```bash
2019-07-14 15:41:57.184 UTC [endorser] callChaincode -> INFO 04e [ch1][a9a32894] Entry chaincode: name:"lscc" 
2019-07-14 15:41:58.262 UTC [dockercontroller] Start -> ERRO 04f start-could not start container: API error (400): OCI runtime create failed: container_linux.go:344: starting container process caused "exec: \"chaincode\": executable file not found in $PATH": unknown
2019-07-14 15:41:58.275 UTC [endorser] callChaincode -> INFO 050 [ch1][a9a32894] Exit chaincode: name:"lscc"  (1090ms)
2019-07-14 15:41:58.275 UTC [endorser] SimulateProposal -> ERRO 051 [ch1][a9a32894] failed to invoke chaincode name:"lscc" , error: API error (400): OCI runtime create failed: container_linux.go:344: starting container process caused "exec: \"chaincode\": executable file not found in $PATH": unknown
error starting container
error starting container
```

## 승

Error 내용을 보면, Container를 시작하는 과정에서 `exec: \"chaincode\": executable file not found in $PATH` Error 가 발생했다는 것을 알 수 있다.

Error 내용만을 가지고는 원인을 정확히 파악하기가 어려워서 Unittest를 추가한 과정을 다시 살펴보기로 했다. 다음과 같다.

[fabric repository](https://github.com/hyperledger/fabric)에서 example02의 [unittest](https://github.com/hyperledger/fabric/blob/release-1.4/examples/chaincode/go/example02/chaincode_test.go)를 `chaincode_example02.go`와 같은 폴더에 복사 하면서 package 명이 서로 달라 아래와 같은 Error와 함께 unittest에 실패 했다.

```bash
can't load package: package .: found packages main (chaincode_example02.go) and example02 (chaincode_example02_test.go) in /Users/jae/git/fabric-playground/chaincode/chaincode_example02/go
```

별 생각없이 `chaincode_example.go`의 `package`를 `chaincode_example02_test.go`의 package 명인 `example02`로 바꾸었고, Unittest는 성공할 수 있었다.

## 전

Unittest는 성공했지만, `chaincode_example02.go`를 instantiate 하는 과정에서 처음 언급한 Error 가 발생한 것이다. 원인은 다음과 같다. 

Excutable Program을 만들기 위해서는 main package 사용해야 한다는 golang의 [규칙](https://golang.org/doc/code.html#PackageNames) 때문에 `chaincode_example02.go`가 ccenv에서 빌드 될 때 정상적으로 Excutable 파일을 만들어내지 못했고 instantiate 과정에서 Excutable 하게 build 된 Chaincode 파일을 찾을 수 없다는 error가 발생한 것이다.

> main package는 go compiler에게 package가 library가 아닌 excutable program으로서 compile 되어야 함을 알려주는 역할을 한다.
>
> * https://thenewstack.io/understanding-golang-packages/

## 결

Fabric Network에 배포하기 위한 chaincode는 build 되어 executable 해야 하기 때문에 main package에 속해야 하고, 아래와 같은 main 함수를 가져야 한다.

```go
package main

// ...
 
func main() {
	err := shim.Start(new(SimpleChaincode))
	if err != nil {
		fmt.Printf("Error starting Simple chaincode: %s", err)
	}
}
```
