# Kotlin 1.5+ changes

Сам роадмап [тут](https://youtrack.jetbrains.com/agiles/153-1251/current?_gl=1*1m4ozzq*_ga*MTkzNTA0MDA0Ni4xNjM0MjIxMzky*_ga_J6T75801PF*MTYzNDI0MTczNS44LjEuMTYzNDI0MzU0Ny40Mw..&_ga=2.95665052.1693832810.1634221401-1935040046.1634221392)

## 1. Exhaustive when block

Концепция описана в [KT-12380](https://youtrack.jetbrains.com/issue/KT-12380)

При указании sealed when (возможно, семантика изменится; рассматривают вариант when!!). 
Sealed when не будут иметь else (ошибка компиляции), но в байткоде else будет присутствовать и будет генерировать исключение.
Текущее поведение
```kotlin
enum class State {  
    OK, FAILED, CREATED  
}  
  
class RecordProcessor {  
    fun processRecord() : String {  
        val state = State.OK  
        //на строке ниже ошибка
        //'when' expression must be exhaustive, add necessary 'FAILED', 'CREATED' branches or 'else' branch instead
	    val result: Boolean? = when (state) { 
            State.OK -> true  
        }  
          
        //на строке ниже ошибка
        //'when' expression must be exhaustive, add necessary 'FAILED', 'CREATED' branches or 'else' branch instead
        return when(result) {  
            true -> "Processed"  
        }  
    }  
}
```
Будет поддерживать Enum, Boolean, sealed classes/interfaces

```kotlin
sealed interface Either<L, R> {  
    fun <U> map(leftBlock: (L) -> U, rightBlock: (R) -> U) : U {  
        // WARNING: Non exhaustive 'when' statements on sealed classes/interfaces  
        // will be prohibited in 1.7, add an 'Left' or 'else' branch instead
        return sealed when(this) { 
            is Right -> rightBlock(?)    
        }  
    }  
}  
  
class Right<L, R> : Either<L, R>  
class Left<L, R> : Either<L, R>
```
Превью фичи доступно в [Kotlin 1.5.30](https://kotlinlang.org/docs/whatsnew1530.html#exhaustive-when-statements-for-sealed-and-boolean-subjects)
Включить можно так:
```kotlin
kotlin {  
    sourceSets.all {  
      languageSettings.apply {  
      languageVersion = "1.6"  
      //progressiveMode = true // false by default  
     }  
  }
}
```
`progressiveMode` Warning превратит в Error
Эта проверка должна еще больше расширить удобство sealed class/interface

## 2. Suspend functions as super-types

Описание тут [KT-18707](https://youtrack.jetbrains.com/issue/KT-18707)

До этой фичи код вида
```kotlin
interface FlowAction : (FlowCollector<String>) -> Unit {  
    companion object {  
        fun emitOne(): FlowAction {  
            return object : FlowAction {  
                override fun invoke(p1: FlowCollector<String>) {  
                    //Suspend function 'emit' should be called only from a coroutine or another suspend function
                    p1.emit("1")  
                }  
            }  
        }  
    }  
}
class FlowProcessor {  
    fun processFlow(flowAction: suspend FlowCollector<String>.() -> Unit): Flow<String> {  
        return flow<String> {  
            emit("123")  
            emit("321")  
            flowAction()  
        }  
  }  
}  
  
fun runFlow() {  
    FlowProcessor().processFlow {  
      this.emit("1234")  
    }  
  FlowProcessor().processFlow(FlowAction.emitOne())  
}
```
не собирался. Теперь можно будет определить `FlowAction` как
```kotlin
interface FlowAction : suspend (FlowCollector<String>) -> Unit
```
Будет полезно для построения своего API для корутин, хотя, если признаться честно, не использовал никогда functions as super-types

## 3. Opt-in для своего API

Можно указывать для своего API OptIn и компилятор будет требовать явного задания OptIn, там где вы собираетесь использовать этот API.
Нужно будет сделать свою аннотацию
```kotlin
@RequiresOptIn(message =  "This API is experimental.")  
@Retention(AnnotationRetention.BINARY)  
@Target(AnnotationTarget.CLASS)  
annotation  class MyDateTime // Opt-in requirement annotation
``` 
Повесить над интересующим обектом
 ```kotlin
 @MyDateTime  
 class DateProvider // A class requiring opt-in
 ```
И при попытке использовать его в клиентском коде компилятор попросить включить OptIn
```kotlin
// Warning: experimental API usage  
fun  createDateSource(): DateProvider {  /* ... */  }  

fun  getDate(): Date {  
  val dateSource =  createDateSource()  // Also warning: experimental API usage  // ...  
}
```
Доступно в 1.5.30. А тут  [полное](https://kotlinlang.org/docs/opt-in-requirements.html#experimental-status-of-the-opt-in-requirements) описание и [тикет](https://youtrack.jetbrains.com/issue/KT-22956)

## 4. Улучшения с рекурсивным генериками 

```kotlin
// Kotlin 1.5.20  
val containerA = PostgreSQLContainer<Nothing>(DockerImageName.parse("postgres:13-alpine")).apply {  
  withDatabaseName("db")  
    withUsername("user")  
    withPassword("password")  
    withInitScript("sql/schema.sql")  
}  
  
// Kotlin 1.5.30  
val containerB = PostgreSQLContainer(DockerImageName.parse("postgres:13-alpine"))  
    .withDatabaseName("db")  
    .withUsername("user")  
    .withPassword("password")  
    .withInitScript("sql/schema.sql")

//Вобщем будем работать для такого описания класса
public class PostgreSQLContainer<SELF extends PostgreSQLContainer<SELF>> extends JdbcDatabaseContainer<SELF> {
```

Источник [KT-40804](https://youtrack.jetbrains.com/issue/KT-40804)


## 5. Смягчили ограничения на builder-interface
Ну например, есть в stdlib вот такая удобная функция
```kotlin
public inline fun <E> buildList(@BuilderInference builderAction: MutableList<E>.() -> Unit): List<E> {
``` 

и вот так сделать нельзя
```kotlin
@OptIn(ExperimentalStdlibApi::class)  
fun builder() {  
   //Builder inference lambda contains inapplicable calls so E cant be inferred. 
   //It could be resolved only with unrestricted builder inference. Please use -Xunrestricted-builder-inference compiler flag to enable it
    buildList {  
        add(11)  
        add(12)  
        set(1, null)  
        val x = get(1)  
        if (x != null) {  
            removeAt(1)  
        }  
    }  
}
```
Взято [отсюда](https://kotlinlang.org/docs/whatsnew1530.html#eliminating-builder-inference-restrictions)

## 6. Создание Annotation class
Аннотации в Java являются интерфейсами, что приводит к двум следствия:
- можно отнаследовать интерфейс
- можно передать интерфейс в качестве параметра функции

Видится, что в kotlin это не особо-то и нужно было (разве что жонглировать рефлексией), но для лучшей interoperability добавили
```kotlin
annotation class InfoMarker(val info: String)  
  
fun processInfo(marker: InfoMarker) = ...  
  
fun main(args: Array<String>) {  
    if (args.size != 0)  
        processInfo(getAnnotationReflective(args))  
    else  
  //тут ошибка Annotation class cannot be instantiated
  processInfo(InfoMarker("default"))  
}
``` 

[Описание от Романа Елизарова](https://youtrack.jetbrains.com/issue/KT-45395)

## 7. Аннотация над типами классов 
```kotlin
@Target(AnnotationTarget.TYPE_PARAMETER)
annotation class Ann

class Foo<@Ann T> {} // now: [UNSUPPORTED] Unsupported [Annotations for type parameters are not supported yet]
```

Эта фича по дефолту будет включена в  1.6. Пока можно использовать опцию  `-language-version 1.6`  начиная с  1.5.30.

Ну и на вопрос нафига (мотивация, типа) приводят такого монстра:
```kotlin
@Property(tries = 3)
fun test(@ForAll list: @Size(5) List<@AlphaChars String>) {
    println(list)
}
```
Ну и как вам мета-программирование, хотели бы жить в таком мире?
Описано в [KT-43714](https://youtrack.jetbrains.com/issue/KT-43714)

## 8. Улучшеный на 23% вывод типов
[Тут](https://youtrack.jetbrains.com/issue/KT-40804) много примеров, что станет лучше

## 9. Multiple receivers
Фишка показалась интересной
```kotlin
interface Extension  
interface Context { // Context is a dispatch receiver  
  fun Extension.f() // Extension is an extension receiver  
}  
 
interface BlockCurrentThreadContext {
   fun <T> Future.await() : T { ...
   }
} 
interface ScheduleWaitInThreadPoolContext {
   fun <T> Future.awai() : T { ....
   }
}
val future: Futue<ByteArray> = ...
runBlockingCurrentThread {
   future.await() //заблокируем текущий поток и подождем
}
runNonBlocking {
  future.await()
}
```
Это мое рассуждение на тему как оно может быть. Если появится, то прям очень круто расширит DSL. Вообще можно почитать ~~PEP-259~~[KEEP-259](https://github.com/Kotlin/KEEP/issues/259), там много интересных концепций накидывают.

## 10. Понаулучшали стандартную библиотеку
### Улучшили работу с Duration
Красиво выводит
|Код| Вывод |
|---|---|
| Duration.days(45).toString() | `45d` |
| Duration.days(1.5).toString() | `1d 12h` |
|Duration.minutes(1230).toString()|`20h 30m`|

И также красиво парсит
```kotlin
val isoFormatString = "PT1H30M"
val defaultFormatString = "1h 30m"
val singleUnitFormatString = "1.5h"
println(Duration.parse(isoFormatString)) // "1h 30m"
println(Duration.parse(defaultFormatString)) // "1h 30m"
println(Duration.parse(singleUnitFormatString)) // "1h 30m"
```

### Добавили полезняшек в Regexp
В Regex добавили matchAt
```kotlin
val releaseText = "Kotlin 1.5.30 is released!"
// regular expression: one digit, dot, one digit, dot, one or more digits
val versionRegex = "\\d[.]\\d[.]\\d+".toRegex()
println(versionRegex.matchesAt(releaseText, 0)) // "false"
println(versionRegex.matchesAt(releaseText, 7)) // "true"
```
Ну и более интересное 
```kotlin
val mixedColor = colorsText.splitToSequence(regex)
```
Естественно, вычисление каждого элемента lazy

Ну и все эти изменения описали [тут](https://kotlinlang.org/docs/whatsnew1530.html#standard-library)

## Еще
Когда-нибудь [запилят](https://youtrack.jetbrains.com/issue/KT-46778) поддержку sealed-классов Java

Ждем [новый компилятор](https://youtrack.jetbrains.com/issue/KT-46756), который "существенно быстрее"

Ну и новый Kotlin Intermediate Representation (IR) - пока посмотреть [не на что](https://kotlinlang.org/docs/whatsnew15.html#stable-jvm-ir-backend) (

