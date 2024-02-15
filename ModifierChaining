#Modifier Chaining

## 1. 함수 체이닝

Jetpack Compose의 Modifier 시스템에서 함수 체이닝은 클래스에 순차적으로 Modification을 적용할 수 있도록 합니다.

```kotlin
Modifier
	.width(10.dp)
	.height(10.dp)
	.padding(start = 8.dp)
	.background(color = Color(0xFFFFF0C4))
```

각 함수 (**`width`**, **`height`**, **`padding`**, **`background`**)는 연쇄적으로 호출될 수 있습니다. 이러한 체이닝은 원하는 Modification이 적용될 때까지 계속되며, 간편하고 간결한 구문을 제공합니다.

<aside>
💡 함수 체이닝을 사용하는 입장에서는 되게 간단하죠?

</aside>

## 2. Modifier와 유사한 패턴에 대해 생각해보기

Modifier와 유사한 패턴을 어떻게 구현할 수 있는지 살펴보겠습니다.

예를 들어 Car 클래스를 고려해 봅시다. 이 클래스에는 ownerName과 color 두 속성이 있습니다.

```kotlin
class Car(var ownerName: String, var color: String) {

    fun changeOwner(newName: String) {
        this.ownerName = newName
    }

    fun repaint(newColor: String) {
        this.color = newColor
    }
}

fun main() {
	var myCar: Car = Car("Sid","Red")
	
	// 함수를 연속해서 호출하는 표준 방법
	myCar.changeOwner("New Owner")
	myCar.repaint("Blue")
	
	// 불가능, 함수를 연쇄적으로 호출할 수 없음
	myCar.changeOwner("Sid Patil").repaint("Green")
}
```

우리가 생각할 수 있는 첫 번째 해결책은 동일한 인스턴스를 반환하는 것이 좋다고 생각할 수 있습니다. 그러나 이 방식은 모든 함수를 동일한 인스턴스를 반환하도록 변경해야 하므로 이상적이진 않습니다.

```kotlin

class Car(var ownerName: String, var color: String) {

	//두 함수 모두 인스턴스를 반환하고, 반환 타입이 있습니다.
	fun changeOwner(newName: String) : Car {
	    this.ownerName = newName
	    return this
	}
	
	fun repaint(newColor: String) : Car {
	    this.color = newColor
	    return this
	}
}
```

위의 방법은 잘 동작하지만 여러 타입을 지원하지 않으며 직관적이지(intuitive) 않습니다. 

Jetpack Compose의 Modifier 시스템은 **`PaddingModifier`**, **`FillModifier`** 등과 같은 다른 타입의 Modifier를 사용하고 결합합니다. 따라서 위의 방법은 적합하진 않습니다.

그렇다면 인스턴스를 반환하여 함수 체이닝을 구현하는 것과 어떻게 다를까요?

1. 첫째로, 이 방법은 여러 타입의 클래스를 지원하며, 다양한 Modifier의 구현체가 반환되어도 체인이 여전히 작동합니다. (Since modifier is an interface)
2. 둘째로, Modifier 시스템은 체이닝을 가능하게 하는 집계(aggregation) 로직을 사용합니다.

## 3. 그렇다면 집계(aggregation)란 무엇일까요?

우리의 맥락에서는 작업(operations)을 함께 적용하는 행위를 나타냅니다. 즉, 집계는 함수를 체인에 연속적으로 적용하여 인스턴스를 변경하는 행위를 의미합니다

Compose Modifier 로직의 핵심은 집계(aggregation)로, 연산을 모두 함께 적용하는 것을 의미합니다. Kotlin의 **`fold()`** 함수는 집계를 가능하게 하며, 모든 작업이 수행된 후 결과를 수집할 수 있습니다. 이는 **`fold`**를 사용하여 숫자를 합하는 예제에서 시각화할 수 있습니다.

```kotlin
val numbers = listOf(1,2,3,4,5)

// 왼쪽에서 오른쪽으로 집계 연산
// 0 + 1 -> 1 + 2 -> 3 + 3 -> 5 + 4 -> 9 + 5 = 15
numbers.fold(0) { total, number -> total + number}

// 오른쪽에서 왼쪽으로 집계 연산
// 0 + 5 -> 5 + 4 -> 9 + 3 -> 12 + 2 -> 14 + 1 = 15
numbers.foldRight(0) {total, number -> total + number}
```

폴드의 방향(왼쪽에서 오른쪽 또는 오른쪽에서 왼쪽)은 최종 결과를 결정하는 데 중요합니다. 우리 예제에서의 작업은 합산이었기 때문에 결과는 동일했습니다. 작업이 복잡하고 이전 작업에 따라 결과가 달라진다면, 폴드 방향에 따라 최종 결과가 달라질 수 있습니다. 이것이 Compose Modifier에서 함수 호출의 순서가 UI를 렌더링하는 방법을 결정하는 데 중요한 이유입니다.

이제 집계(aggregation)가 어떻게 작동하는지 이해했으니, Compose Modifier가 어떻게 작동하는지 자세히 살펴보겠습니다.

## 4. Compose UI 모디파이어의 핵심

1. Modifier Interface: 구현 클래스에서 수행할 수 있는 작업을 개요화합니다.
2. Modifier Element: 모디파이어 체인 내의 노드로, 링크를 형성합니다.
3. Modifier Companion: 체인을 시작하는 데 주로 책임이 있는 객체로, 모디파이어 인터페이스를 구현합니다.
4. Combined Modifier: 체인의 단위를 형성하며 외부 및 내부 링크를 연결합니다.

Compose UI의 이러한 구성 요소의 조합은 간소화된 함수 체이닝 패턴으로 UI를 구축하는 강력하고 유연한 시스템을 가능케 합니다. 이 체인의 시각화는 외부 및 내부 객체 래핑 및 이 래핑으로 인한 링크 형성을 염두에 두고 이루어집니다.

요약하면, Jetpack Compose의 Modifier 시스템은 Kotlin의 확장 함수 기능을 활용하며, 여러 유형과 모디파이어의 다양한 구현을 지원하며 집계 논리를 사용하여 UI 개발에서 직관적인 함수 체이닝을 가능케 합니다.

### I. Modifier Interface

Modifier 인터페이스는 Modifier 로직의 시작 단위입니다. 이 인터페이스는 이를 구현한 클래스에서 수행할 수 있는 다양한 작업을 개요로 제공합니다.

```kotlin
interface Modifier {

    fun <R> foldIn(initial: R, operation: (R, Element) -> R): R

    fun <R> foldOut(initial: R, operation: (Element, R) -> R): R

    fun any(predicate: (Element) -> Boolean): Boolean

    fun all(predicate: (Element) -> Boolean): Boolean

    infix fun then(other: Modifier): Modifier =
        if (other === Modifier) this else CombinedModifier(this, other)

}
```

**`foldIn()`** 함수는 **`fold()`** 함수처럼 왼쪽에서 오른쪽으로 이동하며 집계 섹션에서 이해한 대로 작동합니다. 

**`foldOut()`**은 반대로 작동하며, 체인에서 오른쪽에서 왼쪽으로 이동하며 작동합니다.

**`then()`** 함수는 작업을 연결하는 데 사용되며 Modifier를 인수로 받아 CombinedModifier를 반환합니다. 첫 번째 참조 동일성 검사는 객체를 체인화하지 않도록 보장하며, 그렇게 하지 않은 경우 self의 인스턴스를 반환합니다.

나머지 두 함수인 **`any()`**와 **`all()`**은 런타임에 materialize 과정에서 사용됩니다. Materialize 함수는 Compose의 확장으로, Modifier를 런타임 트리 노드에 첨부하거나 Android Studio의 레이아웃 인스펙터를 통해 툴링에 사용하기 위해 Modifier를 준비합니다.

### II. Modifier Element

Modifier Element은 Modifier 체인 내의 노드로, 이를 연결점으로 생각할 수 있습니다. 여러 연결점이 모여 전체 체인 데이터 구조를 형성합니다. Modifier Element 역시 인터페이스입니다.

```kotlin
interface Element : Modifier {
    override fun <R> foldIn(initial: R, operation: (R, Element) -> R): R =
        operation(initial, this)

    override fun <R> foldOut(initial: R, operation: (Element, R) -> R): R =
        operation(this, initial)

    override fun any(predicate: (Element) -> Boolean): Boolean = predicate(this)

    override fun all(predicate: (Element) -> Boolean): Boolean = predicate(this)
}
```

### III. Modifier Companion

Modifier Companion은 주로 체인을 시작하는 데 책임을 지는 객체로, Modifier 인터페이스를 구현하는 단순한 동반 객체입니다.

```kotlin
companion object : Modifier {
    override fun <R> foldIn(initial: R, operation: (R, Element) -> R): R 
			= initial
    override fun <R> foldOut(initial: R, operation: (Element, R) -> R): R 
			= initial
    override fun any(predicate: (Element) -> Boolean): Boolean = false
    override fun all(predicate: (Element) -> Boolean): Boolean = true
    override infix fun then(other: Modifier): Modifier = other
    override fun toString() = "Modifier"
}

```

이 동반 객체는 Modifier.width().height()...와 같이 체인을 시작할 때 호출되는 것입니다.

### IV. CombinedModifier

Combined Modifier는 체인의 단위이며, Element와 다른 CombinedModifier 인스턴스를 연결하는 Outer 및 Inner 연결점으로 구성됩니다. 이는 Modifier 인터페이스를 구현하는 별도의 클래스입니다.

```kotlin
class CombinedModifier(
    private val outer: Modifier,
    private val inner: Modifier
) : Modifier {
    override fun <R> foldIn(initial: R, operation: (R, Modifier.Element) -> R): R =
        inner.foldIn(outer.foldIn(initial, operation), operation)

    override fun <R> foldOut(initial: R, operation: (Modifier.Element, R) -> R): R =
        outer.foldOut(inner.foldOut(initial, operation), operation)

    override fun any(predicate: (Modifier.Element) -> Boolean): Boolean =
        outer.any(predicate) || inner.any(predicate)

    override fun all(predicate: (Modifier.Element) -> Boolean): Boolean =
        outer.all(predicate) && inner.all(predicate)

    override fun equals(other: Any?): Boolean =
        other is CombinedModifier && outer == other.outer && inner == other.inner

    override fun hashCode(): Int = outer.hashCode() + 31 * inner.hashCode()

    override fun toString() = "[" + foldIn("") { acc, element ->
        if (acc.isEmpty()) element.toString() else "$acc, $element"
    } + "]"
}

```

Modifier를 조직하는 데 데이터 구조로서 체인이라는 용어를 사용하고 있지만, 이를 제대로 이해하는 방법은 실제로 외부 체인 노드가 다른 체인 노드를 감싸고 있다는 것입니다. 체인의 각 노드는 본질적으로 연결점을 형성하는 CombinedModifier입니다.

### Visualizing compose modifier function chain

Modifier 함수 체인을 시각화하는 가장 좋은 방법은 외부와 내부 객체 래핑 및 이 래핑으로 형성된 연결점을 고려하는 것입니다.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/1eaee41b-259b-44b2-863c-7426bbfb2402/b3634a12-1b58-4232-afe8-89217ddb02ca/Untitled.png)

### **fold 방향을 기준으로 뷰 중첩 시각화**

**Visualizing view nesting based on fold direction**

체이닝된 Modifier 살펴보는 또 다른 방법은 조합할 때, Modifier가 어떻게 적용되는지 시각화하는 것입니다. 조합의 방향은 가장 외부의 뷰에서 가장 내부의 뷰로 향합니다.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/1eaee41b-259b-44b2-863c-7426bbfb2402/4b187aad-3902-41ba-90cb-9aac3f9b971f/Untitled.png)

```kotlin
Modifier
	.width(10.dp)
	.height(10.dp)
	.padding(start = 8.dp)
	.background(color = Color(0xFFFFF0C4))
```

이것을 생각할 때, `Modifier.width(10.dp)`가 외부 객체이고, 이것이 `.height(10.dp)`를 래핑하며, 이것이 `.padding(start = 8.dp)`를 래핑하며, 마지막으로 `.background(color = Color(0xFFFFF0C4))`가 이 모든 것을 감싸는 구조입니다. 이것이 체인이 어떻게 형성되는지를 시각적으로 이해할 수 있게 해줍니다.

> 이러한 객체들을 결합된 객체 내부에 래핑하고 이 래핑을 통해 링크를 형성하면서, 집계하는 fold 함수를 사용하여 이 링크를 통해 탐색할 수 있도록 하는 것이 Compose Modifier 로직의 간단한 천재성입니다. or 객체를 결합된 객체 내에 래핑하고 이 래핑을 통해 객체들을 서로 연결하는 것은 조합 모디파이어 논리의 간단한 천재성입니다. 이러한 래핑을 통해 링크를 통합하면서 폴드 함수를 사용하여 링크를 통해 탐색할 수 있게 하는 것이 중요합니다.
> 

또한, 필요한 경우 inner 객체에서 outer 객체로 집계를 활성화할 수 있는 foldOut() 함수도 살펴보았습니다.

## **Building our function chaining with Composition**

[Build function chains using composition in Kotlin](https://siddroid.com/post/kotlin/build-function-chains-using-composition-in-kotlin/?hmsr=joyk.com&utm_source=joyk.com&utm_medium=referral#i-car-example)
