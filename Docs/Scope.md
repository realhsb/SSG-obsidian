# 개념
TCA에서의 scope는 부모 ➡️ 자식으로 상태와 액션을 분리해서 전달할 때 사용한다.
큰 기능 안에서 작은 기능을 **독립적으로** 관리할 때 꼭 필요하다!

> 즉, 부모 도메인 (State/Action/Store/Reducer)에서 자식 도메인만 분리해서 연결하는 터널 역할

## View에서의 scope
부모 Store를 자식 View로 넘길 때 사용
`store.scope(...)`

### 예제: View에서 store 스코핑(Store 기준)
```swift
StandupDetailView(
  store: parentStore.scope(
    state: \.detail, // 자식 액션을 부모 액션으로 포장
    action: { .detail($0) } // 자식 액션을 부모 액션으로 포장
  )
)
```

자식 View는 자기 상태/액션만 알면 되기 때문에, 재사용성과 테스트성이 향상된다!

## Reducer에서의 Scope
부모 Reducer 안에서 자식 Reducer를 통합할 때
`Scope(...) { ... }`

### 예제: Reducer에서 스코핑 (Reducer 기준)
```swift
Scope(
	state: \.detail, // 부모 상태에서 자식 상태 가져옴
	action: /Action.detail // 부모 액션 중 .detail 케이스만 추림
) {
  StandupDetailFeature() // 해당 도메인에 자식 reducer 연결
}
```
부모 reducer에서 자식 reducer를 하위 도메인으로 위임하는 구조!

## scope가 중요한 이유
1. 재사용성
- 자식 feature는 부모와 독립적으로 테스트/개발 가능하다.
- 여러 곳에서 재사용이 가능하다.

2. 모듈화
- 자식 feature를 별도 모듈로 분리해도 부모와 느슨하게 연결된다.

3. 유지보수
- 부모 상태가 커져도 자식은 자기가 관리하는 상태만 알면 된다.

TCA에서 제공하는 여러 스코핑 연산자 중 Scope에 대해 간단히 알아보았다.

밑에서는 scope 함수에 사용된 Swift 문법과 동작 원리에 대한 설명이다.
# scope 동작 원리