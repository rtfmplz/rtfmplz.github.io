---
layout: post
title: Where is the ledger data in hyperledger-fabric
tags: [hyperledger-fabric, block, LedgerData]
author: Jae
comments: true
---

# Where is the ledger data in hyperledger-fabric

## 기

Hyperledger-Fabric은 Orderer에서 block을 생성해서 각 조직의 Leader Peer로 block을 전파한다.

따라서 Peer와 Orderer에 모두 LedgerData가 존재한다.

Peer와 Orderer는 Process이므로 FileSystem어딘가에 LedgerData를 File로 가지고 있다. 본 Post에서는 Peer와 Orderer가 어떤 경로에 어떤 내용으로 LedgerData를 보관하는지 알아보려 한다.

> Leader Peer는 조직 대표로서 Orderer로 부터 블록을 받아 조직의 Peer들에 공유하는 역할을 한다. Leader Peer는 조직 내의 피어들 중 선출할 수도 있고, 조직 내에 하나 이상의 Leader Peer를 설정할 수도 있다.

## 승

먼저 Peer와 Orderer가 Ledger Data를 저장하는 위치를 알아보자.

Peer와 Orderer는 각각 `core.yaml`, `orderer.yaml` 파일을 설정 파일로 사용한다.

각 설정 파일에서 LedgerData가 저장되는 위치를 설정할 수 있다.

### Peer

- core.yaml

```yaml
# Path on the file system where peer will store data (eg ledger). This
# location must be access control protected to prevent unintended
# modification that might corrupt the peer operations.
fileSystemPath: /var/hyperledger/production
```

### Orderer

- orderer.yaml

```yaml
################################################################################
#
#   SECTION: File Ledger
#
#   - This section applies to the configuration of the file or json ledgers.
#
################################################################################
FileLedger:

    # Location: The directory to store the blocks in.
    # NOTE: If this is unset, a new temporary location will be chosen every time
    # the orderer is restarted, using the prefix specified by Prefix.
    Location: /var/hyperledger/production/orderer

    # The prefix to use when generating a ledger directory in temporary space.
    # Otherwise, this value is ignored.
    Prefix: hyperledger-fabric-ordererledger
```

## 전

설정 파일에서 설정한 경로를 tree 명령을 이용해서 하위 경로를 print 해보면 `chains/${channel_name}` 경로에 `blockfile_000000` 파일이 존재하는 것을 확인 할 수 있다. `blockfile_000000` 파일은 이름만 보면 `genesis.block` 일 것 같지만, Blockchain LedgerData로, Block 추가 시 해당 파일에 Block의 내용이 append 된다.

### Peer

- tree /var/hyperledger/production

```bash
/var/hyperledger/production
|-- chaincodes
|   |-- mycc.1.0
|   |-- mycc.1.1
|   |-- mycc.1.2
|   |-- mycc.1.3
|   |-- pdc02.1.0
|   `-- pdc02.1.1
|-- ledgersData
|   |-- bookkeeper
|   |   |-- 000001.log
|   |   |-- CURRENT
|   |   |-- LOCK
|   |   |-- LOG
|   |   `-- MANIFEST-000000
|   |-- chains
|   |   |-- chains
|   |   |   `-- ch1
|   |   |       `-- blockfile_000000
|   |   `-- index
|   |       |-- 000001.log
|   |       |-- CURRENT
|   |       |-- LOCK
|   |       |-- LOG
|   |       `-- MANIFEST-000000
|   |-- configHistory
|   |   |-- 000001.log
|   |   |-- CURRENT
|   |   |-- LOCK
|   |   |-- LOG
|   |   `-- MANIFEST-000000
|   |-- historyLeveldb
|   |   |-- 000001.log
|   |   |-- CURRENT
|   |   |-- LOCK
|   |   |-- LOG
|   |   `-- MANIFEST-000000
|   |-- ledgerProvider
|   |   |-- 000001.log
|   |   |-- CURRENT
|   |   |-- LOCK
|   |   |-- LOG
|   |   `-- MANIFEST-000000
|   `-- pvtdataStore
|       |-- 000001.log
|       |-- CURRENT
|       |-- LOCK
|       |-- LOG
|       `-- MANIFEST-000000
`-- transientStore
    |-- 000001.log
    |-- CURRENT
    |-- LOCK
    |-- LOG
    `-- MANIFEST-000000
```

### Orderer

- tree /var/hyperledger/production

```bash
/var/hyperledger/production
`-- orderer
    |-- chains
    |   |-- ch1
    |   |   `-- blockfile_000000
    |   `-- testchainid
    |       `-- blockfile_000000
    `-- index
        |-- 000001.log
        |-- CURRENT
        |-- LOCK
        |-- LOG
        `-- MANIFEST-000000
```

## 결

`blockfile_000000` 은 protobuf 형식으로 encoding 되어 있다. 아래 명령으로 json 파일 형태로 decoding 할 수 있다. (20190630 기준, 버그가 있는 듯 하다.. [여기](https://jira.hyperledger.org/browse/FAB-15037?workflowName=FAB%3A+Bug+Workflow&stepId=2) 참조.., 아래 방법으로 genesis.block은 decoding이 가능하다.)

```bash
configtxlator proto_decode --type 'common.Block' --input 'blockfile_000000' --output 'block.json'
```

Docker를 이용해서 Hyperledger-Fabric Network를 구성하는 경우, volume을 host와 연결시켜서 용량이 증가하는 것에 유연하게 대응할 수 있도록 해야 한다. 하지만 의도하지 않은 수정을 방지하기 위해 해당 폴더를 액세스 제어 등을 통해서 보호해야 한다.