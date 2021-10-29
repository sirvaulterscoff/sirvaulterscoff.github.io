# Принципы написания хороших юнит-тестов

Прежде чем говорить о принципах написания хороших юнит-тестов, дадим определение тому, что же такое "хороший юнит-тест". В этом посте будет масса акронимов, позволяющих запомнить принципы; для определения "хороший юнит-тест" тоже есть свой акроним:

## F.I.R.S.T 
First, как в фразе test first, поэтому запоминается легко.

[**F**]ast - тесты быстрые

[**I**]solated - тесты могут быть запущены в любое время, в любом порядке. Причина для падения теста только одна. Т.е. если тест падает потому, что сломали код и еще по понедельника - так себе тест

[**R**]epeatable - ну тут к с любым экспериментом: тест должен быть воспроизводимый.

[**S**]elf-validating - тест сам (автоматически) проверяет свои результаты. Тут поинт в том, что если у вас тесты не фейлят сборки или для того чтобы понять что все прошло ок нужно заглядывать в логи - вы куда-то не туда свернули

[**T**]imely - тесты пишутся вместе с кодом. Заметьте этот поинт не про фанатиков TDD, а про то, что вы поставляете "единицу изменений" вместе с тестами. А уж TDDить или не TDDить - решать вам

Коротко
-
[**F**]ast - тесты быстрые

[**I**]solated - в любом порядке, только одна причина падения

[**R**]epeatable - воспроизводимые

[**S**]elf-validating - само валидирующиеся

[**T**]imely - тесты пишутся вместе с кодом

## Что будем тестировать
Б&#0243;льшая часть книг и примеров по этим принципам рассматривают "удобные" примеры - арифметические операции, перевод денег между счетами. Я решил взять какой-то более приземленный пример. 

Наш ландшафт - приложение, которое получает из Kafka сообщение с числовым ключом и строковым значением, в формате Json. Значение валидируется по схеме, затем из БД получаем хранимую сущность - запрос. Ключ сообщения = первичный ключ в таблице. 

Ответ может содержать информацию о том, валидны ли запрошенные данные и на какую дату (паспорт валиден, паспорт невалиден, к примеру)

Ответ может содержать информацию об ошибке и в этом случае, считаем, что информации о том валидны ли документы у нас нет.

Задача покрыть наш функционал достаточным количество тест-кейсов

Приведу фрагмент метода обработки сообщения

```kotlin
@Suppress("unused")  
fun processIncomingMessage(key: String, message: IncomingMessage) {
   //если сообщение успешное, то переносим данные в обхект в БД  
    message.takeIf(::isSuccess)?.apply {  
	  processNewMessage(key, this)  
	  //иначе, обрабатываем его как ошибочное
    } ?: handleFailureMessage(key, message)  
}  
  
private fun processNewMessage(key: String, message: IncomingMessage) {  
    // ключ приводим к типу Long и валидируем
    val keyLong = key.toLongOrNull() ?: throw IllegalArgumentException("Ключ $key не является числом")  
    // загружаем сообщение из БД по ключу
    validationRequestRepository  
		.findById(keyLong)  
        ?.let { existing ->  
        //если нашлось то обноив ему статус
				val newState = when(existing.status) {
				//если этот запрос ожидал ответа или ранее завершился с ошибкой, то перенес данные из ответа  
                ValidationStatus.WAITING -> computeValidity(message)  
                ValidationStatus.FAILED ->  computeValidity(message)  
                // в остальных случаях это какой-то out-of-order ответ
                else -> handleOutOfOrderMessage(message, existing)  
            }  
            validationRequestRepository.setStatus(keyLong, newState )  
        } ?: throw RecordNotFoundException(key)  
}
```

### Первый тест

```kotlin
class IncomingMessageServiceSpec : FreeSpec() {  
    val repository = mockk<ValidationRequestRepository>()  
    val service = IncomingMessageService(repository)  
    init {  
        "verify message processed" {  
		    val id = Random.nextLong()  
            every { repository.findById(id) } returns ValidationRequest(id, ValidationStatus.WAITING)  
            every { repository.setStatus(id, ValidationStatus.VALID) } returns ValidationRequest(id, ValidationStatus.VALID)  
            val validityOn = LocalDateTime.now()  
            service.processIncomingMessage(id.toString(), IncomingMessage(IncomingMessageStatus.OK, response = ExternalServiceResponse(true,  
                validityOn  
            ), error = null))  
            verify(exactly = 1) { repository.setStatus(id, ValidationStatus.VALID) }  
 }  }  
}
```

Насколько он хорош? Ну выглядит так, что соответствует всем критериям RIGHT. Но давайте посмотрим, какие кейсы он на самом деле НЕ покрывает.  Как понять, что мы протестили достаточное количество кейсов? В этом нам помогут следующие принципы:

### Right BICEP
(на самом деле даже не принципы, а скорее набор вопрос для самоконтроля)
Суть проста:
**Right** - а результаты вообще правильные?
**B**oundaries - а все граничные условия?
**I**nverse -
**C**ross-check
**E**error codnitions
**P**erformance

#### Boundaries

Пожалуй самый объемный пункт, включающий в себя кучу всевозможных (и, конечно, всеневозможных проверок): передать какие-то кривые значения, несвязанные данные, максимальные и минимальные значения. 
Вобщем-то в такой формулировке он максимально непонятен (или скорее слишком широк). Поэтому есть еще один акроним-чеклист, который позволяет проверить все ли граничные условия мы учли - **CORRECT**

**C**onformance  - значения соответствуют формату?

**O**rdering - а в том ли порядке передаем, получаем значения?

**R**ange - лежат ли значения между минимум и максимумом?

**R**eference - работает ли код с чем-то внешним, чем код напрямую не управляет?

**E**xistence - существует ли значение?

**C**ardinality - достаточное ли количество значений?

**T**ime - а все ли происходит в нужном порядке? В нужное время?

Итак, на примере нашего сервиса мы можем разработать следующий набор сценариев

- Conformance - проверить обработку кривого JSON
- Ordering - принять ответ для запроса, которого еще нет в БД. Вполне реальный кейс с учетом наличия лага между отправкой в кафку и фиксацией транзакции (если вдруг у нас отправка запросов именно так реализована). В этом случае ответ может придти до того, как мы запрос успели закомитить в БД.
- Range - передать отрицательный идентификатор, очень длинную строку ошибки
- Reference - проверим, что ничего не отработает, если методы репозитория кидают исключения
- Existence - проверим "невозможные сценарии", когда положительный ответ, содержит ошибку или ошибочный ответ содержит данные
-  Cardinality - проверим порядки величин. Например, что очень длинная строка об ошибке обрежется до максимальной длины колонки в БД.
- Timed - проверим, что если дата валидации находится далеко в прошлом (тут вопрос насколько?), то результат проверки не может быть валидным

#### Inverse
Иногда мы можем проверить поведение применяя логическую или арифметическую инверсию.

#### Cross-checking
Кросс-валидация результата. Очень полезный пункт, который хорошо закрывает сломы инфраструктуры тестового кода. Т.е. нужно проверить результат каким-то другим способом. В нашем кейсе это может означать необходимость написать интеграционный тест, который сохраняет значения во встроенную БД. А тест, затем убеждается с помощью запроса в БД, что там оказались все нужные данные

#### Error conditions
Нужно проверить все возможные unhappy path

#### Performance 
Проверить соответствие производительности функционала каким-то ожиданиям. Обычно достаточно просто таймаутов для тестов, чтобы не получить зависшую намертво сборку. Но иногда есть кейсы, когда мы хотим быть уверены, что время исполнения какого-то поведения не выйдет за разумные пределы. Это может быть важно, например, если проверяем код имеет высокую связанность с другими частями системы (т.е. причин для изменения поведения много) и при этом важно, чтобы его время исполнения находилось в определенных пределах. Ну например, потому что мы можем упереться в таймаут транзакции, если метод выполняется слишком долго.

Ну теперь по порядку
#### [B]icep [C]orrect
```kotlin

"does not process invalid json" {  
  assertThrows<Exception> {  
  kafkaListener.receiveMessage("null", "{")  
    }  
}
```

#### [B]icep C[O]rrect
```kotlin
"(O)rdering - правильный ли порядок" - {  
  "receiving response before sending leads to error" {  
	  //arrange  
	  every { repository.findById(404L) } returns null  
  
	  //act  
	  val executable = {  
	  service.processIncomingMessage(  
                404.toString(), IncomingMessage(  
                    IncomingMessageStatus.OK, response = ExternalServiceResponse(  
                        true,  
                        LocalDateTime.now()  
                    ), error = null  
  )  
            )  
        }  
  
	  //assert  
	  assertThrows<RecordNotFoundException>(executable)  
    }  
  // если ваши методы работаю с коллекциями, либо возвращают их, то тут  
 // логично проверить передачу/возврат значений в разном порядке}
 ```

#### [B]icep - Co[R]rect
```kotlin
"(R)ange - Все ли диапазоны значений проходят" - {  
  for (id in listOf(-1L, Int.MAX_VALUE.toLong())) {  
        "verify $id" {  
		  positiveCase(id)  
        }  
  }  
    //Эта проверка также касается возвращаемых величин - следует проверить, что они находятся в каких-то адекватных пределах  
  
}
```

#### [B]icep - Cor[R]ect
```kotlin
"(R)eference - какие зависимости есть у кода и что если они не работают?" - {  
	  //arrange  
	  val expectedId = 505L  
	  every { repository.findById(expectedId) } throws RuntimeException("База данных недоступна")  
  
	    //act  
	  val executable = {  
	  service.processIncomingMessage(  
            expectedId.toString(), IncomingMessage(  
                IncomingMessageStatus.OK, response = ExternalServiceResponse(  
                    true,  
                    LocalDateTime.now()  
                ), error = null  
		  )  
        )  
    }  
  
	  //assert  
	  assertThrows<RuntimeException>(executable).message shouldBe "База данных недоступна"  
	  verify {  
		  repository.findById(withArg { it shouldBe expectedId })  
	    }  
}
```

#### [B]icep - Corr[E]ct
```kotlin
"(E)xistence - проверка несуществующих данных" - {  
  val positiveWithErrorFilled = normalResponse {  
	  ExternalServiceError("111", "Error", "Rejected just because", ByteArray(0))  
    }  
  val negativeWithStatusFilled = errorResponse(buildResponse = {  
	  ExternalServiceResponse(true, LocalDateTime.now())  
    })  
  
    "positive response with error filled" {  
	  testCase(101, positiveWithErrorFilled, ValidationStatus.VALID)  
    }  
  "negative response with normal reponse filled" {  
	  testCase(102, negativeWithStatusFilled, ValidationStatus.FAILED)  
    }  
}
```

#### [B]icep - Corre[C]t
```kotlin
"(С)ardinality - проверка порядка величин" {  
  val negativeWithVeryLongErrorCode = errorResponse(buildError = {  
	  ExternalServiceError("1".repeat(4096), "Error".repeat(1000), "2".repeat(2000), ByteArray(0))  
    })  
    testCase(103, negativeWithVeryLongErrorCode, ValidationStatus.FAILED) {  
	 it.failReason!!.length shouldBeLessThan 256  
  }  
  //Не забываем также, что нужно проверять и допустимые диапазоны  
}
```

#### [B]icep - Correc[T]
```kotlin
"(T)ime - привязан ли код ко времени" {  
  val positiveWithValidityInThePast = normalResponse(buildResponse = {  
	  ExternalServiceResponse(true, LocalDateTime.now().minusYears(200))  
    })  
    testCase(104, positiveWithValidityInThePast, ValidationStatus.INVALID)   
}
```

#### Bi[C]ep 
```kotlin
"[C]ross-checking using other means. Можем ли проверить результат, используя другие средства" {  
	  //тут мог бы быть интеграционный тест, который после  
	  // обработки заглядывает в БД с помощью SQL запроса, если бы я его написал  
	 testCase(105, normalResponse(), ValidationStatus.VALID)  
     //select status from ValidationRequest where id=105  
}
```

### Bic[E]p
```kotlin
"[E]rror conditions - проверьте все unhappy-path" - {  
  //arrange  
  for (id in listOf("null", "", "123a", " 544", "678 ")) {  
        "checking id $id" {  
  //act  
  val executable = { service.processIncomingMessage(id, normalResponse()) }  
  
  //assert  
  assertThrows<IllegalArgumentException>(executable).message shouldBe "Ключ $id не является числом"  
  verify(inverse = true) {  
	   repository.findById(any())  
       repository.saveValidationRequest(any())  
   }
  }  
 }  
}
```

#### Bice[P]
```kotlin
"[P]erformance - все ли норм с производительностью" {  
  measureTimeMillis {  
	  testCase(106, normalResponse(), ValidationStatus.VALID)  
  
    } shouldNotBeGreaterThan 3_000L  
}
```
