### React Fiber

- React Fiber는 React 16.2부터 새롭게 소개된 **새로운 “재조정(Reconciliation) 엔진”**
- React가 Virtual DOM을 비교(diffing)하고, 그 결과를 DOM에 반영하는 **내부 알고리즘(렌더링 엔진)**을 새로 다시 설계
- React 15 -> **Stack Reconciler** 사용 (동기적 렌더링)
- React 16 이후 -> **Fiber Reconciler** 도입 (비동기 렌더링 지원)

### Pre-React_Fiber Era

- 기존 React 15의 렌더링은 단일 호출 스택 기반
- 즉, 컴포넌트 트리를 렌더링하다가 중간에 멈출 수 없었음
- 큰 트리 렌더링 시 **UI가 멈춤 (blocking)**
- `requestAnimationFrame`, `setTimeout` 등의 **우선순위 제어 불가능**
- CPU를 많이 사용하는 렌더링이 **메인 스레드를 독점**

이를 해결하기 위해 React 팀은 **렌더링을 “쪼갤 수 있는 구조”**로 완전히 재설계 --> **Fiber 아키텍처**

### The work of React Fiber

#### 1. 렌더링 작업을 쪼갬 (Interruptible Rendering)

- Fiber는 각 컴포넌트를 “작은 단위의 작업 단편(work unit)”으로 표현
- 이를 **Fiber Node**라고 합니다.
- 이제 React는 트리를 한 번에 전부 처리하지 않고
- 한 노드씩 순차적으로, **중단 가능한 방식**으로 처리가능.

> 즉, React는 렌더링을 하다가
>
> “지금은 사용자 입력이 더 급하네” → 멈춤 → “다시 렌더 이어가기”
>
> 같은 동작이 가능

- 기존 방식의 문제점

1. **렌더링이 중단 불가능함 (Blocking)**
   - 트리가 크거나 복잡하면, diffing 동안 **메인 스레드가 완전히 점유됨**
   - 사용자가 클릭하거나 스크롤을 해도 UI가 멈춤
2. **매번 새 Virtual DOM 객체를 전부 생성**
   - 이전 Virtual DOM은 폐기 → GC 부담 증가
   - React는 모든 노드를 새로 diff해야 해서 비효율적

#### 2. 우선순위 기반 스케줄링 (Scheduling Priority)

- Fiber는 각 작업 단위에 “우선순위”를 부여
- 예
  | 우선순위 | 예시 |
  | --- | --- |
  | 매우 높음 | 사용자 입력 이벤트 (클릭, 타이핑 등) |
  | 보통 | 화면 업데이트 (state 변경) |
  | 낮음 | 백그라운드 데이터 로딩 등 |

- React는 **먼저 처리해야 하는 렌더링부터 수행**하고 덜 중요한 업데이트는 나중으로 미루거나 중단 가능

#### 3. 노드 재사용 (Node Reuse)

React Fiber는 컴포넌트 트리를 구성할 때 **이전 Fiber 노드 객체를 재활용**
즉, 매번 새로운 객체를 만드는 대신, 기존 Fiber 구조를 업데이트

- 불필요한 GC(Garbage Collection) 감소
- 메모리 효율 개선
- 빠른 diffing 가능

#### 4. 렌더링과 커밋의 분리 강화

Fiber는 **Render Phase**와 **Commit Phase**를 명확히 분리

| Phase            | 설명                                                         |
| ---------------- | ------------------------------------------------------------ |
| **Render Phase** | Fiber 노드를 순회하며 업데이트 계산 (비동기 가능, 중단 가능) |
| **Commit Phase** | 실제 DOM 조작 수행 (동기, 중단 불가)                         |

→ 이렇게 해서 React는 “렌더링 계산”과 “화면 반영”을 분리해 더 정교한 스케줄링이 가능

#### 5. 정리

| 항목          | React 15 (이전) | React 16+ (Fiber)                 |
| ------------- | --------------- | --------------------------------- |
| 렌더링 구조   | Stack 기반      | Fiber                             |
| 중단 가능성   | ❌ 불가능       | ✅ 가능                           |
| 우선순위 제어 | ❌ 없음         | ✅ 있음                           |
| 성능 최적화   | 제한적          | Node 재사용, Scheduling 지원      |
| 대표 기능     | 동기 렌더링     | Concurrent Rendering, Suspense 등 |

### References

https://d2.naver.com/helloworld/2690975

https://nami-socket.tistory.com/36

https://medium.com/@sht02048/%EC%83%9D%EA%B0%81%EB%B3%B4%EB%8B%A4-%EB%8D%94-%EC%84%AC%EC%84%B8%ED%95%9C-%EB%A6%AC%EC%95%A1%ED%8A%B8-%EB%A0%8C%EB%8D%94%EB%A7%81-%EA%B3%BC%EC%A0%95-feat-%EB%A6%AC%EC%95%A1%ED%8A%B8-%ED%8C%8C%EC%9D%B4%EB%B2%84-44075084381a

https://djoung.tistory.com/entry/2%EC%9E%A5-2-%EA%B0%80%EC%83%81-DOM%EA%B3%BC-%EB%A6%AC%EC%95%A1%ED%8A%B8-%ED%8C%8C%EC%9D%B4%EB%B2%84
