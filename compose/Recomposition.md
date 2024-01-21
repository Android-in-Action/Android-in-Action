# Recomposition & Stability

# 1. Recompostion 이란?

- Compose에서 UI는 함수로 표현되며, 데이터가 변경될 때마다 Recompostion이 됩니다.
- Recompostion은 Compose에서 UI가 변경될 때 발생하는 프로세스 입니다.
- 변경사항이 감지되면 Compose는 해당 부분을 다시 그리고 업데이트합니다.

→ 리컴포지션은 Composable 함수의 상태가 변경될 때 Compsable 함수를 다시 호출하여, UI를 업데이트 하는 것

# 2. Recomposition & Performance

> 전체 UI 트리를 재구성하는 것은 컴퓨팅 성능과 배터리 수명을 사용하므로 계산 비용이 많이 들 수 있습니다. *Compose는 SmartRecpomposition을* 통해 이 문제를 해결합니다 .
>

공식 문서에 따르면 UI트리를 재구성하는 것은 비용이 많이 들 수 있다고 나와 있습니다. 아래의 요인에 따라 빈번한 리컴포지션 발생 시, 성능저하 이슈가 발생할 수도 있습니다.

1. UI의 복잡성
    - 복잡한 UI에서 많은 컴포저블이 동시에 리컴포지션이 빈번하게 일어나면, 레이아웃 계산과 그리기 작업에 더 많은 자원이 소비될 수 있습니다.
    - 애니메이션 사용
        - 로띠 애니메이션, 고용량 그래픽 이미지, 지속적인 루프 연산 등등
2. 데이터 모델의 규모
    - 대량의 데이터가 자주 변경되면, 리컴포지션 소요 시간이 늘어날 수 있습니다.
3. 기기 성능
    - 성능이 낮은 기기에서는 빈번한 리컴포지션은 더 큰 영향을 미칠 수 있습니다.

경험적으로 일반적인 UI에서 리컴포지션이 몇 번 더 발생하더라도, 퍼포먼스가 떨어진다는 것은 체감하지 못했습니다. 하지만 그럼에도 최적화하는 방법에 대해서 생각해 볼 필요는 있을 것 같습니다. 그러면 불필요한
리컴포지션을 최소화하기 위해 우리가 할 수 있는 것은 무엇일까요? 컴포저블 파라미터의 안정성을 확인하는 것입니다. 불안정한 상태가 많이 포함되어 있으면, 성능 및 기타 문제가 발생할 수 있습니다.

# 3. Stability

컴포즈는 타입을 안정적인 것 또는 불안정적인 것으로 간주합니다.

- Immutable하거나 컴포즈가 리컴포지션 시에 값이 변경되었는 지 알 수 있는 경우 안정적 타입입니다.
- 컴포즈가 리컴포지션 간 값이 변경되었는지 여부를 알 수 없는 경우는 불안정한 타입입니다.

컴포즈는 컴포저블 파라미터의 안정성을 사용하여, 리컴포지션을 스킵할 수 있을 지 결정합니다.

## Type

---

**Stable**

- 모든 원시 값 타입, 문자열, 함수 타입 파라미터는 Stable합니다.
- 인스턴스는 모든 프로퍼티가 Stable 해야 안정적인 것으로 간주합니다.

```kotlin
data class User(
    val id: String
    val name: String
)
```

- equals 결과가 동일한 두 인스턴스의 경우 안정적인 것으로 간주합니다.

```kotlin
User("1", "XXX") == User("1", "XXX")
```

- @Stable 어노테이션을 사용하면, 컴포즈 컴파일러가 안정적인 값으로 인식하게 할 수 있습니다.
- 클래스에 어노테이션을 달면 컴파일러가 클래스에
  관해 [[추론](https://developer.android.com/jetpack/compose/performance/stability?hl=ko)](https://developer.android.com/jetpack/compose/performance/stability?hl=ko)
  하는 내용이 재정의됩니다. [Kotlin의 `!!`](https://kotlinlang.org/docs/null-safety.html#the-operator) 연산자와 유사합니다.
- 컴포저블의 모든 파라미터가 Stable하고, 변경사항이 없다면, 리컴포지션을 스킵할 수 있습니다.

**Unstable**

- 컴포저블에 불안정한 매개변수가 있는 경우 Compose는 구성요소의 상위 요소를 재구성할 때 항상 컴포저블을 재구성합니다.
- Collection
    - Compose는 `List, Set`, `Map`와 같은 컬렉션 클래스를 항상 불안정한 것으로 간주합니다. 이는 해당 파일이 불변성이라고 보장할 수 없기 때문입니다. (MutableList)
    -
    대신 [https://github.com/Kotlin/kotlinx.collections.immutable을](https://github.com/Kotlin/kotlinx.collections.immutable%EC%9D%84)
    사용하거나, 클래스에 `@Immutable` 또는 `@Stable`로 어노테이션을 달아서 안정적인 타입으로 만들 수 있습니다.
    - 따라서 컬렉션 클래스를 사용할 때는 @Immutable 또는 @Stable 어노테이션을 사용해야합니다.

    ```kotlin
    data class Snack(
      val id: Long,
      val name: String,
      val imageUrl: String,
      val price: Long,
      val tagline: String = "",
      val tags: Set<String> = emptySet()
    )

    unstable class Snack {
       stable val id: Long
       stable val name: String
       stable val imageUrl: String
       stable val price: Long
       stable val tagline: String
       unstable val tags: Set<String>
       <runtime stability> = Unstable
    }ƒ
    @Stable
    data class Snack(
      val id: Long,
      val name: String,
      val imageUrl: String,
      val price: Long,
      val tagline: String = "",
      val tags: Set<String> = emptySet()
    ) //snackList는 여전히 unstable


    @Composable
    private fun HighlightedSnacks(
        snacks: ImmutableList<Snack>,
    )

    @Immutable
    data class SnackCollection(
       val snacks: List<Snack>
    )


    @Composable
    private fun HighlightedSnacks(
        index: Int,
    		snack: Snack (stable)
    -   snacks: List<Snack>, (unstable)
    +   snacks: ImmutableList<Snack>, (stable)
        onSnackClick: (Long) -> Unit,
        modifier: Modifier = Modifier
    )

    //or

    @Composable
    private fun HighlightedSnacks(
        index: Int,
        snacks: SnackCollection,
        onSnackClick: (Long) -> Unit,
        modifier: Modifier = Modifier
    )

    //https://developer.android.com/jetpack/compose/performance/stability/fix?hl=ko#immutable-collections
    ```

    ```kotlin

    ```

- Mutable Object

    ```kotlin
    data class Contact(
    	 var name: String,
    	 var number: String
    )
    ```

    - 클래스 내에 변수가 포함되어 있으면, 컴포즈는 이를 불안정하다고 간주합니다.
    - 불안정한 클래스는 속성이 변경되어도 컴포즈가 인식하지 못합니다. 이는 Compose가
      Compose [[상태 객체](https://developer.android.com/reference/kotlin/androidx/compose/runtime/MutableState?hl=ko)](https://developer.android.com/reference/kotlin/androidx/compose/runtime/MutableState?hl=ko)
      의 변경사항만 추적하기 때문입니다.
    - 컴포저블에 Unstable 객체가 포함되어 있다면, 항상 리컴포지션이 발생합니다.
- Painter
    - Painter 는 불안정한 타입입니다. Painter를 파라미터로 갖고 있는 컴포저블은 상위 컴포저블이 이 리컴포지션 될 때, 항상 같이 리컴포지션 됩니다.
    - 따라서 `Painter`를 매개변수로 전달하는 대신 URL 또는 드로어블 리소스 ID를 매개변수로서 컴포저블에 전달하는 것이 좋습니다.

    ```kotlin
    @Composable
    fun UnstableImage(painter: Painter) {
    	//항상 리컴포지션이 발생함.
    }

    @Composable
    fun MyImage(url: String) {

    }

    파라미터로 url 또는 res 값을 받는 것을 권장
    https://developer.android.com/jetpack/compose/graphics/images/optimization?hl=ko#pass-url
    ```

# 4. 요약

- 컴포저블의 매개변수 타입이 불안정하거나 변경될 때 리컴포지션된다.
- 상태가 변경되지 않았는데, 리컴포지션이 발생한다면?
    - 불안정한 타입의 상태를 체크하고, 안정적인 타입으로 만들자
- Unstable 타입을 가지고 있다면, 무조건 리컴포지션된다.
    - 추적이 불가능해서 상태가 바뀌었는지 안바뀌었는지 모르기 때문
- Stable 타입만 가지고 있다면, 상태 변경 시에만 리컴포지션됩니다.
    - 추적이 가능한 상태이므로
- immutable 은 상태가 변하지 않으므로 recomposition에서 제외된다.
- Skippable 컴포저블을 만들기 위해서 파라미터의 Stability를 체크하자
    - 컴포저블을 최대한 작고 구체적으로 만드는 것이 좋ㄷㅏ.

https://funny-turn-e30.notion.site/Recomposition-Stability-580a35170b534819a721d688eaa52196

참고 자료
https://chrisbanes.me/posts/composable-metrics/
https://medium.com/androiddevelopers/jetpack-compose-stability-explained-79c10db270c8
https://medium.com/@laco2951/jetpack-compose-stable%EC%9D%B4-%EC%9E%AC%EA%B5%AC%EC%84%B1%EC%97%90-%EC%96%B4%EB%96%A4-%EC%98%81%ED%96%A5%EC%9D%84-%EC%A3%BC%EB%8A%94%EC%A7%80-%EC%A0%95%EB%A6%AC-7169c9ab8615