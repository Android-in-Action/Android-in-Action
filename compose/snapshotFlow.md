# snapShotFlow

### snapshotFlow를 사용하여 State<T> 객체를 Cold Flow로 변환합니다.

<b>Cold Flow</b>: 각각의 소비자가 수집을 시작하면 데이터를 발행(각 호출마다 실행되므로 서로 다른 Flow 객체로부터 수집), 스트림 생성 후 구독하지 않으면 어떤 연산도 발생하지 않습니다.


Flow를 Compose에서 사용하기 위해서 Flow.collectAsState( )를 호출하여 <br>
<b>Flow -> State</b>로 바꿨다면, 이 API는 <b>State -> Flow</b>로 바꾸는 역할을 합니다.

Flow로 변환하여 사용하는 이유는 Flow가 지원하는 연산자의 이점을 활용할 수 있기 때문입니다.
```kotlin
val listState = rememberLazyListState()

LaunchedEffect(listState) {
snapshotFlow { listState.firstVisibleItemIndex } // Flow<T>를 반환
.map { index -> index > 0 }
.distinctUntilChanged()
.filter { it == true }
.collect {
MyAnalyticsService.sendScrolledPastFirstItemEvent()
}
}
```
위 코드에서 listState.firstVisibleItemInde는 Flow 연산자의 이점을 활용할 수 있는 Flow로 변환됩니다.

snapshotFlow는 수집될 때 블록을 실행하고 읽은 State 객체의 결과를 내보냅니다.

snapshotFlow 블록 내에서 읽은 State 객체의 하나가 변경되면 새 값이 이전에 내보낸 값과 같지 않은 경우 Flow에서 새 값을 수집기에 내보냅니다.




