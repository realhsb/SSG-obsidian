`NavigationStack`은 앱 루트에 설치해야 한다! 왜? 특성 feature에 종속되는 것을 피해 다른 feature를 독립적으로 컴파일한다.
# TCA의 핵심 개념 "독립된 기능"
TCA에서는 앱을 여러 작은 기능 단위로 나누고, 각 기능을 State, Action, Reducer, View로 묶어서 독립적으로 테스트하고 개발할 수 있는 걸 중요하게 여긴다.

예를 들어,
- `StandupsListFeature`: 스탠드업 목록 보여주는 기능
- `StandupDetailFeature`: 선택된 스탠드업에 대한 세부 정보
- `RecordMeetingFeature`: 회의 녹음
이 각각은 별도의 모듈 또는 파일로 나뉘고, 이론적으로는 **각각 따로 컴파일하고 실행 가능**해야 이상적인 구조이다.

# Swift의 빌드 시스템
Swift는 어떤 파일을 수정하면 그 파일에 직접적으로 의존하는 다른 파일들까지 재컴파일한다.
어떤 기능이 다른 기능을 직접 참조하고 있다면, 둘은 컴파일 타이밍상 강하게 묶여 있는 것이다.

## 문제가 있는 구조
```swift
// StandupsListView.swift 내부에
NavigationStack {
  // 여기서 detailFeature와 recordMeetingFeature로 내비게이션함
}
```
이 경우
- `StandupsListView`는 `DetailView`와 `RecordMeetingView`를 직접 사용한다.
- Swift 입장 -> "이 뷰를 바꾸면 연결된 다른 뷰들도 다시 컴파일 해야겠네?" 하고 판단한다.
- 따라서, `StandupsListView`를 수정할 때마다 `DetailView`, `RecordingMeetingView`까지 모두 재컴파일!
➡️ 작업 중이 기능만 고쳐도 전체 기능이 재컴파일 -> 개발 효율 하락 📉

## 이상적인 구조
NavigationStack을 바깥에 두자!
```swift
// AppView.swift (루트)
NavigationStack {
  // 이 아래에서 어떤 feature가 보여질지는 State로 결정
  // 예: StandupsList → Detail → RecordMeeting
}
```
- AppView는 내비게이션을 담당하지만, 실제 각각 기능은 자기 뷰 안에서만 동작한다.
- StandupsList와 Detail은 서로 독립된 기능을 유지
- StandupsList만 고치면 그것만 컴파일하면 된다!