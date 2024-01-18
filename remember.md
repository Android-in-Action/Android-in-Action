블로그 링크<br>
<https://blog.naver.com/pck4949/223324681367>


# (Compose) MutableState, State, remember


<b>Compose는 앱의 현재 상태를 설명하고  상태가 변경될 때마다 UI 업데이트를 처리합니다.</b>

MutableState, State, remember를 알아보기 전에 컴포즈에서의 상태와 컴포즈에서 자주 사용되는 용어를 정리할 필요가 있었습니다.

## Compose에서의 상태
시간이 지남에 따라 변할 수 있는 값을 의미합니다. 
<br>(Room 데이터베이스부터 클래스 변수까지 모든 항목이 포함)

## 핵심 용어
- 컴포지션: Jetpack Compose가 컴포저블을 실행할 때 빌드한 UI에 대한 설명입니다.

![](https://postfiles.pstatic.net/MjAyNDAxMTZfMjI4/MDAxNzA1Mzg0ODcyMjYz.ECDH1lfNK79Gq4XuyGRGq71AROx9yUdxd0mV8QEa9_0g.fqKQebXi2KjTPx9JbkM3xZS2blROY-gbu17RNmOS35Qg.PNG.pck4949/image.png?type=w773 "https://youtu.be/rmv2ug-wW4U")
- 초기 컴포지션: 처음 컴포저블을 실행하여 생성된 컴포지션

- 리컴포지션(재구성): 데이터가 변경될 때 컴포지션을 업데이트하기 위해 컴포저블을 다시 실행하는 것을 의미합니다.

예를 들어 장바구니 화면으로 이동하면 상태 변화에 영향을 받는 UI의 해당 부분을 다시 실행합니다.

![](https://postfiles.pstatic.net/MjAyNDAxMTZfMjIz/MDAxNzA1Mzg1NTk3NzU1.kYtFQp2SNGEO4voQ-LVyh-YxLfb32ipp_Oxz3x_FwTQg.o8oU_bPb-cEaHuo_pgEvPnPZ5yjjjUZouq8YO0CvG_wg.PNG.pck4949/image.png?type=w773 "https://youtu.be/rmv2ug-wW4U")

UI의 모든 부분은 컴포저블 함수이기 때문에 <b>상태가 변경되면 리컴포지션을 통해 화면에 새 데이터를 표시</b>합니다.

​

또한 Compose는 데이터가 변경된 구성요소만 재구성하고 영향을 받지 않는 구성요소는 건너뛰도록 개별 컴포저블에 필요한 데이터를 확인합니다.

## State, MutableState
Compose에는 상태를 읽는 컴포저블에 대한 리컴포지션(재구성)을 예약하는 <b>상태 추적 시스템</b>이 존재합니다. 이를 통해 변경해야 하는 Composable 함수만 재구성할 수 있습니다.

​

이 작업은 '쓰기'(즉, 상태 변경)뿐만 아니라 상태에 대한 '읽기'도 추적하여 실행됩니다.

​

Compose의 <b>State 및 MutableState 타입</b>을 사용하여 상태를 관찰 가능하게 만들 수 있습니다.


### State
~~~
interface State<out T> {
    val value: T
}
~~~

### MutableState
~~~
interface MutableState<T> : State<T> {
    override var value: T
    operator fun component1(): T
    operator fun component2(): (T) -> Unit
}
~~~
보시다시피 State의 value는 immutable입니다. MutableState는 이름에서 알 수 있듯이 value를 바꿀 수 있고, 이 값을 수정하면 리컴포지션이 일어납니다.

<b>mutableStateOf</b> 함수를 사용해서 MutableState를 만들 수 있습니다. 

~~~
fun <T> mutableStateOf(
    value: T,
    policy: SnapshotMutationPolicy<T> = structuralEqualityPolicy()
): MutableState<T> = createSnapshotMutableState(value, policy)
~~~

~~~
import androidx.compose.runtime.mutableStateOf
...

@Composable
fun Greeting() {
    // Creating a state object during composition without using `remember`
    var expanded = mutableStateOf(false) 
    val extraPadding = if (expanded.value) 48.dp else 0.dp
    ...
}
~~~

하지만 컴포저블 함수 내에서 단순히 mutableStateOf 함수를 사용할 수는 없습니다. 리컴포지션은 언제든지 발생할 수 있기 때문입니다. 이는 상태를 초기화하고 새로운 MutableState를 생성과 함께 false 값을 갖게 될 것입니다.


## remember
리컴포지션이 일어날 때 값을 보존하기 위해선 remember 함수를 사용합니다.

remember는 재구성으로부터 해당 값의 변경을 보호하여 초기화되지 않도록 합니다.

​

remember는 객체를 컴포지션에 저장하고(초기 컴포지션), 저장된 값은 리컴포지션 중에 반환됩니다. remember를 호출한 컴포저블이 컴포지션에서 삭제되면 그 객체를 잊습니다.

​

※ 참고<br>remeber는 재구성 과정 전체에서 상태를 유지하는 데 도움은 되지만 <b>구성 변경 전반에서는 상태가 유지되지 않습니다.</b> 

<b>Activity 또는 프로세스가 다시 생성된 이후에도 상태를 보존하려면 rememberSaveable을 사용해야 합니다.</b>rememberSaveable은 Bundle에 저장할 수 있는 모든 값을 자동으로 저장합니다.

~~~
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
...

@Composable
fun Greeting() {
    var expanded = remember { mutableStateOf(false) }
    val extraPadding = if (expanded.value) 48.dp else 0.dp
    ...
}
~~~

컴포저블 함수는 상태를 자동으로 구독합니다. 상태가 변경되면 이러한 필드를 읽는 컴포저블이 재구성되어 업데이트를 표시합니다.

### MutableState 객체를 선언하는 다른 방법들
- by 사용
by 키워드로 Property delegate를 사용할 수 있습니다. 이렇게 하면 항상 value 속성에 접근하지 않아도 됩니다.
~~~
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.runtime.getValue
...

@Composable
fun Greeting() {
    var expanded by remember { mutableStateOf(false) }
    val extraPadding = if (expanded.value) 48.dp else 0.dp
    ...
}
~~~

- 구조 분해
MutableState의 구조 분해 구문을 사용하면 다음과 같이 사용할 수도 있습니다.

![](https://postfiles.pstatic.net/MjAyNDAxMThfNTcg/MDAxNzA1NTEyMTUwODM4.H5ydVJzaVdGb38JS3364Q5hZoz6bEIhRbDIhBwHsIIAg.fRJR2zR7uz5I593g6GmhAz0jMtZMxRMvTJqMgM50GiAg.PNG.pck4949/image.png?type=w773)
component1( ): MutableState 내부의 값을 나타내는 value를 반환 

componetn2( ): 새로운 값이 들어왔을 때 value 업데이트

### 키가 변경될 경우 계산 기억 다시 트리거
remeber는 MutableState와 함께 자주 사용됩니다.
~~~
var name by remember { mutableStateOf("") }
~~~
일반적으로 remember는 <b>calulation 람다 매개변수</b>를 취합니다.

~~~
@Composable
inline fun <T> remember(crossinline calculation: @DisallowComposableCalls () -> T): T =
    currentComposer.cache(false, calculation)
~~~

~~~
inline fun <T> Composer.cache(invalid: Boolean, block: @DisallowComposableCalls () -> T): T {
    @Suppress("UNCHECKED_CAST")
    return rememberedValue().let {
        if (invalid || it === Composer.Empty) {
            val value = block()
            updateRememberedValue(value)
            value
        } else it
    } as T
}
~~~

<b>remember가 처음 실행되면 calulation 람다를 호출하고 그 결과를 저장합니다. 리컴포지션 중에 remeber는 마지막으로 저장된 값을 반환합니다.</b>

​

remember는 컴포지션을 종료할 때까지 값을 저장합니다. 하지만 <b>캐시된 값을 무효화하는 방법</b>이 있습니다.

remember는 <b>key 또는 keys 매개변수</b>도 취합니다.

~~~
@Composable
inline fun <T> remember(
    key1: Any?,
    crossinline calculation: @DisallowComposableCalls () -> T
): T {
    return currentComposer.cache(currentComposer.changed(key1), calculation)
}

@Composable
inline fun <T> remember(
    key1: Any?,
    key2: Any?,
    crossinline calculation: @DisallowComposableCalls () -> T
): T {
    return currentComposer.cache(
        currentComposer.changed(key1) or currentComposer.changed(key2),
        calculation
    )
}

@Composable
inline fun <T> remember(
    key1: Any?,
    key2: Any?,
    key3: Any?,
    crossinline calculation: @DisallowComposableCalls () -> T
): T {
    return currentComposer.cache(
        currentComposer.changed(key1) or
            currentComposer.changed(key2) or
            currentComposer.changed(key3),
        calculation
    )
}

@Composable
inline fun <T> remember(
    vararg keys: Any?,
    crossinline calculation: @DisallowComposableCalls () -> T
): T {
    var invalid = false
    for (key in keys) invalid = invalid or currentComposer.changed(key)
    return currentComposer.cache(invalid, calculation)
}
~~~

이러한 키 중 하나라도 변경될 경우 다음 번에 함수가 재구성될 때 remember는 캐시를 무효화하고 계산 람다 블록을 다시 실행합니다. 

~~~
  override fun changed(value: Any?): Boolean {
        return if (nextSlot() != value) {
            updateValue(value)
            true
        } else {
            false
        }
    }
~~~

또한 상태 캐싱 외에도 <b>remember를 사용하여 초기화하거나 계산하는 데 비용이 많이 드는 객체 또는 작업의 결과를 컴포지션중에 저장할 수도 있습니다. </b>리컴포지션은 매우 자주 실행될 수 있기 때문에 매 리컴포지션마다 무거운 작업을 반복하지 않는 것이 좋습니다.


## 참고
Jetpack Compose의 상태
<https://developer.android.com/codelabs/jetpack-compose-state?hl=ko&continue=https%3A%2F%2Fdeveloper.android.com%2Fcourses%2Fpathways%2Fcompose%3Fhl%3Dko%23codelab-https%3A%2F%2Fdeveloper.android.com%2Fcodelabs%2Fjetpack-compose-state#4>

Jetpack Compose 기초
<https://developer.android.com/codelabs/jetpack-compose-basics?hl=ko&continue=https%3A%2F%2Fdeveloper.android.com%2Fcourses%2Fpathways%2Fcompose%3Fhl%3Dko%23codelab-https%3A%2F%2Fdeveloper.android.com%2Fcodelabs%2Fjetpack-compose-basics#6>

Compose state of mind
<https://youtu.be/rmv2ug-wW4U>

remember 도장깨기
<https://blog.onebone.me/post/jetpack-compose-remember/>

Delegated properties
<https://kotlinlang.org/docs/delegated-properties.html>

## 나중에 볼 것들
Under the hood of Jetpack Compose
<https://medium.com/androiddevelopers/under-the-hood-of-jetpack-compose-part-2-of-2-37b2c20c6cdd>

Compose 생명주기
<https://developer.android.com/jetpack/compose/lifecycle?hl=ko>

Compose 단계
<https://developer.android.com/jetpack/compose/phases?hl=ko#phase3-drawing>

Scoped recomposition in Jetpack Compose
<https://dev.to/zachklipp/scoped-recomposition-jetpack-compose-what-happens-when-state-changes-l78>
