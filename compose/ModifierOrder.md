# (Compose) Modifier Order

Modifier을 사용하면 다음과 같은 종류의 작업을 할 수 있습니다.
- 컴포저블의 크기, 레이아웃, 동작 및 모양 변경
- 접근성 라벨과 같은 정보 추가
- 사용자 입력 처리
- 요소를 클릭 가능, 스크롤 가능, 드래그 가능 또는 확대/축소 가능하게 만드는 높은 수준의 상호작용 추가


Modifier은 표준 Kotlin object입니다. Modifier 클래스 함수 중 하나를 호출하여 수정자를 만듭니다.
```kotlin
import androidx.compose.ui.Modifier

@Composable
private fun Greeting(name: String) {
    Column(modifier = Modifier.padding(24.dp)) {
        Text(text = "Hello,")
        Text(text = name)
    }
}
```

![](https://developer.android.com/static/images/jetpack/compose/modifier-1-modifier.png?hl=ko)

---

## Modifier Order
수정자 함수의 순서는 중요합니다. 각 함수는 이전 함수에서 반환한 Modifier를 변경하므로 순서는 최종 결과에 영향을 줍니다.
```kotlin
@Composable
fun ArtistCard(/*...*/) {
    val padding = 16.dp
    Column(
        Modifier
            .clickable(onClick = onClick)
            .padding(padding)
            .fillMaxWidth()
    ) {
        // rest of the implementation
    }
}
```

![](https://developer.android.com/static/images/jetpack/compose/layout-padding-clickable.gif?hl=ko)
위의 코드에서는 padding 수정자가 clickable 수정자 뒤에 적용되었기 때문에 주변 패딩을 포함하여 전체 영역을 클릭할 수 있습니다. 수정자 순서가 뒤집히면 다음과 같이 padding으로 추가된 공간은 사용자 입력에 반응하지 않습니다.


```kotlin
@Composable
fun ArtistCard(/*...*/) {
    val padding = 16.dp
    Column(
        Modifier
            .padding(padding)
            .clickable(onClick = onClick)
            .fillMaxWidth()
    ) {
        // rest of the implementation
    }
}
```

![](https://developer.android.com/static/images/jetpack/compose/layout-padding-not-clickable.gif?hl=ko)
Modifier의 순서를 부여하기 위한 코드가 어떻게 구성되어 있는지 확인하기 위해 Modifier.padding 코드를 살펴보면 다음과 같습니다.

```kotlin
@Stable
fun Modifier.padding(all: Dp) = this then PaddingElement(
    start = all,
    top = all,
    end = all,
    bottom = all,
    rtlAware = true,
    inspectorInfo = {
        name = "padding"
        value = all
    }
)
```

여기서 주목할 점은 <b>then</b>이라는 infix 함수를 사용한다는 점입니다.

```kotlin
infix fun then(other: Modifier): Modifier =
    if (other === Modifier) this else CombinedModifier(this, other)
```

other가 Modifer 기본 객체이면 현재 Modifier를 반환하고 그렇지 않을 경우 현재 Modifier와 other Modifier를 조합하는 <b>CombinedModifier</b>를 반환합니다.


----

## 참고
- https://developer.android.com/jetpack/compose/modifiers?hl=ko

- https://www.charlezz.com/?p=45589

- https://seungmin.dev/2023/05/03/Jetpack-Compose-Modifier-%EC%8B%AC%EC%B8%B5-%EB%B6%84%EC%84%9D.html

- https://www.youtube.com/watch?v=OeC5jMV342A