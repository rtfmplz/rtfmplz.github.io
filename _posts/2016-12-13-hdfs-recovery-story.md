---
layout: default
title: DataNode의 H/W 스펙이 다른 Hadoop 클러스터 운영 경험기
category: post
author: Jae
---

# DataNode의 H/W 스펙이 다른 Hadoop 클러스터 운영 경험기

## Ambari Alert!!!!!

* DB Data Migration을 10억건 쯤 하고 있는 상황에서 `hdp-d1`에 설치된 Secondary NameNode가 죽었다.
* 추가로 Ambari에서 alert를 미친듯이 내뱉기 시작하고 DataNode에 설치된 YARN node manager가 뻗기 시작했다.

**뭐지!!!!!!!!!!!!!!!!!!!!!!!! @#$^&*()(*&^%$#$%^&@#$%^&**

> 모르겠다 Spark Application 다 멈추고 재부팅 고고
> 
> Restart를 했는데 HDFS가 safs mode로 들어가고 재시작이 되지 않는다................
> 
> 자 이제부터 문제 상황을 파악해 보자

## 문제 상황 파악

* 일단 처음 문제가 발생한 Secondary NameNode를 의심
	* NameNode 망가졌으니 싹 밀고 깔끔하게 다시 깔아야함 이라고 옆에서 훈수가 들어왔다.... 실행에 옮겼으면 진짜 울뻔......
	* 울고 싶지 않으니까 Log를 보고 문제를 파악해 보자
* 문제가 발생한 DataNode 는 두대
	* hdp-d1
	* hdp-d4
* Ambari Alert Error Log를 보니 DataNode의 Data Directory가 90% 이상 찼다????
	* 뭔소리야 `Ambari > HDFS > Disk Remaining` 보면 300TB 이상 남았는디....?
	* 뭐 어쨌든 저장용량이 이상하다니까... hdp-d1에서 `df -h`
		* 느낌이 쎄하다..... 뭐지............
		* hdp-d1 서버의 Root 영역의 저장 공간이 97% Used....
* 환경 설정이 잘못되어 있나??
	* `Ambari > HDFS > Configs > DataNode > DataNode Directories` 를 확인
		* 띄어 쓰기가 잘못되있나...
		* Ambari Web은 ClouderaManager보다 보기도 찾기도 겁나 어렵게 되어 있다....
* 음 근데 DataNode Directories 설정이 하나 뿐이야???
	* 이거다!!!!
	* DataNode의 HDD 갯수가 다른데 Directories 설정이 각각 있어야지!!!!


## 수수께끼는 모두 풀렸다.

* 설치 시 DataNode의 Directory 설정을 잘못함
	1. 현재 1Prostaging 환경의 DataNode들의 H/W 스펙이 각기 다름... (여기저기서 주서다가..)
	2.	Ambari가 각기 다른 DataNode들의 H/W 스펙을 반영한 Directory 설정을 생성하지 못함
	3.	다른 DataNode들에 비해 적은 HDD를 보유한 hdp-d1, hdp-d4가 HDD 수를 초과하는 Data Directory의 저장공간을 Root Directory로 사용함
	4.	Root Directory에 HDFS의 Block 데이터들이 쌓여서 저장 용량이 부족
* 덩달아 서버가 모자라서 hdp-d1에 더부살이 하던 Secondary NameNode도 사망

> 테스트 용도로 임시로 구축하는거라고 유휴 서버 이것저것을 모아서 만들다 보니 DataNode의 스펙이 제각각이었다.  
> 나중에 알았지만 HDFS는 클러스터의 DataNode의 설정을 Group 별로 다르게 줄수 있다. 하지만, 관리가 불편하므로 DataNode의 스펙은 통일하는것을 권장한다.

## 이제, Hadoop을 살려보자

* 일단 Root 용량을 확보해서 HDFS를 실행했다.
	* root 영역의 Data를 다른 디스크로 복사 후 Softlink로 연결 해서 임시로 용량 확보
		* 자 실행되라!!!!!!!!
		* 안된다................ WHY!!

> HDFS가 실행되지 않으면 `Safe Mode Status`를 확인한다.
> Safe Mode이면 Hadoop이 실행되지 않음.. 아래 command로 강제로 Safe Mode를 해제 한다.
> * `hadoop dfsadmin -safemode leave`
> * 어떤 경우에 safe mode로 들어가는지는 잘 모르겠음.....

## 다시, Hadoop 실행!!

![ambari](/images/hdfs-recovery-story/ambari.jpeg)

## Under Replicated Block v.s. Missing Block

* 일단 실행은 했는데 alert가 미친듯이 뜨기 시작한다..
* 위의 캡쳐 화면은 아직 ambari가 아직 시스템을 다 인식하지 못해서 alert를 띄우기 전..... 히히
* Under Replicated Block ??? Missing Block ????

### Under Replicated Block

* Node Manager가 2대나 죽어 있는 상황에서 DB Data Migration 작업을 계속 하고있었으니 Replication 개수가 3개가 안되는 Block이 9만개나 됨...

> 참고. Hadoop의 기본 Replication Factor는 기본이 3 이다.

* 자 이제 Under Replicated Block을 복구해보자
	* Under Replicated Block은 알아서 복구가 되어야 할 것 같은데....
	* 알아서 자동적으로 뿅 되지는 않더라..

* 일단은 Auto로 안되니까 Manually 하게 Under Replicated Block을 복구해보자
	* 친절한 hortonworks에서 script를 제공해준다.
	* [Fix Under-replicated blocks in HDFS manually](https://community.hortonworks.com/articles/4427/fix-under-replicated-blocks-in-hdfs-manually.html)

> 나중에 알았지만 기다리면 Hadoop 이 알아서 처리해준다.. ^^..

```bash
$ su - <$hdfs_user>
$ hdfs fsck / | grep 'Under replicated' | awk -F':' '{print $1}' >> /tmp/under_replicated_files
$ for hdfsfile in `cat /tmp/under_replicated_files`; do echo "Fixing $hdfsfile :" ;  hadoop fs -setrep 3 $hdfsfile; done

```

### Missing Block

* Missing Block은 Under Replicated Block 과 다르게 복구할 방법이 없다.. OTL......
* 다시 생성해야 한다...
* 뭔줄 알고 다시 생성하지??
	* 아래 command로 검색할 수 있다.

```bash
$ hadoop fsck -list-corruptfileblocks

blk_1073741825  /hdp/apps/2.5.0.0-1245/mapreduce/mapreduce.tar.gz
blk_1073752128  /data/mover/dnadb/file/0027000000_0028000000/part-r-00001-ebc61d4a-f0de-4c95-b3de-ac550ec98adc.gz.parquet
blk_1073773574  /data/mover/dnadb/file/0134000000_0135000000/part-r-00002-799f76dd-fe5f-4940-8b4c-9cccd86270aa.gz.parquet
blk_1073801649  /app-logs/kyungjaelee/logs/application_1480385517060_0277/hdp-d4.asd.ahnlab.com_45454_1480442360426

... <생략> ...

blk_1074544278  /data/mover/file/1200000000_1202000000/part-00080
blk_1074555839  /data/mover/file/2000000000_2002000000/part-00191
blk_1074618913  /data/mover/file/0530000000_0532000000/part-00030
blk_1074632666  /data/mover/file/0664000000_0666000000/part-00085
blk_1074638542  /data/mover/file/0720000000_0722000000/part-00165
```

#### Missing Block v.s. Normal Block

* 다음 command를 이용해서 HDFS내 Data Block의 내용을 확인할 수 있다.
	* Missing Block과 Normal Block의 내용을 비교해 보자.
	* Average block replication 항목을 보면  Missing Block 은 0 이다........ OTL......

```bash
$ hdfs fsck <HDFS 경로> -files -blocks -locations
```

```bash
# Log For Missing Block

$ hdfs fsck /data/mover/dnadb/file/0202000000_0203000000/_common_metadata -files -blocks -locations

Connecting to namenode via http://hdp-m1.asd.ahnlab.com:50070/fsck?ugi=hdfs&files=1&blocks=1&locations=1&path=%2Fdata%2Fmover%2Fdnadb%2Ffile%2F0202000000_0203000000%2F_common_metadata
FSCK started by hdfs (auth:SIMPLE) from /172.18.200.115 for path /data/mover/dnadb/file/0202000000_0203000000/_common_metadata at Tue Dec 13 11:25:20 KST 2016
/data/mover/dnadb/file/0202000000_0203000000/_common_metadata 1090 bytes, 1 block(s):
/data/mover/dnadb/file/0202000000_0203000000/_common_metadata: CORRUPT blockpool BP-755980960-172.18.200.111-1480052019963 block blk_1073787288
 MISSING 1 blocks of total size 1090 B
0. BP-755980960-172.18.200.111-1480052019963:blk_1073787288_46464 len=1090 MISSING!

Status: CORRUPT
 Total size:    1090 B
 Total dirs:    0
 Total files:   1
 Total symlinks:                0
 Total blocks (validated):      1 (avg. block size 1090 B)
  ********************************
  UNDER MIN REPL'D BLOCKS:      1 (100.0 %)
  dfs.namenode.replication.min: 1
  CORRUPT FILES:        1
  MISSING BLOCKS:       1
  MISSING SIZE:         1090 B
  CORRUPT BLOCKS:       1
  ********************************
 Minimally replicated blocks:   0 (0.0 %)
 Over-replicated blocks:        0 (0.0 %)
 Under-replicated blocks:       0 (0.0 %)
 Mis-replicated blocks:         0 (0.0 %)
 Default replication factor:    3
 Average block replication:     0.0
 Corrupt blocks:                1
 Missing replicas:              0
 Number of data-nodes:          4
 Number of racks:               1
FSCK ended at Tue Dec 13 11:25:20 KST 2016 in 1 milliseconds

The filesystem under path '/data/mover/dnadb/file/0202000000_0203000000/_common_metadata' is CORRUPT
```

```bash
# Log For Normal Block

$ hdfs fsck /data/mover/dnadb/file/0203000000_0204000000/_common_metadata -files -blocks -locations

Connecting to namenode via http://hdp-m1.asd.ahnlab.com:50070/fsck?ugi=hdfs&files=1&blocks=1&locations=1&path=%2Fdata%2Fmover%2Fdnadb%2Ffile%2F0203000000_0204000000%2F_common_metadata
FSCK started by hdfs (auth:SIMPLE) from /172.18.200.115 for path /data/mover/dnadb/file/0203000000_0204000000/_common_metadata at Tue Dec 13 11:29:54 KST 2016
/data/mover/dnadb/file/0203000000_0204000000/_common_metadata 1090 bytes, 1 block(s):  OK
0. BP-755980960-172.18.200.111-1480052019963:blk_1073787492_46668 len=1090 repl=3 [DatanodeInfoWithStorage[172.18.200.119:50010,DS-6a4ed626-167d-4950-a080-49f9b0237dc7,DISK], DatanodeInfoWithStorage[172.18.200.114:50010,DS-7004f08e-dfcb-4288-acab-fb4fe352f78b,DISK], DatanodeInfoWithStorage[172.18.200.112:50010,DS-a9363aee-72c6-445e-acfe-0e1814c61a03,DISK]]

Status: HEALTHY
 Total size:    1090 B
 Total dirs:    0
 Total files:   1
 Total symlinks:                0
 Total blocks (validated):      1 (avg. block size 1090 B)
 Minimally replicated blocks:   1 (100.0 %)
 Over-replicated blocks:        0 (0.0 %)
 Under-replicated blocks:       0 (0.0 %)
 Mis-replicated blocks:         0 (0.0 %)
 Default replication factor:    3
 Average block replication:     3.0
 Corrupt blocks:                0
 Missing replicas:              0 (0.0 %)
 Number of data-nodes:          4
 Number of racks:               1
FSCK ended at Tue Dec 13 11:29:54 KST 2016 in 1 milliseconds

The filesystem under path '/data/mover/dnadb/file/0203000000_0204000000/_common_metadata' is HEALTHY
```

* Missing Block은 더이상 사용할 수 없기 때문에 아래 Command를 이용해서 `hdfs://lost+found`로 옮기거나 삭제 할 수 있다.

```bash
$hadoop fsck -move		# /lost+found 로 이동
$hadoop fsck -delete	# 삭제
```

* Missing Block은 말그대로 Hadoop에서 포기한 Block들이다..
* 아쉽지만 다시 생성해야 한다.......... Good Luck...

## Recovery

* 2 Alerts 는 갑자기 작업량이 많아지면 생긴 alert 이니까 신경쓰지 말자..... ㅋㅋ
* 극뽁, 얏호

![recovery](/images/hdfs-recovery-story/recovery.jpeg)

## 결론

* DataNode의 H/W 스펙이 다르다면 Group 을 통해서 DataNode 설정을 각각 관리할 수 있다.
	* 관리가 귀찮을 수 있지만, 각각 관리하지 않으면 재앙이 발생할 수 있다.
	* 귀찮으면 H/W 스펙을 맞추자..
* Under Replicated Block은 시간이 지나면 Hadoop이 알아서 처리해준다.
* Missing Block은 리스트를 확인한 후 다시 생성해야 한다.

