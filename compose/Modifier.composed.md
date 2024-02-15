### Android 공식문서

> Declare a just-in-time composition of a Modifier that will be composed for each element it modifies.
> composed may be used to implement stateful modifiers that have instance-specific state for each modified element, allowing the same Modifier instance to be safely reused for multiple elements while maintaining element-specific state.

수정하는 각 요소에 대해 구성될 Modifier의 just-in-time composition을 선언합니다. composed를 사용하여 수정된 각 요소에 대해 인스턴스별 상태를 갖는 stateful modifier를 구현할 수 있으므로 요소별 상태를 유지하면서 여러 요소에 대해 동일한 Modifier 인스턴스를 안전하게 재사용할 수 있습니다

> If inspectorInfo is specified this modifier will be visible to tools during development. Specify the name and arguments of the original modifier.

inspectorInfo를 지정하면 개발 중 도구에 이 modifier가 표시됩니다. 원래 modifier의 이름과 arguments을 지정하십시오.


```kotlin
fun Modifier.composed(
    inspectorInfo: InspectorInfo.() -> Unit = NoInspectorInfo,
    factory: @Composable Modifier.() -> Modifier
): Modifier = this.then(ComposedModifier(inspectorInfo, factory))
```

inspectorInfo는 이름과 속성을 가지고 있습니다.

```kotlin
inspectorInfo = debugInspectorInfo {
	name = "scroll"
	properties["state"] = state
	properties["reverseScrolling"] = reverseScrolling
	properties["flingBehavior"] = flingBehavior
	properties["isScrollable"] = isScrollable
	properties["isVertical"] = isVertical
}
```

예전에는 composed를 통해 Android Modifier와 연관된 기능들을 만들어왔지만 현재 공식문서에서는 지양하라고 얘기하고 있습니다.

> Note: There is another API for creating custom modifiers, composed {}. This API is no longer recommended due to the performance issues it created. Modifier.Node was designed from the ground up to be far more performant than composed modifiers. 
> For more details on the problems with composed modifiers, see the Android Dev Summit talk Compose Modifiers Deep Dive.

출처: https://developer.android.com/jetpack/compose/custom-modifiers