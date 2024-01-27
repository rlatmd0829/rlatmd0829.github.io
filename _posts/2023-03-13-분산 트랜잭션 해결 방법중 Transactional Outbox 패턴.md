---
title: 분산 트랜잭션 해결 방법중 Transactional Outbox 패턴
date: 2023-03-13 12:00:00 +09:00
categories: [MSA, Transaction]
tags: [MSA, Transaction]
---

## Transactional Outbox 패턴

이 패턴은 데이터베이스의 트랜잭션 내에서 발생하는 이벤트를 캡처하고 이벤트를 외부 메시지 큐 또는 저장소에 안전하게 기록하는 방법을 다룹니다. 주로 데이터베이스 트랜잭션 외에도 외부 시스템에 변경 사항을 전파해야 할 때 사용됩니다.

### 내용

주문 데이터베이스에 새로운 주문을 추가하고, 이벤트를 발행하기 직전에 서비스가 끊기게 된다면 어떻게 될까요? 보통 클라우드에서 대규모 시스템을 운영하는 경우 이런 문제는 충분히 일어날 수 있습니다.

새로운 주문은 주문 데이터베이스에 있지만, 연계된 다른 서비스들은 이벤트를 전달 받지 못하는 상황이 됩니다.

이런 문제를 해결하기 위한 방법은, **데이터베이스를 업데이트하는 트랜잭션 안에 발행해야할 메시지를 데이터베이스에 저장**하고, 별도의 프로세스가 데이터베이스에 저장된 이벤트를 읽어서 메시지 브로커에 전송하는 것입니다.

- 메시지를 발행하는 시스템은 **데이터베이스 트랜잭션과 함께 이벤트를 '아웃박스' 테이블에 저장하고**, 이를 별도의 프로세스가 메시지 브로커로 전송합니다. 


- 트랜잭션이 제공하는 원자성을 이용해, 발행하지 않아야 하는 메시지가 대기열 테이블에 들어가거나, 발행해야 하는 메시지가 대기열 테이블에 들어가지 않는 문제를 방지합니다.


- 이후 저장된 메시지를 성공할 때까지 발행합니다(At-Least Once Delivery). 메시지를 수신하는 시스템은 같은 메시지를 여러 번 수신해도. 한 번 수신한 것과 같이 처리해, 메시지가 정확히 한 번 처리되도록(Exactly-Once Processing) 합니다.

이 방식을 **Transactional Outbox** 패턴이라고 합니다


### 기존 Choreography  saga 패턴 적용한 그림

<img width="706" alt="image" src="https://github.com/rlatmd0829/rlatmd0829.github.io/assets/70622731/2b12f37e-fffa-4a38-b0e0-9d1b5355e4af">


### 위에 saga 패턴에 Transactional Outbox 패턴 적용한 그림

<img width="728" alt="image" src="https://github.com/rlatmd0829/rlatmd0829.github.io/assets/70622731/04d50c20-d568-4470-b7e0-4f1329a2ff35">


### 보통 saga패턴 또는 event sourcing 패턴과 같이 사용하는 경우가 많습니다.

- Saga 패턴은 여러 서비스 간의 긴 트랜잭션을 관리하고, 각 단계에서의 트랜잭션 문제를 처리하여 시스템을 일관된 상태로 유지하는 데 중점을 둡니다. 이 과정에서 트랜잭션의 각 단계에서 이벤트를 발생시키는 경우, Transactional Outbox 패턴을 사용하여 이벤트를 안전하게 기록하고 외부 시스템으로 전달할 수 있습니다.

- Event Sourcing 패턴은 시스템의 상태 변경을 추적하고 이력을 기록하면서, Transactional Outbox 패턴은 데이터베이스 트랜잭션 내에서 발생하는 이벤트를 안전하게 기록하고 외부 시스템으로 전파합니다.







-----------



### Reference

- https://devocean.sk.com/blog/techBoardDetail.do?ID=165445
- https://blog.gangnamunni.com/post/transactional-outbox
