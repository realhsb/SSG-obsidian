# 목표
이번 강의는 ... Navigation을 어떻게 설계할지 배운다.
> the most interesting thing to look at next is how can we navigate to this new detail screen from the standups list

특히, standups list 뷰에서 detail 뷰로 이동하는 방법 구현이 주된 목표이다.

총 2가지 방법이 존재한다.
- Tree-based navigation
- Stack-based navigation

## Tree-based navigation
트리 기반 내비게이션은 옵셔널 타입의 모델의 상태 변화에 따라 화면을 전환한다.
- `nil` -> 화면 닫힘
- `non-nil` -> 화면 열림
예: `var detail: DetailState?`

중첩이 가능해서 트리 구조처럼 표현된다.
단순하고, `SwiftUI`의 `sheet`나 `fullScreenCover`와 잘 맞는다.
이 앱에서 이미 사용 중이다. (ex. `sheet`로 `StandupForm` 띄울 때)

## Stack-based Navigation
배열로 상태 표현한다.
예: `var path: [DetailState]`

여러 화면을 쌓거나 drill-down할 때 자연스럽고 표현력이 좋다
이번 강의에서 **리스트 -> 디테일 화면 이동**에 이 방식을 선택했다.

# Detail 화면으로 이동하기
## NavigationStack 위치 정하기
`NavigationStack`은 앱 루트에 설치해야 한다! 왜? 특성 feature에 종속되는 것을 피해 다른 feature를 독립적으로 컴파일한다.

즉, StandupsListView를 감싸는 AppView를 만들어서 NavigationStack을 선언한다.
(AppView에서는 앱의 루트 내비게이션을 독립적으로 관리한다.)
[[왜 NavigationStack이 뷰 안쪽에 있으면 문제일까?]]

만약, NavigationStack을 StandupsListView 내부에 설치하면,
- Detail 및 RecordingMeeting 같은 하위 기능은 분리 가능
- 그러나 StandupsList 기능을 수정할 때마다 Detail과 RecordingMeeting까지 모두 컴파일
-> 기능 간 불필요한 의존성 발생

## AppView & AppFeature 만들고 NavigationStack 선언하기

이렇게 뼈대를 만드는 작업을 scaffolding 이라고 한다.
```swift
import ComposableArchitecture
import SwiftUI 

struct AppFeature: Reducer {// NavigationStack에서 사용될 Feature들 선언
  struct State {

  }
  enum Action {

  }

  var body: some ReducerOf<Self> {
    Reduce { state, action in
      switch action {

      }
    }
  }
}

struct AppView: View {
  var body: some View {
    NavigationStack {

    }
  }
}

#Preview {
  AppView()
}
```


## Scope로 상위 Reducer와 하위 Reducer 연결하기

```swift
struct AppFeature: Reducer {
  struct State {
    var standupsList = StandupsListFeature.State()
  }
  enum Action {
    case standupsList(StandupsListFeature.Action)
  }
	var body: some ReducerOf<Self> {
	  Scope(
	    state: \.standupsList,
	    action: /Action.standupsList
	  ) {
	    StandupsListFeature()
	  }
	
	  Reduce { state, action in
	    switch action {
	    case .standupsList:
	      return .none
	    }
	  }
	}
}
```
AppFeature는 전체 앱을 대표하는 도메인이고, StandupsListFeature는 그 안에서 일부 역할을 담당하는 하위 도메인이다.
[[Scope]]를 이용해 부모 상태에서 자식 상태만 > 잘라내어 < 해당 자식 reducer에 넘긴다.
```swift
Scope(
  state: \.standupsList,             // AppFeature.State에서 일부 잘라냄
  action: /Action.standupsList       // AppFeature.Action 중 해당 케이스만 추림
) {
  StandupsListFeature()              // 실제 자식 reducer를 실행
}
```
## 스택 관리하기
### StackState: 스택의 상태 표현
```swift
struct AppFeature: Reducer {
  struct State {
    var path = StackState<Path.State>() // 내비게이션 스택의 현재 상태 추적
    ...
  }
}
```

### Path 도메인 정의
```swift
struct Path: Reducer {
  enum State { // enum으로 정의된 여러 목적지 화면들의 상태
    case detail(StandupDetailFeature.State)
  }
  enum Action {
    case detail(StandupDetailFeature.Action)
  }
  var body: some ReducerOf<Self> {
	Scope(state: /State.detail, action: /Action.detail) {
      StandupDetailFeature()
    }
  }
}
```
-이 Path reducer는 **내비게이션 스택에 올라갈 수 있는 모든 화면을 하나로 관리**한다.

### [[StackAction]]으로 스택 액션 처리하기
```swift
enum Action {
  case path(StackAction<Path.State, Path.Action>)
  ...
}
```
➡️ Navigation Stack에 발생하는 모든 일(pushed, popped, child action)을 AppFeature 액션으로 올려보내는 통로를 만든다.

Path.State, Path.Action를 받은 이유?
- Path.State는 화면에 어떤 걸 보여줄지 나타냄
- Path.Action은 그 화면에서 어떤 일이 벌어졌는지 나타냄

### Reducer에서 .forEach로 Path 통합하기
```swift
var body: some ReducerOf<Self> {
  Reduce { state, action in
    switch action {
    case .path:
      return .none
    ...
    }
  }
  .forEach(\.path, action: /Action.path) { // \.path가 property가 아니기 때문에, \.$path가 아닌 \.path
    Path()
  }
}
```
- StackState에 들어있는 각 화면에 대해 해당하는 reducer를 호출한다.
- IdentifiedArray 같은 리스트 상태에 대해 **여러 개의 자식 reducer**를 실행한다.

## AppFeature가 뭐하는 앤데 그래서
왜 이렇게 이해가 안 되는 걸까... 어렵다 그래서 AppFeature는 뭐하는 애인 걸까...
AppFeature는 앱 전체의 최상위 도메인이다!
쉽게 말해서
> 앱 전체의 상태와 기능을 하나로 통합해서 관리하는 중앙 본부

### 왜 필요하지?
- NavigationStack은 앱의 여러 화면을 스택처럼 쌓고 관리해야 한다.
- TCA에서는 그걸 상태 기반으로 선언적으로 구성하기 때문에 하나의 중심 reducer인 AppFeature 안에 모아두는 게 편하다!

## AppFeature.State 요소들의 용도
```swift
struct State {
  var standupsList = StandupsListFeature.State() // 루트 뷰 (첫 화면)
  var path = StackState<Path.State>()            // 스택 (push/present)되는 화면들
}
```

1. standupsList
	- 루트 화면, standups 목록 화면 상태
	- NavigationStack에서 가장 아래에 깔려 있는 첫 번째 화면
	- 절대 pop되지 않음 (never be popped off)
2. path
	- NavigationStack에 push되는 상세 화면들을 담는 Stack
	- ex) .detail, recordingMeeting, .meetingHistory 등
	- StackState<Path.State> 형태로, 여러 타입의 화면을 enum으로 표현

정리하자면
```
AppFeature
├── standupsList → 루트 뷰 (첫 화면)
└── path         → 스택(push/present)되는 화면들

NavigationStackStore {
  root: StandupsListView(store: .scope(… standupsList …))
  destination: { state in
    switch state {
      case .detail: StandupDetailView
      case .record: RecordMeetingView
    }
  }
}
```
그렇다 ...

## [[X NavigationStackStore]]와 [[X CaseLet]]으로 화면 관리하기
화면 전환도 상태로 표현하고 관리하고자 `NavigationStackStore`를 사용했다.
```swift
struct AppView: View {
    
    let store: StoreOf<AppFeature>
    
  var body: some View {
    NavigationStackStore(
        self.store.scope(state: \.path, action: { .path($0) }) // 루트뷰 (항상 존재)
    ) {
        StandupsListView(
            store: self.store.scope(
                state: \.standupsList,
                action: { .standupsList($0) }
            )
        )
    } destination: { state in
        switch state { // StackState에 푸시된 상태에 따라 뷰를 보여줌
        case .detail:
            CaseLet( 
            // enum으로 된 상태와 액션에서 특정 case만 추출하여
            // 자식 뷰에 넘길 store를 자동 생성
                /AppFeature.Path.State.detail,
                 action: AppFeature.Path.Action.detail,
                 then: StandupDetailView.init(store: )
            )
        }
    }
  }
}
```

## 결합도를 높이는 NavigationLink?
이제 링크를 눌렀을 때, 해당 화면으로 연결해야 한다. 이때 쉽게 떠올리는 방법은 바로 NavigationLink?겠지?
하지만 NavigationLink는 결합도를 높인다고 한다.
왜?
```swift
NavigationLink(
  destination: StandupDetailView(...),
  label: { Text("Go to Detail") }
)
```
예시를 확인해보자. 이렇게 NavigationLink를 사용할 경우,

**StandupDetailView의 정의를 컴파일 시점에 반드시 알아야 한다.**
➡️ 즉, StandupsListView는 StandupDetailView에 **직접적으로 의존**!

이게 바로 모듈 간 **결합도 증가**다. (의존하지 않아도 되는 View를 강제로 의존)

TCA는 그대신
```swift
NavigationStackStore(
  self.store.scope(state: \.path, action: { .path($0) })
) {
  StandupsListView(...)
} destination: { state in
  switch state {
  case .detail:
    StandupDetailView(store: ...)
  }
}
```

```swift
NavigationLink(
  state: AppFeature.Path.State.detail(
    StandupDetailFeature.State(standup: standup)
  )
) {
  CardView(standup: standup)
}
```
이렇게 > State < 만을 통해서 navigation을 관리한다.

- NavigationStackStore는 **그저 어떤 상태를 보여줄지를 알기만** 하면 된다.
- 여기서의 navigation은 "이 상태가 있으면 이 View를 띄운다"는 **상태 기반 선언**이다!
➡️ **View, Reducer, Action, Dependency 전부가 필요하지 않고**, **그 View에 해당하는 상태(state)**만 있으면 된당.

## Deep Linking
기존 SwiftUI 개발하면서 불편했던 점이, Navigation으로 연결된 화면을 접근하려면 일일이 화면을 push해줘야 한다. 하지만, TCA에서는 상태를 통한 UI 생성이 가능하기 때문에 deep linking을 사용할 수 있다.

```swift
var editedStandup = Standup.mock
let _ = editedStandup.title += " Morning Sync"
AppView(
  store: Store(
    initialState: AppFeature.State(
      path: StackState([
        .detail(
          StandupDetailFeature.State(
            editStandup: StandupFormFeature.State(
              focus: .attendee(editedStandup.attendees[3].id)
              standup: editedStandup
            ),
            standup: .mock
          )
        )
      ]),
      standupsList: StandupsListFeature.State(
        standups: [.mock]
      )
    )
  ) {
    AppFeature()
  }
)
```
- StackState는 navigation stack을 모델링.
- 그 안에 .detail(...) 상태가 들어 있음 → 앱 실행 시 곧바로 **상세 화면으로 push**됨.
- 그 안에 또 editStandup이 nil이 아니라 **편집 화면이 활성화된 상태**
- 그 편집 화면에서는 .attendee(id) 포커스까지 지정됨

앱 실행 시 바로:
- “Design Morning Sync”라는 제목의 standup이
- 편집 상태로 열리고
- 네 번째 참석자의 이름 텍스트필드에 포커스가 가 있음

**deep linking이란?**
- 앱을 특정 화면 _또는 그 상태_로 직접 진입시키는 것
- 웹에서 myapp://standups/42/edit 같은 URL로 특정 위치로 이동하는 걸 의미  

TCA에서는 URL이 아니라도, **상태를 직접 구성하면 그게 곧 deep link**
> 🔄 “**탭 없이**, 사용자의 행동 없이, 상태만 설정하면 UI가 그에 맞게 바로 따라감”

## Delegate: 하위 도메인의 수정사항을 상위 도메인에게 최신화하기

```swift
case let .path(.element(id: id, action: .detail(.delegate(action)))):      
    switch action { // 자식이 변경 사실을 알려주면, standupslist배열을 갱신해서 UI 최신화
        case let .standupUpdated(standup):
              state.standupsList.standups[id: standup.id] = standup
              return .none
          }
          
//           (1) 자식뷰 내부에서 saveStandupButtonTapped 버튼을 눌렀을 때 -> 즉시 반영
case let .path(.element(id: id, action: .detail(.saveStandupButtonTapped))):
    guard case let .some(.detail(detailState)) = state.path[id: id]
    else { return .none }
    state.standupsList.standups[id: detailState.standup.id] = detailState.standup
    return .none
          
//  (2) 뷰가 pop(뒤로 가기)될 때 -> 애니메이션이 끝나야 반영
//   자식 화면의 수정 내용을 부모가 최신화하는 방법 중 하나인 pop시 동기화 방식
case let .path(.popFrom(id: id)): // 사용자가 뒤로가기 / 스와이프해서 뷰 pop, stack에서 pop된 id를 전달 받음
    guard case let .some(.detail(detailState)) = state.path[id: id] // pop 되기 전 상태를 stack에서 찾아냄. 이 상태가 detail인 경우만 작업 진행
    else { return .none }
    state.standupsList.standups[id: detailState.standup.id] = detailState.standup // 상세화면에서 편집된 최신 데이터를 리스트에 반영. standup.id로 기존 아이템을 찾아 덮어씌움.
```



```swift
//
//  AppView.swift
//  Standups_scrumdinger
//
//  Created by Soop on 5/13/25.
//

import ComposableArchitecture
import SwiftUI

struct AppFeature: Reducer {
  struct State {
      var path = StackState<Path.State>()   // 내비게이션
      var standupsList = StandupsListFeature.State()    // 기본 화면
  }
    
  enum Action {
      case path(StackAction<Path.State, Path.Action>)
      case standupsList(StandupsListFeature.Action)
  }
    
    struct Path: Reducer {  // 화면 전환
        enum State {
            case detail(StandupDetailFeature.State)
        }
        
        enum Action {
            case detail(StandupDetailFeature.Action)
            
        }
        
        var body: some ReducerOf<Self> {
            Scope(state: /State.detail, action: /Action.detail) {
                StandupDetailFeature()
            }
        }
    }
    
  var body: some ReducerOf<Self> {
      Scope(state: \.standupsList, action: /Action.standupsList) {
          StandupsListFeature()
      }
    Reduce { state, action in
      switch action {
      case let .path(.element(id: id, action: .detail(.delegate(action)))):
          
          switch action { // 자식이 변경 사실을 알려주면, standupslist배열을 갱신해서 UI 최신화
          case let .standupUpdated(standup):
              state.standupsList.standups[id: standup.id] = standup
              return .none
          }
          
          
      case .path:
          return .none
        
      case .standupsList:
          return .none
      }
    }
    .forEach(\.path, action: /Action.path) {
        Path()
    }
  }
}

struct AppView: View {
    
    let store: StoreOf<AppFeature>
    
  var body: some View {
    NavigationStackStore(
        self.store.scope(state: \.path, action: { .path($0) })
    ) {
        StandupsListView(
            store: self.store.scope(
                state: \.standupsList,
                action: { .standupsList($0) }
            )
        )
    } destination: { state in
        switch state {
        case .detail:
            CaseLet(
                /AppFeature.Path.State.detail,
                 action: AppFeature.Path.Action.detail,
                 then: StandupDetailView.init(store: )
            )
        }
    }
  }
}

#Preview {
    AppView(
        store: Store(
            initialState: AppFeature.State(
                standupsList: StandupsListFeature.State(standups: [.mock])
            )
        ) {
            AppFeature()
        }
    )
}

```