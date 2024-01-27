---
title: 분산 트랜잭션 해결 방법중 Event Sourcing
date: 2023-03-07 12:00:00 +09:00
categories: [MSA, Transaction]
tags: [MSA, Transaction]     
---

# Event Sourcing

Event Sourcing은 순차적으로 발생하는 이벤트를 모두 저장하는 패턴입니다.

Command에 의해 Processor 가 생성한 Event를 Event Store에 저장한다. 이렇게 저장한 데이터를 Event Handler 를 통해 과거부터 지금까지 이벤트를 확인하여 현재 상태(state)를 유추해냅니다.

이벤트 소싱에서는 데이터를 삭제(delete) 하거나 수정(update) 하지 않고 삽입(insert) 에 대한 작업만 수행합니다.

<img width="780" alt="image" src="https://github.com/rlatmd0829/rlatmd0829.github.io/assets/70622731/fd46dd6e-d978-432b-9e89-b7cc433ed132">
￼

이벤트 소싱같은 경우에는 데이터를 저장하는데에는 적합하지만, 읽기에는 비효율적입니다. 그래서 CQRS와 같이 사용합니다.



### CQRS

시스템의 상태를 변경하는 Command 와 시스템을 조회만 하는 Query를 분리하는 마이크로 서비스 패턴을 말합니다.

<img width="832" alt="image" src="https://github.com/rlatmd0829/rlatmd0829.github.io/assets/70622731/ee27ed76-eb81-4b8b-9635-f75b301f3a7c">


Event Sourcing과 CQRS 가 서로의 단점을 보완해주기 때문에 같이 사용하는 경우가 많습니다.



### EventSourcing에 단점을 CQRS가 보완


Event Sourcing에서 Event는 상태가 변경되는 경우만 기록하면 됩니다. 단순 조회에 경우는 모델의 정보를 변화시키지 않기 때문에 Event를 발행할 필요가 없습니다.

그래서 CQRS를 적용하여 읽기와 쓰기 db를 나눠 쓰기에서는 이벤트를 발행하고 읽기 db에서는 빠르게 조회를 수행할수 있도록 진행합니다.

따라서 이벤트 소싱의 단점인 READ에 적합하지 않다는 문제를 해결할 수 있습니다.



### CQRS에 단점을 EventSourcing이 보완

CQRS는 읽기 DB와 쓰기 DB가 분리되어 사용해 데이터 불일치가 발생할 수 있습니다. 이벤트 소싱은 이 단점을 보완해줍니다.

Command로 변경된 Aggregate를 Event Store에 저장하여, Event Handler로 Read db의 데이터 일관성을 보장해주는 것입니다.

Read db를 따로 두어 이벤트발생시 저장해주고 query에서 접근할때는 읽기 db로 접근하여 읽어올수 있도록 하면됩니다.


### 다른방법

그러나 Event Sourcing이 분산 시스템에서 모든 분산 트랜잭션을 해결하는 완벽한 솔루션은 아닙니다. 특히 복잡한 트랜잭션 처리나 일부 동기화된 작업의 관리 등에 있어 추가적인 고려가 필요할 수 있습니다.


Saga 패턴과 Event Sourcing은 각각 분산된 트랜잭션 관리와 상태 관리를 위해 다른 방식으로 접근하지만, 이전 상태로 롤백하거나 보상하는 개념은 Saga 패턴에서 주로 사용되며, Event Sourcing은 상태 변경 이력을 기록하고 성공적인 상태를 반영하면서 실패한 상태를 보상적으로 해결합니다.



-----------



### Reference

- https://velog.io/@borab/%EB%B6%84%EC%82%B0-%ED%99%98%EA%B2%BD%EA%B3%BC-Event-Driven-Architecture
- https://doqtqu.tistory.com/337
