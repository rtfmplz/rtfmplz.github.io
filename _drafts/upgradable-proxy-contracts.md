# The Basics of Upgradable Proxy Contracts in Ethereum

Smart Contract가 Ethereum 블록 체인에 배포되면 변경이 불가능하므로 업그레이드 할 수 없다. 코드를 다른 계약으로 재구성함으로써 스토리지를 동일하게 유지하면서 로직을 업그레이드 할 수 있다. 사실, 업그레이드 가능한 지능 계약이 인기를 얻고 있으며 Jack Tanner는 사용 된 모든 기술을 설명하는 [좋은 기사](https://blog.indorse.io/ethereum-upgradeable-smart-contract-strategies-456350d0557c)를 갖고있었다. 그러나 그렇게 말하면서 나는 계약을 전혀 업그레이드하지 말아야한다고 생각한다. 좋은 예는 ICO를 시작할 때 토큰 로직을 업그레이드 할 수 있다는 것에 동의하지 않는다. 소유주가 약속을 지킬 수 있기 때문이다.

upgradability 토론을 떠나서이 기사에서 nick johnson이 처음으로 만든 프록시 기술에 중점을 둘 것이다.

아이디어는 1 개의 스토리지 계약, 1 개의 레지스트리 계약 및 1 개의 로직 계약을 갖는 것이다. 로직 계약에 새 기능을 추가하거나 기존 기능을 업그레이드 할 필요가있을 때마다 현재 로직 계약을 상속 받아 새로운 로직 계약을 생성한다.



`Storage` 계약은 단순히 상태를 보유하고 가능한 한 간단하게 만듭니다.

```java
pragma solidity ^0.4.21;

contract Storage {
    uint public val;
}
```

`Registry` 계약은 논리 계약에 대한 프록시를 제공하여 상속 된 스토리지 계약의 상태를 수정합니다.

```java
pragma solidity ^0.4.21;

import './Ownable.sol';
import './Storage.sol';

contract Registry is Storage, Ownable {

    address public logic_contract;

    function setLogicContract(address _c) public onlyOwner returns (bool success){
        logic_contract = _c;
        return true;
    }

    function () payable public {
        address target = logic_contract;
        assembly {
            let ptr := mload(0x40)
            calldatacopy(ptr, 0, calldatasize)
            let result := delegatecall(gas, target, ptr, calldatasize, 0, 0)
            let size := returndatasize
            returndatacopy(ptr, 0, size)
            switch result
            case 0 { revert(ptr, size) }
            case 1 { return(ptr, size) }
        }
    }
}
```

레지스트리 계약서에는 어떤 논리 계약서를 사용해야하는지에 대한 지식이 있어야합니다. setLogicContract 함수를 사용하여이를 설정할 수 있습니다. 우리는 단순한 Ownable.sol을 사용하여 admin 만 setLogicContract 함수를 호출 할 수 있도록했습니다. 어셈블리의 대체 기능은 일부 사용자에게는 익숙하지 않을 수 있지만이 특정 코드는 실제로 프록시 계약의 표준입니다. 기본적으로 내부 계약을 외부 계약으로 변경할 수 있습니다. Ownable 계약 이전에 Storage를 초기화하는 것도 매우 중요합니다. 이 시퀀스를 잘못 얻는 것은 재앙입니다. 왜 그런가?

델리게이트 콜 어셈블리 코드는 편리하지만 위험합니다. 그러니 그걸 가지고 살아 가기 전에 어떤 일이 일어나는지 알도록하십시오.

다음으로 논리 부분에 대해 이야기 해 봅시다.


```java
pragma solidity ^0.4.21;

import './Storage.sol';

contract LogicOne is Storage {
    function setVal(uint _val) public returns (bool success) {
        val = 2 * _val;
        return true;
    }
}
```

위의 코드는 간단합니다. LogicOne 계약은 "val"스토리지를 수정하기위한 계약입니다.


## Implementation

1. Registry.sol과 LogicOne.sol을 모두 배포합니다.

2. LogicOne이 배포 한 주소를 Registry.sol에 등록합니다. 즉,


```
Registry.at(Registry.address).setLogicContract(LogicOne.address)
```

3. 우리는 LogicOne ABI를 사용하여 레지스트리 계약에서 "val"스토리지를 수정합니다.

```
LogicOne.at(Registry.address).setVal(2)
```

4. LogicOne을 LogicTwo로 업그레이드 할 준비가되면 LogicTwo 계약을 전개하고이를 가리 키도록 레지스트리 계약을 업데이트합니다.

```
Registry.at(Registry.address).setLogicContract(LogicTwo.address)
```

5. 이제 LogicTwo로 레지스트리의 스토리지를 제어 할 수 있습니다.

```
LogicTwo.at(Registry.address).setVal(2)
```

당장 코드에 뛰어 들기보다는 우리가하고있는 일을 소화하는 데 약간의 시간이 걸리는 것이 중요합니다. LogicOne 및 LogicTwo의 저장소는 레지스트리 계약의 저장소와 동일하지 않습니다. 그것들은 스토리지 설계에 의해서만 관련이 있습니다 (이것은 매우 중요합니다).

## Conclusion

이 기사에 사용 된 코드는 github에서 찾을 수 있으며 주로 개념적 설명 목적으로 만 사용됩니다. 나는 그것의 맨손으로 해골을 만들기 위해 많은 복잡성과 수표를 줄였습니다. 프로덕션을 위해 사용하려는 경우, 귀하의 목적을 위해 더 많은 실사를 수행 할 것을 제안합니다.