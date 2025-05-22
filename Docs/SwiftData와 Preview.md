> SwiftData를 처음 사용하면서 느꼈던 여러 불편함 중 하나가 바로 Preview에 SwiftData 적용하는 것이었다.

# GQ
Preview에 SwiftData를 어떻게 적용할 수 있을까?

# GA
Apple Developer 공식문서에서 제공하는 SwiftDataAnimals를 뜯어보자!

SwiftData에 대해서 C2 때 공부했었는데, 따로 문서 정리는 하지 않았다... 이후에 할 예정

# 1. 파일 구조
![[Screenshot 2025-05-23 at 12.12.51 AM.png]]
*(과연 캡처를 이렇게 하는 게 올바른 행동일까)*

폴더링은 Models, Views, PreviewHelper, 그리고 SampleData로 총 4개로 나뉘었다.
그 중에서 눈에 띄는 건 PreviewHelper랑 SampleData였다.

# 그래서 어떻게 프리뷰에 스데를 썼을까...?

![[Screenshot 2025-05-23 at 1.42.44 AM.png]]
어떻게 했냐고 물었다.

의심이 가는 파일들을 먼저 열어보았다. (바로 ModelContainerPreview와 Preview+ModelContainer)
![[Screenshot 2025-05-23 at 12.31.20 AM.png]]
주석... 너무 길어서 당황햇다 그치만 다시 또 탐색해봤다... *자세히 설명해줘서 감사합니다*

## ModelContainerPreview<Content: View>
먼저 ModelContainerPreview를 살펴봤다. 
Content와 ModelContainer가 프로퍼티인 struct였고, 생성자가 3개나 있었다.

```swift
import SwiftUI
import SwiftData

struct ModelContainerPreview<Content: View>: View {
    var content: () -> Content
    let container: ModelContainer

    init(@ViewBuilder content: @escaping () -> Content, modelContainer: @escaping () throws -> ModelContainer) {
        self.content = content
        do {
            self.container = try MainActor.assumeIsolated(modelContainer)
        } catch {
            fatalError("Failed to create the model container: \(error.localizedDescription)")
        }
    }

    init(_ modelContainer: @escaping () throws -> ModelContainer, @ViewBuilder content: @escaping () -> Content) {
        self.init(content: content, modelContainer: modelContainer)
    }

    var body: some View {
        content()
            .modelContainer(container)
    }
}

```

더 자세히 살펴보겠다!

---
## 프로퍼티
```swift
var content: () -> Content
let container: ModelContainer
```
- content: 실제로 Preview에서 렌더링할 SwiftUI 뷰
- container: 미리 생성된 SwiftData 컨테이너

---

## 생성자 1 - 일반

> Creates an instance of the model container preview.
> This view creates the model container before displaying the preview content. 
> The view is intended for use in previews only.

요약하자면, model container를 사용하는 preview를 생성하는 struct이고, 미리 모델컨테이너를 생성해서 SwiftUI뷰를 프리뷰에서 보여줄 때 사용한다. 그리고 .modelContainer(container)를 통해 SwiftData 컨텍스트를 뷰에 주입한다.

```swift
init(@ViewBuilder content: @escaping () -> Content, modelContainer: @escaping () throws -> ModelContainer)
```
- 파라미터로 content 뷰와 modelContainer 생성 클로저를 받음. 이떄 [[@ViewBuilder]]를 통해 여러 개의 뷰를 받도록 함.
- MainActor.assumeIsolated를 통해 메인 스레드에서 SwiftData 컨테이너를 안전하게 초기화
- 실패 시 fatalError로 Preview 실패

---

## 생성자 2 - 인자 순서 반대
```swift
init(_ modelContainer: @escaping () throws -> ModelContainer, @ViewBuilder content: @escaping () -> Content)
```
위의 생성자의 인자 순서가 반대이다. 왜 반대로 했냐몬 Preview에서 더 자연스러운 순서를 제공하려고 이렇게 했다고 함...

---
## body
```swift
var body: some View {
    content()
        .modelContainer(container)
}
```

그렇게 입력 받은 뷰와 컨테이너를 통해, `.modelContainer(container)`로 연결

---
# 사용 예시
```swift
#Preview {
    ModelContainerPreview {
        AnimalEditor(animal: nil)
            .environment(NavigationContext())
        } modelContainer: {
            let schema = Schema([AnimalCategory.self, Animal.self])
            let configuration = ModelConfiguration(isStoredInMemoryOnly: true)
            let container = try ModelContainer(for: schema, configurations: [configuration])
            Task { @MainActor in
                AnimalCategory.insertSampleData(modelContext: container.mainContext)
            }
        return container
    }
}
```

이렇게 ModelContainerPreview에 보여주고 싶은 뷰와 modelContainer를 입력하면 된다!
그렇지만 프로젝트에서는 일일이 저렇게 작성하지 않고 sample model container를 만들어서 주입했다

바로 Preview+ModelContainer 파일을 보면

```swift
import SwiftData

extension ModelContainer {
    static var sample: () throws -> ModelContainer = {
        let schema = Schema([AnimalCategory.self, Animal.self])
        let configuration = ModelConfiguration(isStoredInMemoryOnly: true)
        let container = try ModelContainer(for: schema, configurations: [configuration])
        Task { @MainActor in
            AnimalCategory.insertSampleData(modelContext: container.mainContext)
        }
        return container
    }
}

```
이렇게 ModelContainer를 extension하여 example을 추가한 걸 볼 수 있다.
example을 정의해서 사용하면
```swift
#Preview {
    ModelContainerPreview(ModelContainer.sample) {
        ThreeColumnContentView()
            .environment(NavigationContext())
    }
}
```

이렇게 일일이 container를 작성하지 않아도 깔끔하게 주입할 수 있당 ~