https://developer.apple.com/documentation/swiftui/viewbuilder

> A custom parameter attribute that constructs views from closures.

```swift
@ViewBuilder content: @escaping () -> Content
```
SwiftUI 전용 특수 속성. 여러 개의 뷰를 선언형으로 구성할 수 있게 한다.
Swift의 결과 빌더(result builder) 중 하나다.

```swift
ModelContainerPreview {
    Text("Hello")
    Image(systemName: "star")
}
```
이걸 사용하면, 프리뷰에서 한 번에 여러 뷰를 리턴할 수 있다.

공식문서의 예시를 확인해보면
```swift
func contextMenu<MenuItems: View>( @ViewBuilder menuItems: () -> MenuItems ) -> some View
```

```swift
myView.contextMenu {
    Text("Cut")
    Text("Copy")
    Text("Paste")
    if isSymbol {
        Text("Jump to Definition")
    }
}
```

이런 식으로 여러가지 뷰를 return 키워드 없이 선언형으로 작성이 가능하다!