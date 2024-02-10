---
title: Jmeter를 활용한 Websocket 성능테스트
date: 2023-07-21 12:00:00 +09:00
categories: [성능테스트, Jmeter]
tags: [Jmeter, 성능테스트, http, websocket, JSON-RPC]
---

## Jmeter 설정

<hr style="height: 2px; border: none; background-color: white;" />

JMeter는 Apache 소프트웨어 재단에서 개발한 오픈 소스의 Java 기반의 성능 테스트 도구입니다. 주로 웹 응용 프로그램의 성능을 측정하고 부하 테스트를 수행하는 데 사용됩니다

brew 명령어를 이용해 jmeter를 간단하게 설치 할 수 있습니다.

```shell
brew install jmeter
```

설치가 완료되었으면 해당 명령어를 이용해 Jmeter를 실행해줍니다.

```shell
open /opt/homebrew/opt/jmeter/bin/jmeter
```

그러면 아래와 같은 화면이 뜨면서 실행됩니다.

![image](https://github.com/rlatmd0829/rlatmd0829.github.io/assets/70622731/980947f5-5c68-48d6-8040-2e335dfb6a35)

<br>

## Jmeter Websocket 테스트

<hr style="height: 2px; border: none; background-color: white;" />

Jmeter에서 Websocket 테스트를 하기 위해서는 websocket plugin을 다운받아야 합니다.

JMeter 상단의 options 탭 -> Plugins Manager를 클릭하고,

Plugin Manager에서 상단의 Available Plugins 탭을 클릭 하여 WebSocket Sampler by Peter Doornbosch를 설치하면 됩니다.

<img width="1021" alt="image" src="https://github.com/rlatmd0829/rlatmd0829.github.io/assets/70622731/ddb4a7ca-8f0d-44be-8148-5c70b0983049">

테스트 순서를 알아보겠습니다.

1. 먼저 Thread Group을 추가해서 유저를 몇명 테스트 할 건지 정합니다.
2. Thread Group > Add > Sampler > WebSocket Open Connection 을 만들어서 웹소켓 연동할 서버 주소를 넣습니다.
3. Thread Group > Add > Sampler > WebSocket Single Write Sampler 에서 연동된 웹소켓으로 Data를 보냅니다.

### 웹소켓 연결

<img width="1008" alt="image" src="https://github.com/rlatmd0829/rlatmd0829.github.io/assets/70622731/573a895a-9d46-4d4b-bb3e-635d5e959ff6">

### 메세지 전송

연동된 웹소켓으로 전송하고 싶은 메세지를 전송합니다.

<img width="1012" alt="image" src="https://github.com/rlatmd0829/rlatmd0829.github.io/assets/70622731/10eeead3-1244-40ee-819c-9a484b8a6d3b">


### Header에 Authorization Token 추가

Websocket Open Connection > Add > Config Element > HTTP Header Manager 을 만들어서 토큰을 넣어줍니다.

<img width="606" alt="image" src="https://github.com/rlatmd0829/rlatmd0829.github.io/assets/70622731/e2a896aa-a382-4cd3-9edf-86c03b2bbf4b">

### csv를 활용하여 변수에 넣어 사용하는법

값을 넣어야할게 많을 경우에는 CSV Data Set Config 를 활용해서 csv를 넣어서 값을 가져올 수 있습니다.

<img width="1016" alt="image" src="https://github.com/rlatmd0829/rlatmd0829.github.io/assets/70622731/954a959c-cd72-41a0-816a-94266392ceb6">

열별로 네이밍을 나눌때에는 `,` 를 사용하여 구분하면 됩니다.

### 이전요청 응답값을 변수에 넣어 사용하는법

응답값이 아래와 같이 들어올 때 JSON Extractor를 사용하여 `data.token` 을 가져와 변수에 넣을 수 있습니다.

```json
"data": {
  "token": "jqhudoajoidfaidoa"
}
```

아래와 같이 JSON Extractor 를 사용하면 token 변수에 응답값을 넣을 수 있어 다른곳에서도 사용 가능합니다.

<img width="1011" alt="image" src="https://github.com/rlatmd0829/rlatmd0829.github.io/assets/70622731/ad3b878b-c2cf-4507-981d-21323318b3b2">


이전 요청 응답값으로 나온 data.token 값을 token으로 넣을 수 있고 csv에서 가져온 값도 다음 요청에서 활용할 수 있습니다.

<img width="1009" alt="image" src="https://github.com/rlatmd0829/rlatmd0829.github.io/assets/70622731/bb8fd631-e71d-4031-965a-998c60d5803e">


### 그 외 테스트에 도움이 될만한 내용

- Loop Controller를 사용하면 특정 구간만 반복 호출할 수 있습니다.
- Constant Timer를 사용하면 요청할 때 딜레이를 줄 수 있습니다.
- View Result Tree를 사용하면 request response를 확인할 수 있습니다.
- Graph Result를 사용하면 요청에 대한 그래프를 보여줍니다.
- Summary Report를 사용하면 결과 요약을 보여줍니다.
