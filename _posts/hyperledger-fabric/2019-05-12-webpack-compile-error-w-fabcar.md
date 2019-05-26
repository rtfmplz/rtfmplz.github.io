---
layout: post
title: Webpack compile error with "fabric-client"
tags: [hyperledger-fabric, node.js, webpack, browser, troubleshooting]
author: Jae
comments: true
---


# Webpack compile error with "fabric-client"

> Vue.js를 공부하면서 fabcar dapp을 node.js가 아닌 vue.js를 기반으로 동작하도록 upgrade 하려했다.
> 
> 결국 실패 했는데, 그 이유 기록한다.

## Fabcar

Fabcar는 Hyperledger-fabric의 기본 dapp 예제로서 [fabric-samples](https://github.com/hyperledger/fabric-samples)에 포함되어 있다.

Fabcar를 싱행시키기 위해서는 아래와 같은 선행 조건이 필요하다.

1. 운영중인 HLF network 있고, fabric-ca 가 동작 중이어야 한다.
2. HLF에 "fabcar" chaincode가 설치되어 있어야 한다.

위의 선행 조건이 충족되면 fabcar 폴더에서 `npm install` 을 수행해서 fabcar를 실행하는데 필요한 패키지들을 설치한다.

이후 아래와 같은 명령들을 실행할 수 있다.

1. `node enrollAdmin.js`
2. `node registerUser.js`
3. `node query.js`
4. `node invoke.js`

## Webpack

> 본 post는 HLF 1.3을 기반으로 작성되었다. 2019년 5월 기준 HLF 1.4가 배포되어 fabcar도 fabric-client가 아닌 fabric-network api를 사용하도록 변경되었지만, 본 글에서 다루고자 하는 내용에는 영향을 주지 않는다.

node.js 기반으로 동작하는 fabcar는 `fabric-client`, `fabric-ca-client`, `fs` 등 다양한 javascript library를 사용해서 구현되어있는데, 이런 fabcar를 vue.js를 이용해서 browser에서 동작하는 web application으로 바꾸기 위해서 bundler를 이용해서 의존관계에 있는 package 들을 하나의 javascript로 묶어줘야 하고, vue.js 는 webpack(bundler)을 사용해서 이런 동작을 할 수 있게 지원한다.

> webpack 의 상세 동작 및 자세한 설정들은 아직 깊게 공부하지 못해서 아래 링크로 대체한다.
>
> * https://webpack.js.org/
> * https://d2.naver.com/helloworld/0239818
> * https://blog.hanumoka.net/2018/08/26/vue-20180826-web-webpack/

## Create Project and Run

vue.js를 기반 fabcar를 만드는 작업은 다음의 절차로 작업을 수행했다.

* `vue-cli`를 설치

```bash
npm install -g vue-cli
```

* vue.js project 생성

```bash
vue init webpack-simple myproject
```

* fabcar의 기능들을 vue.js를 통해서 html과 연결
* bundle.js 생성 및 로컬 서버를 통해 index.html 실행

```bash
npm run dev
```

## Error !!!

위의 절차 중 마지막 단계인 `npm run dev`를 수행하면, webpack이 사용된 모듈의 의존관계를 파악해서 하나의 bundle.js 파일로 만들고, 로컬 서버(`localhost:8080`)가 실행되어 web browser에서 index.html의 내용을 확인할 수 있게한다.

다른 문제가 없다면, index.html 페이지가 아니라, 아래와 같은 error가 browser에 표시된다.... -_ -...

```javascript
Failed to compile.

./node_modules/nconf/node_modules/os-locale/index.js
Module not found: Error: Can't resolve 'child_process' in '/Users/jae/git/fabric-playground/fabcar-vuejs/node_modules/nconf/node_modules/os-locale'
 @ ./node_modules/nconf/node_modules/os-locale/index.js 2:19-43
 @ ./node_modules/nconf/node_modules/yargs/index.js
 @ ./node_modules/nconf/lib/nconf/stores/argv.js
 @ ./node_modules/nconf/lib/nconf/stores ^\.\/.*$
 @ ./node_modules/nconf/lib/nconf.js
 @ ./node_modules/fabric-ca-client/lib/Config.js
 @ ./node_modules/fabric-ca-client/lib/utils.js
 @ ./node_modules/fabric-ca-client/lib/FabricCAServices.js
 @ ./node_modules/fabric-ca-client/index.js
 @ ./src/fabcar/enrollAdmin.js
 @ ./src/main.js
 @ multi (webpack)-dev-server/client?http://localhost:8080 webpack/hot/dev-server ./src/main.js
```

## Cause Analysis

google 에서 error를 검색해보면 [이 곳](https://github.com/webpack/webpack/issues/744)에서 error의 이유를 확인할 수 있다. 이유를 정리하면 다음과 같다.

`fabric-ca-client`가 사용하는 `nconf` 모듈은 `child_process` 모듈을 사용하는데, `child_process` 모듈은 node.js에 내장된 모듈로, browser에서 사용할 수 없기 때문에 webpack에서 bundle.js에 포함시키지 않는 것 같다.

`package.json`에 `"browser": { "fs": false, "child_process": false }` 내용을 추가해서, `child_process` 를 `None object`로 대체하도록 할 수 있지만, 그럼 `fabric-ca-client`가 제대로 동작하지 않으므로 의미가 없다.

## Result

Javascript library를 사용할 때는 Browser 기반과 node.js 기반을 구분해서 사용하자. 끝!