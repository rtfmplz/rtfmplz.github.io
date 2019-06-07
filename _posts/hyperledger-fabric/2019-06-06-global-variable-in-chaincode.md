---
layout: post
title: Global Variable in Chaincode
tags: [hyperledger-fabric, chaincode, golang]
author: Jae
comments: true
---


# Global Variable in Chaincode

## Prologue

Supply Chain Management를 구현한 Chaincode 를 분석 할 일이 있었다. 

주문에 대한 정보는 원장에 기록하고, 생산자, 운송업자, 소매상이 하나의 주문에 대해 각각의 역할을 수행하면서 주문의 상태를 업데이트 하도록 작성된 Chaincode 였다. 

이 Chaincode는 최초에 들어온 주문의 수량을 GoLang의 Global Variable에 저장하고, Supply Chain의 마지막에 소매상이 최초에 요청한 주문의 수량과 자신이 받은 물건의 수를 비교해서 최종적으로 주문을 완료 또는 취소 하도록 구현되어 있었다. 

해당 Chaincode를 분석하면서, Chaincode에서 Global Variable이 어떻게 동작하는지 확실하게 알고 넘어가고자 간단한 예제를 만들어 봤다.

## Example Chaincode

아래 Chaincode는 Instantiate 시 `Count` 라는 Global Variable을 0으로 초기화 한 후, `count()` 함수를 호출하면 `Count` 변수를 1 씩 증가시키고 현재 값을 반환한다.

```go
import (
	"fmt"
	"strconv"

	"github.com/hyperledger/fabric/core/chaincode/shim"
	pb "github.com/hyperledger/fabric/protos/peer"
)

// SimpleChaincode example simple Chaincode implementation
type SimpleChaincode struct {
}

var Count int

func (t *SimpleChaincode) Init(stub shim.ChaincodeStubInterface) pb.Response {
	Count = 0
	fmt.Println("test_global_variable Init")

	return shim.Success(nil)
}

func (t *SimpleChaincode) Invoke(stub shim.ChaincodeStubInterface) pb.Response {
	fmt.Println("test_global_variable Invoke")
	function, _ := stub.GetFunctionAndParameters()
	if function == "count" {
		return t.count(stub)
	}

	return shim.Error("Invalid invoke function name. Expecting \"invoke\"")
}

func (t *SimpleChaincode) count(stub shim.ChaincodeStubInterface) pb.Response {
	Count = Count + 1
	CountAsByte := []byte(strconv.Itoa(Count))

	return shim.Success(CountAsByte)
}

func main() {
	err := shim.Start(new(SimpleChaincode))
	if err != nil {
		fmt.Printf("Error starting Simple chaincode: %s", err)
	}
}
```

## Test Scenario

1. chaincode를 peer1에만 설치한 후, `count()` 함수를 호출해서 `Count` 값이 증가하는 것을 확인한다.

```bash
CORE_PEER_ADDRESS=peer1.org1:7051 peer chaincode install -n count -v 1.0 -p github.com/chaincode/test/global_variable/
CORE_PEER_ADDRESS=peer1.org1:7051 peer chaincode instantiate -o orderer1.ordererorg:7050 -C ch1 -n count -v 1.0 -c '{"Args":["init", ""]}' -P "OR ('Org1MSP.member')"
CORE_PEER_ADDRESS=peer1.org1:7051 peer chaincode query -C ch1 -n count -c '{"Args":["count", ""]}'
```

2. chaincode를 peer2에 설치한 후, `Count` 값을 확인한다.

```bash
CORE_PEER_ADDRESS=peer2.org1:7051 peer chaincode install -n count -v 1.0 -p github.com/chaincode/test/global_variable/
CORE_PEER_ADDRESS=peer2.org1:7051 peer chaincode query -C ch1 -n count -c '{"Args":["count", ""]}'
```

## 결론

결론부터 이야기하면 `Count`는 원장(ledger)에 포함되지 않기 때문에, peer1의 `Count` 값과, peer2의 `Count` 값은 서로 공유되지 않는다.

> Chaincode에서는 원장(ledger)에 기록을 위해서 write-set을 transaction을 통해제출할 때, `ChaincodeStubInterface.PutState()`를 사용한다.

Chaincode는 원장에 접근하기 위한 interface를 구현한 프로그램으로, 독자적인 Docker Container 안에서 수행된다.

Chaincode Docker Container는 각각의 Peer에 Chaincode Install 후, 함수 최초 호출 시 생성된다. 위의 Test Scenario 기준, peer1의 Container는 `instantiate`, peer2의 Container는 `query` 시점에 생성된다. 따라서 각각의 Peer는 Chaincode 실행을 위한 개별 Docker Container를 사용므로 그 안에서 동작하는 프로그램 또한 각각이 독립적인 프로세스가 된다. 따라서 각각의 `Count`값이 다르다.


