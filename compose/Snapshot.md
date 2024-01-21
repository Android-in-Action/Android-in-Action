## Snapshot

### Snapshot이란?
Compose는 State를 다룰 때 Snapshot System에 기반하여 관리합니다.

Compose에서 Composable function에 의해 읽히는 모든 State는 다음과 같은 함수에 의해 반환되는 특수 상태 객체에 의해 뒷받침되어야 합니다.
- mutableStateOf / MutableState
- mutableStateListOf / SnapshotStateList
- mutableStateMapOf / SnapshotStateMap
- derivedStateOf
- rememberUpdatedState
- collectAsState

기본적으로 State<T> 인터페이스(MutableState<T>를 포함) 또는 StateObject 인터페이스를 기반으로 구현되어 있습니다.

이 중 예시로서 mutableStateOf의 내부를 확인하면 Snapshot을 기반으로 구현되어 있는 것을 확인할 수 있습니다.
```kotlin
fun <T> mutableStateOf(
    value: T,
    policy: SnapshotMutationPolicy<T> = structuralEqualityPolicy()
): MutableState<T> = createSnapshotMutableState(value, policy)
```

Snapshot 상태 객체를 사용하는 주된 이유는 상태를 유지하고 해당 상태가 변경되면 컴파일이 자동으로 업데이트하게 됩니다.

---
지금부터 설명되는 대부분의 API는 일상적인 코드에서 사용하기 위한 것이 아닙니다. Compose Snapshot이 어떻게 동작하는지만 파악하는 용도로 보시는 것이 좋습니다.

### Registering snapshots

간단한 Dog Class를 만들고 이름이 MutableState 개체에 저장하여 snapshots을 지원하도록 만드는 예시 코드입니다.
```kotlin
class Dog {
  var name: MutableState<String> = mutableStateOf(“”)
}

fun main() {
  val dog = Dog()
  dog.name.value = “Spot”
  val snapshot = Snapshot.takeSnapshot()
  dog.name.value = “Fido”

  // When finished with the snapshot, it must always be disposed.
  // Subsequent examples omit this step for brevity, but don’t
  // forget to do it if you copy and paste any of this code!
  snapshot.dispose()
}
```

### Restoring snapshots
snapshot은 복원기능도 가지고 있습니다.
```kotlin
fun main() {
  val dog = Dog()
  dog.name.value = “Spot”
  val snapshot = Snapshot.takeSnapshot()
  dog.name.value = “Fido”

  println(dog.name.value)
  snapshot.enter { println(dog.name.value) }
  println(dog.name.value)
}

// Output:
Fido
Spot
Fido
```
takeSnapshot()은 호출된 시점에 현재 dog의 이름값을 기억하며 enter 함수는 snapshot 상태를 함수의 모든 코드로 일시적으로 복원합니다

### Mutable snapshots
만약 Snapshot 내부의 값을 변경하고 싶다면 어떻게 해야할까요? 
```kotlin
fun main() {
  val dog = Dog()
  dog.name.value = "Spot"

  val snapshot = Snapshot.takeSnapshot()

  println(dog.name.value)
  snapshot.enter {
    println(dog.name.value)
    dog.name.value = "Fido"
    println(dog.name.value)
  }
  println(dog.name.value)
}

// Output:
Spot

java.lang.IllegalStateException: Cannot modify a state object in a read-only snapshot
```
지금까지의 방법대로 코드를 작성하면 takeSnapshot()으로 생성된 스냅샷은 읽기 전용이기 때문에 값을 수정할 수 없습니다. mutable snapshot를 만들기 위해서는 <b>takeMutableSnapshot()</b>이라는 다른 함수가 필요합니다.

```kotlin
fun main() {
  val dog = Dog()
  dog.name.value = "Spot"

  val snapshot = Snapshot.takeMutableSnapshot()
  println(dog.name.value)
  snapshot.enter {
    dog.name.value = "Fido"
    println(dog.name.value)
  }
  println(dog.name.value)
}

// Output:
Spot
Fido
Spot
```
takeMutableSnapshot()를 사용했을 때 값을 수정할 순 있지만 enter 블록에서 이름을 변경한 후에도 반환을 입력하자마자 변경 내용이 revert됩니다. Snapshot을 사용하면 과거를 확인할 수 있을 뿐만 아니라 변경 내용을 격리하고 변경 내용이 다른 것에 영향을 미치는 것을 방지할 수 있습니다. 

하지만 변경 내용을 “저장”하려면 어떻게 해야 할까요? 우리는 snapshot을 “apply”해야 합니다.

```kotlin
fun main() {
  val dog = Dog()
  dog.name.value = "Spot"

  val snapshot = Snapshot.takeMutableSnapshot()
  println(dog.name.value)
  snapshot.enter {
    dog.name.value = "Fido"
    println(dog.name.value)
  }
  println(dog.name.value)
  snapshot.apply()
  println(dog.name.value)
}

// Output:
Spot
Fido
Spot
Fido
```
apply()를 호출했을 때 변경했던 값이 적용된 것을 볼 수 있습니다. 이런 과정을 편리하게 하는 helper 함수인 <b>Snapshot.withMutableSnapshot()</b>이 있습니다.

```kotlin
fun main() {
  val dog = Dog()
  dog.name.value = "Spot"

  Snapshot.withMutableSnapshot {
    println(dog.name.value)
    dog.name.value = "Fido"
    println(dog.name.value)
  }
  println(dog.name.value)
}
```

지금까지의 내용을 요약하자면 다음과 같습니다
- 모든 state의 snapshot을 관리합니다.
- 특정 코드 블록에 대해 스냅샷을 "복원"합니다.
- 상태값을 변경합니다.

---
이제부터 어떻게 변화를 관찰할 수 있는지 확인하려고 합니다.

### Tracking reads and writes
관찰자 패턴을 이용하여 상태의 변화를 관찰하는 것은 두 부분으로 나누어진 과정입니다. 
1. 상태 레지스터에 의존하는 코드의 부분들을 리스너들을 변화시킵니다. 
2. 값이 바뀌면 등록된 모든 리스너에게 알림을 보냅니다. 관찰자를 수동으로 등록하고 등록 해제를 기억하는 것은 오류가 발생할 수 있습니다. 

Compose state는 명시적인 기능 호출없이 상태 읽기 시 관찰자를 등록할 수 있도록 도와줍니다. 예를 들어 상태를 읽는 코드가 합성 가능한 함수일 경우 Compose runtime은 함수를 재구성하여 업데이트합니다. 

**takeMutableSnapshot()** 함수에는 "읽기 관찰자"와 "쓰기 관찰자"가 필요합니다. 둘 다 enter 블록 안에서 snapshot 상태 값을 읽거나 쓸 때마다 호출되는 "(Any) -> Unit" 함수들입니다.

Dog의 이름이 접근가능한지 알아보겠습니다

```kotlin
fun main() {
  val dog = Dog()
  dog.name.value = "Spot"

  val readObserver: (Any) -> Unit = { readState ->
    if (readState == dog.name) println("dog name was read")
  }
  val writeObserver: (Any) -> Unit = { writtenState ->
    if (writtenState == dog.name) println("dog name was written")
  }

  val snapshot = Snapshot.takeMutableSnapshot(readObserver, writeObserver)
  println("name before snapshot: " + dog.name.value)
  snapshot.enter {
    dog.name.value = "Fido"
    println("name before applying: ")
    // This could be inlined, but I've separated the actual state read
    // from the print statement to make the output sequence more clear.
    val name = dog.name.value
    println(name)
  }
  snapshot.apply()
  println("name after applying: " + dog.name.value)

}

// Output:
name before snapshot: Spot
dog name was written
name before applying: 
dog name was read
Fido
name after applying: Fido
```
dog.name 속성에 저장된 MutableState 인스턴스가 읽기 및 쓰기 관찰자에게 전달되는 것을 확인할 수 있습니다. 관찰자는 값에 액세스할 때도 즉시 호출됩니다. 위의 출력에는 "dog name was read"는 내용이 실제 이름 앞에 출력되어 있습니다.

스냅샷의 또 다른 장점은 상태 읽기가 콜 스택의 깊이에 상관없이 관찰된다는 것입니다. 이것은 상태 읽기 코드를 함수나 속성으로 factor할 수 있다는 것을 의미하며, 그 읽기는 계속 추적됩니다. 상태 유형의 속성 위임 확장자는 이것에 의존합니다. 이것을 증명하기 위해 예제에 몇 가지 방향을 추가하고 읽기가 여전히 보고된다는 것을 확인할 수 있습니다

```kotlin
fun Dog.getActualName() = nameValue
val Dog.nameValue get() = name.value

fun main() {
  val dog = Dog()
  dog.name.value = "Spot"

  val readObserver: (Any) -> Unit = { readState ->
    if (readState == dog.name) println("dog name was read")
  }

  val snapshot = Snapshot.takeSnapshot(readObserver)
  snapshot.enter {
    println("reading dog name")
    val name = dog.getActualName()
    println(name)
  }
}

// Output:
reading dog name
dog name was read
Spot
```

또한 Snapshot은 상태 변경 사항은 중복 제거됩니다. 상태 값이 동일한 값으로 설정되면 해당 값에 대한 변경 사항은 기록되지 않습니다. 충돌하는 스냅샷 쓰기를 검토할 때 이 게시물의 뒷부분에서 볼 수 있듯이 "동일한" 값을 정의하는 논리는 실제로 정책을 통해 사용자 지정할 수 있습니다.


### SnapshotStateObserver
특정 함수의 모든 읽기를 추적한 다음 이러한 상태 값이 변경되면 콜백을 실행하는 패턴은 매우 일반적이어서 이를 구현하는 클래스인 `SnapshotStateObserver`가 있습니다. 클래스의 사용법은 다음과 같습니다

1. `SnapshotStateObserver`를 생성하고 Java 스타일의 `Executor` 역할을 하는 함수를 전달하여 변경 알림 콜백을 실행합니다.
2. `start()`을 눌러 변경 사항을 확인하기 시작합니다.
3. 한 번 이상 `observeReads()`를 호출하여, 읽기 값 중 하나가 변경되었을 때 실행할 콜백 뿐만 아니라 에서 읽기를 관찰할 수 있는 기능을 전달합니다. 콜백이 호출될 때마다 관찰 중인 상태 집합이 지워지고 변경 사항을 계속 추적하려면 `observeReads()`를 다시 호출해야 합니다.
4. `stop()` 및 `clear()`를 호출하여 변경 사항을 모니터링하고 변경 사항을 확인하는 데 필요한 리소스를 해제합니다.

이는 좀 더 긴 예이지만 `SnapshotStateObserver`를 사용하여 dog의 이름에 대한 상태 변경을 관찰할 수 있는 방법을 보여줍니다
```kotlin
fun main() {
  val dog = Dog()

  fun immediateExecutor(runnable: () -> Unit) {
    runnable()
  }

  fun blockToObserve() {
    println("dog name: ${dog.name.value}")
  }

  val observer = SnapshotStateObserver(::immediateExecutor)

  fun onChanged(scope: Int) {
    println("something was changed from pass $scope")
    println("performing next read pass")
    observer.observeReads(
      scope = scope + 1,
      onValueChangedForScope = ::onChanged,
      block = ::blockToObserve
    )
  }

  dog.name.value = "Spot"

  println("performing initial read pass")
  observer.observeReads(
    // This can be literally any object, it doesn't need
    // to be an int. This example just uses an int to
    // demonstrate subsequent read passes.
    scope = 0,
    onValueChangedForScope = ::onChanged,
    block = ::blockToObserve
  )

  println("starting observation")
  observer.start()

  println("initial state change")
  Snapshot.withMutableSnapshot {
    dog.name.value = "Fido"
  }

  println("second state change")
  Snapshot.withMutableSnapshot {
    dog.name.value = "Fluffy"
  }

  println("stopping")
  observer.stop()

  println("third state change")
  Snapshot.withMutableSnapshot {
    // This change won't trigger the callback.
    dog.name.value = "Fluffy"
  }
}

// Output:
performing initial read pass
dog name: Spot
starting observation
initial state change
something was changed from pass 0
performing next read pass
dog name: Fido
second state change
something was changed from pass 1
performing next read pass
dog name: Fluffy
stopping
third state change
```
하지만 snapshot state 값들은 snapshot 내부에서만 변경되는 것이 아니라 어디서나 변경할 수 있다는 점은 여전히 중요한 요소입니다. 초기 샘플로 돌아가서 메인 기능에서 이름을 직접 변경할 수 있으므로 이러한 "최상위 수준" 쓰기를 확인할 수 있는 방법이 있어야 합니다.

### Nested snapshots
실제로 모든 코드는 snapshot에서 실행됩니다. 명시적으로 생성된 snapshot이 없어도 말입니다. 스냅샷에는 아직 설명하지 않은 다른 기능이 있습니다. 
다른 snapshot의 enter 블록 안에서 `takeMutableSnapshot()`을 호출하면 내부 snapshot이 적용되면 외부 스냅샷에만 적용됩니다. 
그런 다음 변경 사항이 snapshot 계층 위로 계속 전파되도록 하려면 변경 사항이 `root snapshot`을 차례로 적용해야 합니다. 
이 `root snapshot`은 "global snapshot"이라고 불리며, 항상 열려 있습니다. global snapshot은 조금 특별한 경우이므로, 그 이야기를 시작하기 전에 `snapshot nesting`에 대해 알아보겠습니다.

innerSnapshot이라는 nested snapshot을 생성하여 dog의 이름을 변경하고 어떤 일이 발생하는지 알아보겠습니다
```kotlin
fun main() {
  val dog = Dog()
  dog.name.value = "Spot"

  val outerSnapshot = Snapshot.takeMutableSnapshot()
  println("initial name: " + dog.name.value)
  outerSnapshot.enter {
    dog.name.value = "Fido"
    println("outer snapshot: " + dog.name.value)

    val innerSnapshot = Snapshot.takeMutableSnapshot()
    innerSnapshot.enter {
      dog.name.value = "Fluffy"
      println("inner snapshot: " + dog.name.value)
    }
    println("before applying inner: " + dog.name.value)
    innerSnapshot.apply().check()
    println("after applying inner: " + dog.name.value)
  }
  println("before applying outer: " + dog.name.value)
  outerSnapshot.apply().check()
  println("after applying outer: " + dog.name.value)
}

// Output:
initial name: Spot
outer snapshot: Fido
inner snapshot: Fluffy
before applying inner: Fido
after applying inner: Fluffy
before applying outer: Spot
after applying outer: Fluffy
```
마지막 두 줄, 즉 외부 스냅샷을 적용하기 전까지는 최상위 값에 "Fluffy"로 이름을 변경하는 작업이 적용되지 않았습니다. 
스냅샷을 상태 변이 작업으로 적용하는 것을 생각하면 이해가 됩니다. 
스냅샷 내부의 상태 변이는 apply()를 호출하기 전까지는 적용되지 않기 때문에 중첩된 스냅샷의 응용 프로그램은 외부 스냅샷을 적용하기 전까지는 볼 수 없는 또 다른 유형의 작업에 불과합니다.

### The global snapshot
global snapshot은 snapshot tree의 루트에 위치하는 변형 가능한 snapshot입니다. 
적용하려면 일반적인 변형 가능한 snapshot 달리 Global Snapshot에는 “apply” 작업이 없고 적용할 작업이 없습니다. 
대신 "advanced" 상태가 될 수 있습니다. 
Global snapshot을 advance하는 것은 자동으로 적용한 후 즉시 다시 여는 것과 유사합니다. 
마지막 Advance 이후 Global Snapshot에서 수정된 모든 Snapshot State 값들에 대해 변경 알림이 전송됩니다.

글로벌 스냅샷을 발전시키는 방법은 다음 세 가지가 있습니다:
1. mutable snapshot을 적용합니다. 위 예에서는 outer snapshot을 적용하면 global snapshot에 적용되고 global snapshot은 advanced됩니다.
2. `Snapshot.notifyObjectsInitialized`을 호출합니다. 마지막 진행 이후 변경된 상태 값에 대한 변경 알림을 보냅니다.
3. `Snapshot.sendApplyNotifications()`을 호출합니. 이는 `n다ifyObjectsInitialized`와 유사하지만 실제로 변경된 사항이 있는 경우에만 snapshot을 advance시킵니다. 이 함수는 mutable snapshot이 global snapshot에 적용될 때마다 첫 번째 경우에 암묵적으로 호출됩니다.

global snapshot은 `takeMutableSnapshot` 호출로 생성되지 않기 때문에 일반적인 것처럼 읽기 및 쓰기 관찰자를 전달할 수 없습니다. 대신 global snapshot에서 상태 변경이 적용되는 시점에 대한 콜백을 설치하는 전용 기능인 `Snapshot.ApplyObserver()`가 있습니다.

명시적인 스냅샷 없이 `registerApplyObserver` 및 `sendApplyNotifications`를 사용하여 dog의 이름을 변경합니다

```kotlin
fun main() {
  val dog = Dog()
  Snapshot.registerApplyObserver { changedSet, snapshot ->
    if (dog.name in changedSet) println("dog name was changed")
  }

  println("before setting name")
  dog.name.value = "Spot"
  println("after setting name")

  println("before sending apply notifications")
  Snapshot.sendApplyNotifications()
  println("after sending apply notifications")
}

// Output:
before setting name
after setting name
before sending apply notifications
dog name was changed
after sending apply notifications
```
Compose runtime은 global snapshot API를 사용하여 UI에 대한 그리기를 담당하는 Frame과 Snapshot을 조정합니다.
무언가가 composition 외부의 snapshot state 값을 변경할 때(예: 클릭이 일부 상태를 변경하는 이벤트 핸들러를 시작하는 것과 같은 사용자 이벤트), 
Compose runtime은 다음 frame을 그리기 전에 `sendApplyNotifications` 호출이 발생하도록 예약합니다. 
frame을 생성할 때가 되면 `sendApplyNotifications` 호출을 통해 global snapshot을 진행하고 변경 사항을 적용합니다. 
global snapshot에서 전송된 결과 변경 알림은 재구성해야 할 composables를 결정하는 데 사용됩니다. 
그런 다음 Compose는 변경 가능한 스냅샷을 만들고 해당 snapshot 내부의 composables을 재구성한 다음 최종적으로 스냅샷을 적용합니다. 
그 snapshot application은 global snapshot을 다시 발전시키고 재구성하는 동안 수행된 상태 변경 사항을 다른 스레드에서 실행되는 코드를 포함하여 global snapshot에서 실행되는 코드에 표시되도록 만듭니다.

### Multithreading and the global snapshot
global snapshot는 다중 스레드 코드에 중요한 의미를 갖습니다. 
명시적인 snapshots를 사용하지 않고 백그라운드 스레드에서 코드를 실행하는 것은 드문 일이 아닙니다. 
응답과 함께 MutableState를 업데이트하는 IO 디스패처의 LaunchedEffect에서 네트워크 호출을 고려해 보겠습니다.
snapshots가 없다면, 다른 스레드에서 상태를 나타내는 코드를 읽는다면 즉시 새 값을 볼 수 있고, 잘못된 시간에 값이 변경되면 race condition을 일으킬 수 있습니다. \
그러나 snapshots에는 "분리"라는 속성이 있습니다.

지정된 스레드의 snapshot 내에서는 해당 snapshot이 적용될 때까지 다른 스레드의 값을 상태로 변경한 내용이 표시되지 않습니다.
snapshot은 다른 snapshot으로부터 "분리"됩니다. 
코드가 어떤 상태에서 작동해야 하고 그 사이에 다른 스레드가 그것을 방해할 수 없도록 하려면 일반적으로 그 코드는 뮤텍스와 같은 것을 사용하여 해당 상태에 대한 액세스를 보호합니다. 
그러나 snapshot들은 분리되어 있기 때문에 대신 snapshot을 생성하고 snapshot 내부의 상태에서 작동할 수 있습니다. 
그러면 다른 스레드가 상태를 변경하는 경우 snapshot이 있는 스레드는 snapshot이 적용될 때까지 변경 사항을 전혀 알지 못하며, 그 반대의 경우도 마찬가지입니다. 
snapshot이 적용되고 global snapshot이 자동으로 advance될 때까지 snapshot 내부의 상태를 변경한 내용은 다른 스레드에서 볼 수 없습니다.

### Conflicting snapshot writes
만약 컴포지트가 snapshot을 사용하여 여러 스레드에서 병렬로 코드를 실행한다면, 여러 개의 snapshot이 필요할 것입니다.

몇 가지 변형 가능한 이름을 선택하고 이름을 동시에 변형해 보겠습니다

```kotlin
fun main() {
  val dog = Dog()
  dog.name.value = "Spot"

  val snapshot1 = Snapshot.takeMutableSnapshot()
  val snapshot2 = Snapshot.takeMutableSnapshot()

  println(dog.name.value)
  snapshot1.enter {
    dog.name.value = "Fido"
    println("in snapshot1: " + dog.name.value)
  }
  // Don’t apply it yet, let’s try setting a third value first.

  println(dog.name.value)
  snapshot2.enter {
    dog.name.value = "Fluffy"
    println("in snapshot2: " + dog.name.value)
  }

  // Ok now we can apply both.
  println("before applying: " + dog.name.value)
  snapshot1.apply()
  println("after applying 1: " + dog.name.value)
  snapshot2.apply()
  println("after applying 2: " + dog.name.value)
}

// Output:
Spot
in snapshot1: Fido
Spot
in snapshot2: Fluffy
before applying: Spot
after applying 1: Fido
after applying 2: Fido
```
두 enter 블록 모두 내부적으로 수정된 값을 확인했으며 첫 번째 snapshot은 변경 사항을 적용했지만 두 번째 snapshot을 적용한 후에도 이름은 "Fluffy"가 아닌 "Fido"였습니다. 
여기서 무슨 일이 일어나고 있는지 이해하기 위해 apply() 방법을 자세히 알아보겠습니다. 

실제로 `SnapshotApplyResult` 유형의 값이 반환되는데, 이는 Success 또는 Failure가 될 수 있는 sealed class입니다. 
apply() 호출 주변에 statements 출력을 추가하면 첫 번째는 성공하지만 두 번째는 실패하는 것을 확인할 수 있습니다. 
그 이유는 업데이트 간에 해결할 수 없는 충돌이 있기 때문입니다.
두 snapshots 모두 동일한 초기 값("Spot")을 기준으로 동일한 이름 값을 변경하려고 합니다. 
두 번째 snapshot는 "Spot"이라는 이름으로 실행되었기 때문에 snapshot 시스템에서는 새 값인 "Fluffy"가 여전히 정확하다고 가정할 수 없습니다. 
snapshot을 적용한 후 enter 블록을 다시 실행하거나 이름을 병합하는 방법을 명시적으로 알려주어야 합니다. 
이는 충돌이 있는 깃 브랜치를 병합하려고 할 때와 동일한 상황입니다. 충돌이 해결될 때까지 병합이 실패합니다.

Compose는 실제로 병합 충돌을 해결하기 위한 API를 가지고 있습니다! 
`mutableStateOf()`는 선택적인 `SnapshotMutationPolicy`을 취합니다. 
정책은 특정 유형(equivalent)의 값을 비교하는 방법과 충돌(merge)을 해결하는 방법을 모두 정의합니다. 

- structuralEqualityPolicy – 동일한 방법(==)을 사용하여 개체를 비교하고, 모든 쓰기는 충돌하지 않는 것으로 간주됩니다
- referentialEqualityPolicy – 참조(===)로 개체를 비교하며, 모든 쓰기는 충돌하지 않는 것으로 간주됩니다.
- neverEqualPolicy – 모든 개체를 동일하지 않은 개체로 취급하며 모든 쓰기는 충돌하지 않는 것으로 간주됩니다. 예를 들어, 스냅샷에 변경 가능한 값이 유지되고 == 또는 ===에서 감지할 수 없는 방식으로 값이 변경되었음을 나타내는 데 이 값을 사용해야 합니다. 변경 가능한 상태의 개체에 변경 가능한 개체를 유지하는 것은 안전하지 않지만 개체 구현이 사용자의 제어에 없는 경우 유용합니다.

사전에 구축된 정책 중 쓰기 충돌을 해결하는 정책은 없습니다. 충돌 해결은 사용 사례별로 매우 다르므로 기본값이 합리적이지 않습니다. 
반면에 충돌을 사소한 방식으로 해결할 수도 있습니다: 병합 방법에 대한 방법 설명서에는 카운터의 예가 포함되어 있습니다. 
카운터가 두 snapshots씩 독립적으로 증분되는 경우 병합된 카운터 값은 카운터가 두 snapshots에서 모두 증분된 양의 합입니다.

```kotlin
class Dog {
  var name: MutableState<String> =
    mutableStateOf("", policy = object : SnapshotMutationPolicy<String> {
      override fun equivalent(a: String, b: String): Boolean = a == b

      override fun merge(previous: String, current: String, applied: String): String =
        "$applied, briefly known as $current, originally known as $previous"
    })
}

fun main() {
  // Same as before.
}

// Output:
Spot
in snapshot1: Fido
Spot
in snapshot2: Fluffy
before applying: Spot
after applying 1: Fido
after applying 2: Fluffy, briefly known as Fido, originally known as Spot
```
snapshot 적용으로 인해 해결할 수 없는 쓰기 충돌이 발생하면 적용 작업이 실패한 것으로 간주됩니다. 이 시점에서 새 snapshot를 사용하고 변경 내용을 다시 시도하는 것만 남았습니다.

### Conclusion
Compose’s snapshot 시스템은 다음과 같은 설계의 주요 기능을 제공합니다:

1. Reactivity: 상태 정보 코드는 항상 자동으로 최신 상태를 유지합니다. 기능 개발자뿐만 아니라 라이브러리 개발자로서도 무효화 또는 구독을 추적하는 것에 대해 걱정할 필요가 없습니다. 수동으로 더티 플래그를 추가하지 않고도 snapshot을 사용하여 자신만의 "반응형" 구성 요소를 만들 수 있습니다.
2. Isolation: Stateful 코드는 다른 스레드에서 실행되는 코드에 의해 그 아래의 상태가 변경될 걱정 없이 상태에서 작동할 수 있습니다. Compose자는 이 이점을 이용하여 여러 백그라운드 스레드에 팬아웃 recompositions과 같이 이전 View 툴킷에서는 할 수 없었던 트릭을 수행할 수 있습니다.

나머지 구성을 사용하지 않고 스냅샷 시스템으로 플레이하려면 런타임 아티팩트에 대한 종속성을 추가하기만 하면 됩니다:

```kotlin
implementation("androidx.compose.runtime:runtime:$composeVersion")
```

#### 참고자료
https://dev.to/zachklipp/introduction-to-the-compose-snapshot-system-19cn