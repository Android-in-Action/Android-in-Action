# LaunchedEffect

## Side effect란?
Side effect는 Compose에서만 사용되는 용어가 아닌 프로그래밍 용어로 여기저기서 자주 쓰이는 것 같습니다.

Side effect는 <b>함수에서 참조하는 변수가 외부에 있어서 변화할 수 있을 때 생기는 것</b>입니다.

Jectpack Compose에서의 Side effect는 <b>컴포저블 범위 밖에서 발생하는 앱 상태에 대한 변경</b>입니다.

컴포저블은 Side-effects에 의존해서는 안 된다고 합니다. 이유가 뭘까요?

컴포즈는 기본적으로 <b>단방향 데이터 흐름</b>을 따릅니다.

단방향 데이터흐름은 <b>상태는 아래로 이동하고 이벤트는 위로 이동하는 디자인 패턴</b>입니다.

Compose에서 단방향 데이터 흐름을 따르기 위해 상태를 컴포저블의 호출자로 옮기는 상태 호이스팅 또는 컴포저블의 로직과 상태를 관리하는 상태 홀더를 구성하기도 합니다.

![](https://developer.android.com/static/images/jetpack/compose/state-unidirectional-flow.png?hl=ko)

만약 안쪽 컴포저블에서 바깥쪽에 있는 컴포저블의 상태를 변경한다면 어떨까요?

단방향이 아닌 양방향 의존성이 생기게 되는 것입니다.
이로 인해 예측할 수 없는 효과가 발생할 수 있습니다.

그러나 Side effect가 필요한 때도 있습니다. 예를 들어 스낵바를 표시하거나 특정 상태 조건에 따라 다른 화면으로 이동하는 일회성 이벤트를 트리거할 때입니다. 이러한 작업은 컴포저블의 lifecycle를 인식하는 관리되는 환경에서 호출해야 합니다.

Side effect가 필요한 때에도 해당 Side effect가 예측 가능한 방식으로 실행되도록 Effect API를 사용해야 합니다.

## LaunchedEffect: 컴포저블 범위에서 suspend 함수를 실행
컴포저블 내부에서 suspend 함수를 안전하게 호출하려면 LaunchedEffect 컴포저블을 사용해야 합니다.

LaunchedEffect는 coroutine scope에서 side effect를 실행시키기 위한 컴포저블 함수입니다.
이 함수는 UI 스레드를 블락하지 않고 긴 시간의 작업(네트워크 호출, 애니메이션)을 처리할 때 유용하게 사용할 수 있을 것입니다.

핵심 기능은 다음과 같습니다.
- LaunchedEffect가 컴포지션에 들어가면 인자로 전달된 코드 블록을 코루틴으로 실행
- LaunchedEffect가 컴포지션을 벗어나면 코루틴이 취소
- key를 활용한 코루틴 재실행


리컴포지션이 일어날 때마다 LaunchedEffect가 취소되고 다시 실행하면 비효율적이기 때문에 LaunchedEffect는 key가 바뀔 때만 LaunchedEffect의 suspend 함수를 취소하고 재실행합니다.
```kotlin
@Composable
@NonRestartableComposable
@OptIn(InternalComposeApi::class)
fun LaunchedEffect(
    key1: Any?,
    block: suspend CoroutineScope.() -> Unit
) {
    val applyContext = currentComposer.applyCoroutineContext
    remember(key1) { LaunchedEffectImpl(applyContext, block) }
}

@Composable
@NonRestartableComposable
@OptIn(InternalComposeApi::class)
fun LaunchedEffect(
    key1: Any?,
    key2: Any?,
    block: suspend CoroutineScope.() -> Unit
) {
    val applyContext = currentComposer.applyCoroutineContext
    remember(key1, key2) { LaunchedEffectImpl(applyContext, block) }
}
```
LaunchedEffect는 remember와 마찬가지로 다수의 key를 인자로 받는데요.
LaunchedEffect 내부에서 remember를 사용하여 코루틴 실행을 위한 객체를 초기화하고
인자로 받은 key를 remember의 인자로 넘겨줍니다.

이렇게 하면 key가 하나라도 변경될 경우 remember는 새로 값을 생성하고 기억할 것입니다.
```kotlin
internal class LaunchedEffectImpl(
    parentCoroutineContext: CoroutineContext,
    private val task: suspend CoroutineScope.() -> Unit
) : RememberObserver {
    private val scope = CoroutineScope(parentCoroutineContext)
    private var job: Job? = null

    override fun onRemembered() {
        job?.cancel("Old job was still running!")
        job = scope.launch(block = task)
    }
    
    override fun onForgotten() {
        job?.cancel(LeftCompositionCancellationException())
        job = null
    }

    override fun onAbandoned() {
        job?.cancel(LeftCompositionCancellationException())
        job = null
    }
}
```
RemeberObserver 인터페이스를 구현하는 LaunchedEffectImpl 클래스는 컴포지션에서 처음 사용될 때와 더 이상 사용되지 않을 때 알림을 받습니다.
```kotlin
interface RememberObserver {
    // 컴포지션에 의해 성공적으로 remember되면 호출됨
    fun onRemembered()
    // 컴포지션에서 이 객체를 잊어버릴 때 호출
    fun onForgotten()
    // 컴포지션에 의해 성공적으로 remember되지 않았을 때 호출
    fun onAbandoned()
}

```
사용 예시
```kotlin
@Composable
fun MyComposable() {
    val isLoading = remember { mutableStateOf(false) }
    val data = remember { mutableStateOf(listOf<String>()) }

    // LaunchedEffect는  오랜 시간이 걸리는 작업을 비동기적으로 실행하기 위해 사용한다.
    // LaunchedEffect내의 로직은 isLoading.value 값이 변경될때마다 취소하고 재실행될 것이다.
    LaunchedEffect(isLoading.value) {
        if (isLoading.value) {
            // 장기간의 작업
            val newData = fetchData()
            // 기존 데이터를 새로운 데이터로 교체한다.
            data.value = newData
            isLoading.value = false
        }
    }

    Column {
        Button(onClick = { isLoading.value = true }) {
            Text("Fetch Data")
        }
        if (isLoading.value) {
            // 로딩 시, 로딩 인디케이터를 보여준다.
            CircularProgressIndicator()
        } else {
            // 데이터를 보여준다.
            LazyColumn {
                items(data.value.size) { index ->
                    Text(text = data.value[index])
                }
            }
        }
    }
}

private suspend fun fetchData(): List<String> {
    // 네트워크 작업을 본뜨기 위해 일부러 2초 delay를 걺
    delay(2000)
    return listOf("Item 1", "Item 2", "Item 3", "Item 4", "Item 5",)
}
```
여기서는 block 람다 내부에 fetchData() 함수를 호출하고 있습니다. 데이터를 가져온 이후 data 변수를 업데이트하고 isLoading의 상태값을 false로 변경하면서 로딩 인디케이터를 숨기고 데이터를 리스트 형태로 보여줍니다.

## 참고
https://developer.android.com/jetpack/compose/side-effects?hl=ko

https://medium.com/@l2hyunwoo/%EB%B2%88%EC%97%AD-jetpack-compose-side-effects-in-details-a98ca403768

https://developer.android.com/codelabs/jetpack-compose-advanced-state-side-effects?hl=ko#0

https://yebon.kim/posts/check-side-effect

https://velog.io/@milkyway/Side-Effect
