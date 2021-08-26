# ФП это не то, что мы думаем

## Как мы обрабатываем null значения?

В kotlin есть elvis-оператор и null-safe reference. Если в запросе передали externalId то мы загружаем пользователей, иначе возвращаем BAD REQUEST 
```kotlin
@GetMapping("/api/v1/users")  
fun listUsers(@RequestParam("externalId") externalId: String?) : ResponseEntity<List<User>> {  
    return externalId?.let {  
			 ResponseEntity.ok(usersRepository.findUsersByExternalId(externalId))  
    } ?: ResponseEntity.badRequest().build()  
}
```

Чуть усложним пример - предположим, что externalId может содержать несколько значений
```kotlin
@GetMapping("/api/v1/users")  
fun listUsers(@RequestParam("externalId") externalId: String?) : ResponseEntity<List<User>> {  
    return externalId?.let {  
		 it.split(",")  
    }?.flatMap {  
		  usersRepository.findUsersByExternalId(it)  
    }?.let {  
		  ResponseEntity.ok(it)  
    } ?: ResponseEntity.badRequest().build()  
}
```

В коде нам приходится по цепочке тянуть ?.

А теперь еще добавим деталей - будем упаковывать ответ в UsersResponse

```kotlin
@GetMapping("/api/v1/users")  
fun listUsers(@RequestParam("externalId") externalId: String?) : ResponseEntity<UsersResponse> {  
    val response =  externalId?.let {  
		 it.split(",")  
    }?.flatMap {  
		  usersRepository.findUsersByExternalId(it)  
    }?.let {  
		  UsersResponse(it, ResponseStatus.PROCESSED)  
    } ?: UsersResponse(emptyList(), ResponseStatus.FAILED)  
      
    return ResponseEntity.ok(response)  
}
```

В целом у меня следующие претензии к этому подходу:

- ?. расползаются по всей цепочке вызовов, не увеличивая читаемость (да блин, я знаю что тут возможно null значение)
- ?: располагается строго в конце и сложно его использовать когда подставить дефолтное значение требуется в середение цепочки

## Посмотриим optional

По моим наблюдениям 90% используют optional следующим образом
```kotlin
 @GetMapping("/api/v1/users")  
    fun listUsers(@RequestParam("externalId") externalId: Optional<String>) : ResponseEntity<UsersResponse> {  
        val response = if (externalId.isPresent) {  
            val users = externalId.get()  
                .split(",")  
                .flatMap {  
				  usersRepository.findUsersByExternalId(it)  
                }  
			  UsersResponse(users, ResponseStatus.PROCESSED)  
        } else {  
            UsersResponse(emptyList(), ResponseStatus.FAILED)  
        }  
        return ResponseEntity.ok(response)  
    }  
}
``` 

Естественно, это не то, как задумывалось использовать Optional. Обратите внимание, что как и коллекции Optional имеет методы map/flatMap
```kotlin
@GetMapping("/api/v1/users")  
fun listUsers(@RequestParam("externalId") externalId: Optional<String>) : ResponseEntity<UsersResponse> {  
    return externalId.map {  
		 it.split(",")  
    }.map { listOfIds ->  
		  listOfIds.flatMap { id ->   
			 usersRepository.findUsersByExternalId(id)  
        }  
    }.map { users ->   
	 UsersResponse(users, ResponseStatus.PROCESSED)  
    }.or {  
	  Optional.of(UsersResponse(emptyList(), ResponseStatus.FAILED))  
    }.map {  
  ResponseEntity.ok(it)  
    }.get()  
}
```
Тут несколько  моментов:

- мы можем предоставить какое-то дефолтное значение в середине цепочки
- но его почему-то нужно заворчивать в Optional
- мы собрали итоговое значение одной цепочкой
- но в конце почему-то нужно использовать get
- мы не можем преобразовать Optional -> Stream (вру, можем, но оно превращается в стрим с единичным значением), поэтому растет вложенность

## Arrow
Посмотрим, какие средства предоставляет arrow

```kotlin
    @GetMapping("/api/v1/users")  
    fun listUsers(@RequestParam("externalId") externalId: String?) : ResponseEntity<UsersResponse> {  
        return ResponseEntity.ok(externalId.splitOrEmpty().fold( {  
		  UsersResponse(it.flatMap { usersRepository.findUsersByExternalId(it) }, ResponseStatus.PROCESSED)  
        }, {it}))  
}  
  
  
fun String?.splitOrEmpty() : Either<List<String>, UsersResponse> {  
    return this?.let {  
	  split(",")  
    }?.let {  
	  Either.Left(it)  
    } ?: Either.Right(UsersResponse(listOf(), ResponseStatus.FAILED))  
}
``` 

Тип Either всегда содержит только одно значение (левое или правое)

Еще один вариант - на корутинах
```kotlin
@GetMapping("/api/v1/user/{id}/location")  
suspend fun getUserLocation(@PathVariable("id") id: String?): ResponseEntity<Location> {  
    return nullable {  
	  val _id = id.bind()  
        val user = usersRepository.findUserById(_id).bind()  
        val location = user.location().bind()  
        ResponseEntity.ok(location)  
    } ?: ResponseEntity.notFound().build()  
}
``` 

### Композиция функций
```kotlin
class UsersController(  
    private val usersRepository: UsersRepository  
) {  
  
    @GetMapping("/api/v1/users")  
    fun getUsers(@RequestParam("id") id: String?): ResponseEntity<List<User>> {  
        val find = usersRepository::findUsersByExternalIds compose String::splitByComa  
        val optionalFind = Option.lift(find)  
        val users = Option<List<User>>::toReponseEntity compose optionalFind  
        return users(Option.fromNullable(id))  
    }  
      
}  
  
fun String.splitByComa() = this.split(",")  
  
fun Option<List<User>>.toReponseEntity(): ResponseEntity<List<User>> {  
    return this.fold({  
  ResponseEntity.notFound().build()  
    }, {  
  ResponseEntity.ok(it)  
    })  
}
```

### Частичное применение функций

```kotlin
class UsersController(  
    private val usersRepository: UsersRepository  
) {  
  
    @GetMapping("/api/v1/users")  
    fun getUsers(@RequestParam("id") id: String?): ResponseEntity<List<User>> {  
        val find =   usersRepository::findUsersById compose String::toListOfInts  
        return Option.lift(find).partially1(Option.fromNullable(id)).invoke().toReponseEntity()  
    }  
}  
  
fun String.toListOfInts() = this.splitByComa().map(String::asInt)  
  
fun String.asInt() = this.toInt()  
  
fun String.splitByComa() = this.split(",")  
  
fun Option<List<User>>.toReponseEntity() : ResponseEntity<List<User>> {  
    return this.fold({  
  ResponseEntity.notFound().build()  
    }, {  
  ResponseEntity.ok(it)  
    })  
}
```

### Карирование
```kotlin
class UsersController(  
    private val usersRepository: UsersRepository  
) {  
  
    @GetMapping("/api/v1/users")  
    fun getUsersByIdAndLocation(@RequestParam("id") id: String?, location: Location): ResponseEntity<List<User>> {  
        val find =   usersRepository::findUsersByExternalIdsAndLocation  
            .swap()  
            .curried()  
            .invoke(location)  
        return Option.lift(find).partially1(Option.fromNullable(id?.splitByComa())).invoke().toReponseEntity()  
    }  
}  
  
fun <A, B, R> ((A, B) -> R).swap() = { b: B, a: A -> this(a,b)}  
fun String.toListOfInts() = this.splitByComa().map(String::asInt)  
  
fun String.asInt() = this.toInt()  
  
fun String.splitByComa() = this.split(",")  
  
fun Option<List<User>>.toReponseEntity() : ResponseEntity<List<User>> {  
    return this.fold({  
  ResponseEntity.notFound().build()  
    }, {  
  ResponseEntity.ok(it)  
    })  
}
```
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjA3MDQ4MDQ4OF19
-->