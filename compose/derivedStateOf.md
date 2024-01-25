# SideEffect

SideEffect란 함수에서 결과값을 반환하기 위해 하는 행동들을 제외한 모든 행동입니다. 대표적인 사례로 함수 내부에서 전역 변수의 상태를 수정하거나, API를 호출하는 행위가 있습니다.

SideEffect는 시간에 의존적입니다. 함수를 호출하는 시간(시점)에 따라 동작이 달라질 수 있다는 뜻입니다.

아래 함수들은 SideEffect를 포함한 함수의 예시입니다. getUserList 함수는 서버에 유저목록을 요청하는 함수입니다. DB의 유저 상태는 계속 변화할 수 있기 때문에 해당 함수는 호출하는 시간에 따라 결과값이 달라질 수 있습니다. 따라서 SideEffect가 포함되어 있는 함수입니다. 아래 getScore함수 또한 마찬가지 입니다. key 값이 멤버변수이고, 멤버변수를 참조하고 있기 때문입니다.

```kotlin
suspend fun getUserList(): List<User> {
	return userApi.userList() //side effect
}

class {
		var key = 10

		fun getScore(point: Int): Int {
				return point * key //side Effect
		}
		
		fun updateKey(key: Int): Int {
				this.key = key
		}
}
```

 SideEffect는 코드 추적 및 디버깅을 어렵게 하면서, 코드의 복잡도를 높일 수 있기 때문에 최소화하는 것이 좋습니다. 거대한 함수 하나에 여러 멤버변수의 상태를 참조하고 있고, 해당 변수의 상태를 여기저기서 바꾸는 코드를 보면서, 코드 파악에 어려움을 겪었던 경험이 다들 있으셨을 거라 생각합니다. 

하지만 SideEffect를 아예 없애는 것은 불가능합니다. 따라서 SideEffect를 최소화하기 위해 SideEffect가 발생하는 지점을 한 곳에 묶고, 작은 단위로 함수를 분리하는 노력이 필요하다고 생각합니다.

## Compose SideEffect

*부수효과는 컴포저블 함수의 범위 밖에서 발생하는 앱 상태에 관한 변경사항입니다.*

컴포저블에는 부수효과가 없는 것이 좋습니다. 컴포저블의 리컴포지션의 보장되지 않고, 스킵될 수도 있기 때문입니다. 그러나 부수효과가 반드시 필요한 경우도 있습니다. 예를 들어 스낵바를 표시하거나 특정 상태 조건에 따라 다른화면으로 이동하는 등 일회성 이벤트를 트리거할 때 입니다.

컴포즈에서 사이드 이펙트를 어떻게 올바르게 처리할 수 있을까요? 컴포즈에서는 사이드 이펙트를 핸들링하기 위한 몇 가지 기능을 제공합니다. 컴포즈에서는 사이드 이펙트가 필요한 경우, 예측가능한 방식으로 실행되도록 Effect API 사용을 제안합니다. 

Effect API 중에서 derivedStateOf, DisposableEffect 에 대해 소개하겠습니다.

# derivedStateOf

Q. 왜 SideEffect로 취급되지?

Q. listState 스크롤 시 리컴포지션?

### *derivedStateOf: 하나 이상의 상태 객체를 다른 상태로 변환*

*다른 상태 객체에서 특정 상태가 계산되거나 파생되는 경우 `[derivedStateOf](https://developer.android.com/reference/kotlin/androidx/compose/runtime/package-summary?hl=ko#derivedStateOf(kotlin.Function0))`를 사용하세요. 이 함수를 사용하면 계산에서 사용되는 상태 중 하나가 변경될 때만 계산이 실행됩니다.*

state1과 state2가 변화함에 따라 result값이 업데이트 됩니다.

```kotlin
val result = remember { derivedStateOf { calculation(state1, state2) } }
```

그러면 아래 코드랑 똑같은 것 아닌가요? 라는 의문점이 들 수 있습니다.

아래 코드 또한 state1 또는 state2가 변경되면, result 값이 업데이트 됩니다.

```kotlin
val result = remember(state1, state2) { calculation(state1, state2) }
```

결과적으로 result 값은 위 두코드가 동일하지만. 

`remember(key)`와 `derivedStateOf`의 리컴포지션의 빈도에서 차이가 있습니다.

아래 코드는 LazyColumn의 첫번째 아이템이 보이면 버튼을 활성화하고, 그렇지 않으면 비활성화하는 코드입니다.

1. derivedStateOf 를 사용한 경우, firstVisibleItemIndex > 0 값이 바뀔 때만 ListScreen 컴포저블이 리컴포지션 됩니다.
2. 아래 2,3 번 코드는 scrollPosition 값이 변할 때마다 ListScreen()이 리컴포지션됩니다.
    - 우리는 첫번째 아이템이 보이거나 보이지 않는 타이밍에만 UI를 업데이트하고 싶은데, UI업데이트가 불필요한 상황에서도 계속 리컴포지션이 발생하고 있는 비효율적인 상황입니다.

```kotlin
@Composable
fun ListScreen() {
    val list = remember {
        mutableStateOf(listOf("A", "B", "C"))
    }
    val listState = rememberLazyListState()

		//1
	  val isEnabled by remember { derivedStateOf { listState.firstVisibleItemIndex > 0 } } //listState.firstVisibleItemIndex > 0 값이 바뀔 때만 리컴포지션 됨.
		//2
		val isEnabled = listState.firstVisibleItemIndex > 0 //listState.firstVisibleItemIndex 이 바뀔 때마다 리컴포지션 됨.
    //3
		val isEnabled by remember(listState.firstVisibleItemIndex) { mutableStateOf(listState.firstVisibleItemIndex > 0) } //listState.firstVisibleItemIndex이 바뀔 때마다 리컴포지션 됨.    
    
		Column(modifier = Modifier.fillMaxSize()) {
        LazyColumn(
            modifier = Modifier.weight(1f),
            state = listState
        ) {
            items(list.value) {
                Text(
                    modifier = Modifier
                        .fillMaxWidth()
                        .height(50.dp),
                    text = it,
                    textAlign = TextAlign.Center,
                )
            }
        }
        Box(
            modifier = Modifier
                .fillMaxWidth()
                .height(100.dp)
                .background(if (isEnabled) Color.Red else Color.Gray),
        )
    }
}
```

정리하면, **`derivedStateOf {}`UI를 업데이트하고자 하는 빈도보다 상태나 키가 더 많이 변경될 때 사용하면 됩니다.** 만약, 키가 변경되는 횟수와 UI가 업데이트 되어야 하는 횟수가 동일하다면, **`derivedStateOf` 를 사용하지 않아도 됩니다.**

[Compose의 부수 효과  |  Jetpack Compose  |  Android Developers](https://developer.android.com/jetpack/compose/side-effects?hl=ko#derivedstateof)

[Jetpack Compose — When should I use derivedStateOf?](https://medium.com/androiddevelopers/jetpack-compose-when-should-i-use-derivedstateof-63ce7954c11b)

[언제 derivedStateOf를 써야할까?](https://velog.io/@beokbeok/언제-derivedStateOf를-써야할까)

# **DisposableEffect: 정리가 필요한 효과**

키가 변경되거나 컴포저블이 컴포지션을 종료한 후 *정리*해야 하는 부수 효과의 경우 `[DisposableEffect](https://developer.android.com/reference/kotlin/androidx/compose/runtime/package-summary?hl=ko#DisposableEffect(kotlin.Any,kotlin.Function1))`를 사용하세요. `DisposableEffect` 키가 변경되면 컴포저블이 현재 효과를 *삭제*(정리)하고 효과를 다시 호출하여 재설정해야 합니다.

사용 사례

1. 액티비티 or 프래그먼트에 라이프사이클에 따른 처리
2. 자원 해제

```kotlin
@Composable
fun HomeScreen(
    lifecycleOwner: LifecycleOwner = LocalLifecycleOwner.current,
    onStart: () -> Unit, // Send the 'started' analytics event
    onStop: () -> Unit // Send the 'stopped' analytics event
) {
    // Safely update the current lambdas when a new one is provided
    val currentOnStart by rememberUpdatedState(onStart)
    val currentOnStop by rememberUpdatedState(onStop)

    // If `lifecycleOwner` changes, dispose and reset the effect
    DisposableEffect(lifecycleOwner) {
        // Create an observer that triggers our remembered callbacks
        // for sending analytics events
        val observer = LifecycleEventObserver { _, event ->
            if (event == Lifecycle.Event.ON_START) {
                currentOnStart()
            } else if (event == Lifecycle.Event.ON_STOP) {
                currentOnStop()
            }
        }

        // Add the observer to the lifecycle
        lifecycleOwner.lifecycle.addObserver(observer)

        // When the effect leaves the Composition, remove the observer
        onDispose {
            lifecycleOwner.lifecycle.removeObserver(observer)
        }
    }

    /* Home screen content */
}
```
