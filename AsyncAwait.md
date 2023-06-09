
## Вступление

Ассинхронщина занимает неотьемлемую часть написания кода на C# но к сожалению мало кем хорошо обьясняется. Даже на православном метаните написанны какие то туманные , хотя и верные изречения, не дающие в полной мере понять, что такое эти ваши асинки. Цель этой статьи сформировать у начинающего програмиста ментальную модель происходящего, на которую можно будет опереться в решении связанных с ассинхронностью проблем. 

Alert: Для простоты понимания в некоторых местах придется ссылаться на _полуправду_, эти высказывания будут помечены косым текстом. Полагается что читатель углубится в __полностьюправдивое__ значение этих высказываний в зависимости от своих интересов и практической необходимости.

Alert2: Чтоб полностью понять материал рекомендуется копировать код из примеров и запускать его у себя на машине, модифицировать его, играться, пробовать ломать. Мы же хотим уметь пользоваться асинками, а не только знать теорию, верно?

## Task и паралельное выполнение кода

Поскольку асинки крепко завязаны на паралельном выполнении кода, рассмотрим сначала его.

В C# есть поддержка многопоточного программирования и одним из классов для этого является Task. Чтоб запустить метод паралельно с текущим, нужно создать обьект класса Task передав в его конструктор желаемый метод(это может быть любой метод, статический, не статический, делегат, лямбда, Action, или что либо еще, не важно). И вызвать метод Start().
```c#
Task task = new Task(SomeMethod);
task.Start();
```
Акцентирую внимание, что SomeMethod начнет свое выполнение только после вызова Start();
до него, метод просто хранится в обьекте task.

Напишем простую программу которая паралельно методу Main запустит еще один.

```c#
class Program
{
	static void TaskMethod()
	{
		for(int i = 0; i<10000; i++)
			Console.WriteLine("Я выполняюсь одновременно с методом Main");
	}

	static void Main()
	{
		Task task = new Task(TaskMethod);
		task.Start();
		
		for (int i = 0; i < 10000; i++)
			Console.WriteLine("Я метод Main");
	}
}
```

Если запустить этот код, в консоль будет поочередно выводится то "Я метод Main", то "я выполняюсь одновременно с методом Main".  Почему мы не получили сначала 10000 "Я выполняюсь одновременно с методом Main" и только потом 10000 "Я метод Main"?

Завершение работы task.Start() совсем не означает что TaskMethod закончил исполнение, это только сигнал о том, что _c# понял нашу прозьбу_ начать исполнять TaskMethod паралельно текущему методу и сделал это. С этого момента методы Main и TaskMethod работают одновременно.



В процессе написания более сложных програм, может возникнуть необходимость дождаться когда паралельный метод завершит свою работу, скажем необходимо модифицировать текущую програму, чтоб после того как все строки выведутся в консоль, вывести "все строки выведены".

Если в конец метода Main просто добавить Console.WriteLine("все строки выведены"), эта строка выведется последней, как и планировалось, а может и не последней, возможно где то между "я метод..." и "я выполняюсь....". Одним словом, сейчас сложно сказать в какой момент TaskMethod точно закончит свою работу.  

Для гарантии завершения работы Task, есть метод Wait();
```c#
Task task = new Task(SomeMethod);
task.Start();
// сейчас метод SomeMethod выполняется паралельно текущему
task.Wait();
// сейчас метод SomeMethod уж точно закончил свою работу
```

То есть с учетом новых требований изначальную программу можно переписать следующим образом:

```c#
class Program
{
	static void TaskMethod()
	{
		for(int i = 0; i<10000; i++)
			Console.WriteLine("Я выполняюсь одновременно с методом Main");
	}

	static void Main()
	{
		Task task = new Task(TaskMethod);
		task.Start();

		for (int i = 0; i < 10000; i++)
			Console.WriteLine("Я метод Main");
			
		task.Wait();
		
		Console.WriteLine("Все строки выведены");
	}
}
```

Еще один случай использования метода Wait() - наличие возвращаемого значения. Скажем есть задача обработать данные с сервера. Очевидно что запрос к серверу требует обращения к сети, ожидания ответа от сервера и так далее и так далее. Это небыстрая операция. Но поскольку результат этой операции пригодится в будущем можно заранее запустить запрос паралельно паралельно. Во время его выполнения заняться другими не менее важными делами, а когда настанет время использовать результат, запрос либо уже обработается, либо будет близок к этому, а значит общее время работы програмы сократиться.

```c#
class Program
{
	static int GetUsedIdFromServer(){ return 4; }

	static void Main()
	{
		Task getUserIdTask = new Task(GetUsedIdFromServer);
		getUserIdTask.Start();

		Console.WriteLine("делаем не менее важные дела");
			
		getUserIdTask.Wait();
		
		int userId = ???;
		Console.WriteLine("айди пользователя:" + userId);
	}
}

```

Эта программа не скомпилируется по двум причинам.
Во-первых, кто вообще придумал присваивать три знака вопроса вместо кода который должен достать результат выполнения getUserIdTask. 
Во-вторых, чтоб Task поддерживал методы с возвращемым значеним, ему нужно прямо указать об этом, иначе он не сможет их обрабатывать. 

У класса Task есть дженерик вариация Task\<ResultType> добавляющая поддержку возвращаемых значений. Он добавляет 
```c#
ResultType ?Result{get; }; 
```
свойство к обычному Task классу. 

С имеющейся информацией дописываем программу до рабочего состояния.

```c#
class Program
{
	static int GetUsedIdFromServer(){ return 4; }

	static void Main()
	{
		Task<int> getUserIdTask = new Task<int>(GetUsedIdFromServer);
		getUserIdTask.Start();

		Console.WriteLine("делаем не менее важные дела");
			
		getUserIdTask.Wait();
		
		int userId = getUserIdTask.Result;
		Console.WriteLine("айди пользователя:" + userId);
	}
}

```

Важные замечания:
Task и Task\<ResultType> не поддерживает передачу параметров, для обхода этого ограничения используют лямбда-замыкания

```c#
class Program
{
	static int GetUsedIdFromServer(string name){ return 4; }

	static void Main()
	{
		string name = "SomeUserName";
		Task<int> getUserIdTask = new Task<int>(() => GetUsedIdFromServer(name));
	}
}
```


Task\<ResultType>.Result будет доступен только после завершения работы его метода, до этого Result равен null. Для уверенности в наличии Task\<ResultType>.Result следует вызывать Task.Wait();


## Замечая замешательство

Я думаю каждый писал определение ассинхронного метода. Если вы не писали, то вот пример

```c# 
static async Task DoingStuffAsync(){
    Console.WriteLine("занимаемся асинками");
}

static async Task<int> AlsoStuffAsync(){
    return 5;
}

```

Мы знаем что ассинхронной метод содержит ключевое слово async в своем заголовке. Он почему то должен возвращать Task, а в случае возвращаемого значения, его тип нужно вписать в треугольные скобки возле таск. Ну и еще иногда нужно дописывать ключевое слово await при его вызове.  

## Всё во что я верю - ложь

Временно откинем ключевые слова async и await. Почему возвращаемый тип метода это Task\<int> но при этом можно вернуть просто int, без всяческий new Task(){Result = 5}; и прочих конструкций? Какая то магия, не находите?)

Всё дело в том что на самом деле ассинхронный метод _состоит из двух методов_. К примеру, в случае AsloSuffAsync компилятор c# без вашего метода _сгенерирует два метода_.
```c# 
static int AlsoStuffAsync_AsyncMethodImplementation(){
    return 5;
}

static Task<int> AlsoStuffAsync(){
    Task<int> task = new Task<int>(AlsoStuffAsync_AsyncMethodImplementation);
    task.Start();
    return task;
}
```

Первый из них, не смотря на свое странное имя, действительно будет содержать весь код метода. Назовем этот метод реализацией ассинхронного метода. Он уже не носит титул ассинхронного, это самый обычный метод.

Второй метод - вспомогательный. Его суть в том, чтоб создать и запустить таск, передав в него реализацию ассинхронного метода. Благодаря тому что вспомогательный метод имеет такое же имя что и изначальный ассинхронный метод, его вызов синтаксически никак не отличается от вызова самого обычного метода. Кстати вспомогательный метод тоже _является самым обычным методом без всяких асинков_.

Получается что ключевое слово async просто сигнализирует компилятору, чтоб он сгенерировал не один, а два метода по вышеописаному принципу. Из этого же и следует то что его тип возвращаемого значения обязательно должен быть Task. По другому просто не получилось бы достать возвращаемое значение.

Зачем так усложнять? Для упрощения. Разработчики современных языков программирования стараются подмечать языковые конструкцие, которые пользователи языков постоянно повторяют, и стараются их успростить. Согласитесь, понимая что ключевое слово async создано для того чтоб компилятор без вас нагенерировал дополнительный метод, начинаешь понимать что в целом класс, удобно, не нужно постоянно писать то, что вместо меня напишут во вспомогательном методе. 

## Многообразие ожидания

Вызов ассинхронного метода никак не отличается от обычного, просто передал параметры и всё.
```c#
Task<int> task = AlsoStuffAsync();
```
И ожидаемо, поскольку вспомогательный метод вернет Task, его результатом тоже будет Task, поэтому на него накладываются все ограничения которые нужно соблюдать для уверенности в том, что метод закончил свое выполнение и .Result is not null.

При этом можно спокойно запустить несколько ассинхронных методов и подождать их выполнения как нибудь потом.

```c#
Task<int> task1 = AlsoStuffAsync();
Task<int> task2 = AlsoStuffAsync();
Task<int> task3 = AlsoStuffAsync();

task1.Wait();
task2.Wait();
task3.Wait();
```

или использовать чуть более элегантную функцию для ожидания нескольких тасок

```c#
Task<int> task1 = AlsoStuffAsync();
Task<int> task2 = AlsoStuffAsync();
Task<int> task3 = AlsoStuffAsync();

Task.WaitAll(task1, task2, task3);
```

Неудобство тут только в том, что если необходимо мгновенно ждать результат после выполнения ассинхронного метода, код становится слишком длинным и разработчик вынужден каждый раз писать одно и то же
```c#
Task<int> task1 = AlsoStuffAsync();
task1.Wait();
int result = task1.Result;
```

Слишком много кода. А если программа потребует большого колличества вызовов ассинхронных методов, в одном месте, разработчик вынужден бодаться в дополнительных переменных.
```c#
Task<int> task1 = GetSomethingFromServer();
task1.Wait();
int id = task1.Result;

Task<string> task2 = GetUserNameFromId(id);
task2.Wait();
string name = task2.Result;
...
// и так далее
```

Очевидным решением было бы вынести повторяющийся код в метод, скажем вот такой

```c# 
T WaitAndGetResult<T>(Task<T> task){
	task.Wait();
	return task.Result;
}
```

Теперь предыдущий код можно отрефакторить следующим образом

```c#
int id = WaitAndGetResult(GetSomethingFromServer());
string name = WaitAndGetResult(GetUserNameFromId(id));
...
// и так далее
```
Намного лучше!

Разработчики c# посмотрели на это, почесали репу и пошли еще дальше, представив ключевое слово await, которое __идейно__ делает тоже самое что и WaitAndGetResult

```c#
int id = await GetSomethingFromServer();
string name = await GetUserNameFromId(id);
...
// и так далее
```

Причем не обязательно awaitить только ассинхронные функции, это можно делать с любым таском

```c#
Task<int> task1 = GetSomethingFromServer();
Task<string> task2 = GetSomethingElseFromServer();

int something = await task1;
int somethingElse = await task2;
```

На самом деле в ключевом слове await заложен чуть более глубокий смысл, но о нем позже, для базового понимания асинков хватит и простого WaitAndGetResult.