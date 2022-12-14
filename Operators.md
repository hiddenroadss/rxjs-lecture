# Оператори

RxJS в основному корисний своїми операторами, навіть якщо Об'єкт Спостереження є основою.

Оператори є основними елементами, які дозволяють легко складати складний асинхронний код декларативним способом.

## Що таке оператори?

Оператори - це функції, які повертають Об'єкт Спостереження.

Їх існує два типи:

1. Оператори створення - інкапсулють всередині логіку створення Об'єкту Спостереження і повертають його.
2. Оператори трансформації - приймають аргументом один Об'єкт Спостереження, підписуються на нього, трансформують логіку і повертають інший.

Поговоримо спочатку про оператори створення, так як їх легше зрозуміти. Ми з вами вже створювали Об'єкти Спостереження за допомогою `new Observable(...)` і можна помітити, що це вимагає від нас написання досить великого шматка коду.

Для нашої зручності в RxJS є оператори створення - функції, які інкапсулюють всередині логіку по створенню Об'єкту Спостереження і дають нам писати код декларативно.

```js
const observable$ = new Observable((subscriber) => {
	subscriber.next(1);
	subscriber.next(2);
	subscriber.next(3);
	subscriber.complete();
});

// Працює аналогічно
const observable2$ = of(1, 2, 3);
```

```js
function fromEvent(element, event) {
	return new Observable((subscriber) => {
		const callback = (e) => {
			subscriber.next(e);
		};
		element.addEventListener(event, callback);

		return function () {
			element.removeEventListener(event, callback);
		};
	});
}

function of(...args) {
	return new Observable((subscriber) => {
		args.forEach((item) => {
			subscriber.next(item);
		});
		subscriber.complete();
	});
}
```

Ще однією перевагою використання функцій є можливість замикання. Можемо розглянути приклад з інтервалом:

```js
function interval(intervalInMs) {
	let count = 0;
	return new Observable((subscriber) => {
		const intervalRef = setInterval(() => subscriber.next(count++), intervalInMs);

		return function () {
			clearInterval(intervalRef);
		};
	});
}
```

В даному випадку нам потрібно зберігати десь змінну `count` і замикання для цього ідеально підходить.

Після того, як ми розглянули приклади того, як би ми створювали ці Об'єкти Спостереження з нуля, ми можемо оцінити те, що бібліотека зробила більшість роботи за нас і все, що нам потрібно - це викликати правильний оператор:

```js
import { interval, fromEvent, ajax } from 'rxjs';
const interval$ = interval(1000);

const button = document.queryElement('.button');
const buttonClicks$ = fromEvent(button, 'click');

const apiRequest$ = ajax('someurl');
```

### Оператори трансформації

Оператори трансформації приймають аргументом один Об'єкт Спостереження, підписуються на нього і повертають інший Об'єкт Спостереження трансформувавши в ньому логіку.

```ts
function increment(source: Observable<any>) {
	return new Observable((subscriber) => {
		const sub = source.subscribe({
			next: (data) => subscriber.next(data + 1),
			error: (err) => subscriber.error(err),
			complete: () => subscriber.complete(),
		});

		return () => {
			sub.unsubscribe();
		};
	});
}
```

Як же ми можемо його використовувати? Давайте почнемо з першого, що приходить в голову:

```ts
const observable$ = of(1, 2, 3);

const incremented$ = increment(observable$);

// Виведе 2, 3, 4
incremented$.subscribe(console.log);
```

Як і написано в визначенні, ми просто передали оператору аргументом вхідний Об'єкт Спостереження і отримали новий, трансформований Об'єкт Спостереження.

В чому недоліки такого підходу?

Давайте створимо ще один оператор `multiplyByTwo`:

```ts
function multiplyByTwo(source: Observable<any>) {
	return new Observable((subscriber) => {
		return source.subscribe({
			next: (data) => subscriber.next(data * 2),
			error: (err) => subscriber.error(err),
			complete: () => subscriber.complete(),
		});
	});
}

const observable$ = of(1, 2, 3);

const result$ = multiplyByTwo(increment(observable$));

// Виведе 4, 6, 8
result$.subscribe(console.log);
```

В даному випадку ми хочемо спочатку збільшити число на одиницю, а потім помножити число на два. Як це працює? Джаваскрипт бачить функцію і її виклик. Щоб розпочати виконання йому треба передати аргументи, які ми туди поставили. В нашому випадку він не може це зробити одразу, тому що ми передали не об'єкт, рядок, функцію, тощо, а _виклик_ функції. Для того, щоб отримати результат йому потрібно її виконати. Отже він спочатку виконує `increment(observable$)`. В цьому випадку аргументом ми передаємо вже не виклик функції, тому вона відпрацьовує і повертає нам новий Об'єкт Спостереження, який буде інкрементувати число перед віддачею його підписнику. Тепер у нас у виклику функції `multiplyByTwo` не інший виклик функції, а Об'єкт Спостереження і ми можемо викликати її. Отримуємо новий Об'єкт Спостереження, який додатково буде множити число на 2 перед віддачею його підписнику.

Якщо додати ще один оператор вийде це:

```js
const observable$ = of(1, 2, 3);

const result$ = increment(multiplyByTwo(increment(observable$)));

// Виведе 5, 7, 9
result$.subscribe(console.log);
```

<!-- До речі, тепер ви бачите, чому саме оператор - це функція, яка приймає один аргумент - вхідний Об'єкт Спостереження і завжди віддає новий Об'єкт Спостереження. Це дозволяє нам робити ланцюги з операторів -->

Згодьтеся, більшості людей це буде досить складно читати, так як виконання йде справа наліво (або зсередини назовні). Хоча це один з варіантів виконання функцій в функціональному програмуванні і до цього легко звикнути. Але є інший аспект, який робить роботу з таким підходом майже неможливою.

В наших прикладах з операторами `increment` i `multiplyByTwo` ми використовували максимально просту логіку. В реальному світі ми будемо створювати щось гнучкіше, наприклад оператор `multiply`, який буде приймати аргументом число на яке нам треба помножити інше.

Як же ми можемо це вирішити, якщо ми знаємо, що оператор _має_ приймати один аргумент - вхідний Об'єкт Спостереження?

Наша відповідь - замикання:

```ts
function multiply(multiplier: number) {
	return function (source: Observable<number>) {
		return new Observable((subscriber) => {
			return source.subscribe({
				next: (data) => subscriber.next(data * multiplier),
				error: (err) => subscriber.error(err),
				complete: () => subscriber.complete(),
			});
		});
	};
}
```

Як це працює?:

```ts
const observable$ = of(1, 2, 3);

const result$ = multiply(2)(increment(observable$));

// Виведе 4, 6, 8
result$.subscribe(console.log);
```

Джаваскрипт бачить функцію `multiply` і її виклик з аргрументом `2`. Викликає її і отримує назад функцію, яка приймає вхідний Об'єкт Спостереження і віддає новий. Єдина відмінність в тому, що в замиканні цієї функції вже є змінна `multiplier`, яку ми і будемо використовувати далі `subscriber.next(data * multiplier)`. Ось і вся магія.

Для повноти експерименту давайте зробимо оператор `increment` також гнучким:

```ts
function incrementBy(num: number) {
	return function (source: Observable<number>) {
		return new Observable((subscriber) => {
			return source.subscribe({
				next: (data) => subscriber.next(data + num),
				error: (err) => subscriber.error(err),
				complete: () => subscriber.complete(),
			});
		});
	};
}
```

тоді наш виклик буде:

```ts
const observable$ = of(1, 2, 3);

const result$ = multiply(2)(incrementBy(5)(observable$));

// Виведе 12, 14, 16
result$.subscribe(console.log);
```

Погодьтеся, даний код читати дуже-дуже складно. А це всього лише 2 оператори. Рішення цієї проблеми ми можемо знайти також у функціональному програмуванні.

```ts
const observable$ = of(1, 2, 3);

const result$ = observable$.pipe(incrementBy(5), multiply(2));

// Виведе 12, 14, 16
result$.subscribe(console.log);
```

<!-- Pipable operators – це тип, який можна передати в Об'єкт Спостереження за допомогою синтаксису `observableInstance.pipe(operator())`.
При виклику вони не змінюють існуючий екземпляр Об'єкту Спостреження.
Замість цього вони повертають новий Об'єкт Спостереження, логіка підписки якого базується на першому.

> Pipeable оператор — це функція, яка приймає Об'єкт Спостереження як вхідні дані та повертає інший Об'єкт Спостереження. Це чиста операція: вхідний Об'єкт Спостереження залишається незмінним.

Pipeable оператор є по суті чистою функцією, яка приймає один Об'єкт Спостереження як вхідні дані та генерує інший Об'єкт Спостереження як вихідні дані. Підписка на вихідний Об'єкт Спостереження також підписується на вхідний Об'єкт Спостереження.
?? Example

**Оператори створення** — це інший вид операторів, які можна викликати як окремі функції для створення нового Об'єкту Спостереження.

Наприклад: `of(1, 2, 3)` створює Об'єкт Спостереження, який буде віддавати 1, 2 і 3 один за одним. Оператори створення будуть розглянуті більш детально в наступному розділі.

Наприклад, оператор під назвою `map` є аналогом однойменного методу `Array.prototype.map`. Подібно до того, як `[1, 2, 3].map(x => x * x)` дасть `[1, 4, 9]`, Об'єкт Спостереження створюється так:

```javascript
import { of, map } from 'rxjs';

of(1, 2, 3)
	.pipe(map((x) => x * x))
	.subscribe((v) => console.log(`value: ${v}`));

// Logs:
// value: 1
// value: 4
// value: 9
```

видасть `1, 4, 9`. Іншим корисним оператором є `first`??:

```javascript
import { of, first } from 'rxjs';

of(1, 2, 3)
	.pipe(first())
	.subscribe((v) => console.log(`value: ${v}`));

// Logs:
// value: 1
```

Зауважте, що `map` потрібно створювати на льоту, оскільки їй потрібно надати функцію відображення.??
На відміну від цього, `first` може бути константою, але все одно створюється на льоту.
Як правило, усі оператори створюються незалежно від того, чи потрібні їм аргументи чи ні.

## Piping

Pipable operators є функціями, тому їх можна використовувати як звичайні функції: `op()(obs)` — але на практиці, як правило,
багато з них згортаються разом і швидко стають нечитабельними: `op4()(op3()(op2( )(op1()(obs))))`.??
З цієї причини у Об'єктів Спостереження є метод під назвою `.pipe()`, який виконує те саме, але його набагато легше читати:

```javascript
obs.pipe(op1(), op2(), op3(), op4());
```

З точки зору стилістики, `op()(obs)` ніколи не використовується, навіть якщо є лише один оператор; `obs.pipe(op())` є загальноприйнятим.

## Кулькові діаграми

Щоб пояснити, як працюють оператори, текстових описів часто недостатньо.
Багато операторів пов’язані з часом, вони можуть, наприклад, затримувати, відбирати, дроселювати або зменшувати видання значень різними способами.

Діаграми часто є кращим інструментом для цього.
Кулькові діаграми — це візуальне представлення того, як працюють оператори, і включають вхідні Об'єкт(и) Спостереження, оператор і його параметри, а також вихідний Об'єкт Спостереження.

> На кулькових діаграмах час тече зліва направо, і діаграма описує, як значення («кульки») випускаються під час виконання Об'єкту Спостереження.

Об'єкти Спостереження є асинхронними операціями, тому нам потрібен спосіб представити плин часу. Це можна зробити за допомогою стрілки, що рухається зліва направо.

![image](https://miro.medium.com/max/600/1*aUjUVnDKc-JdNzD0y8IMWg.png)

Вертикальна лінія на кінці стрілки означає успішне завершення Об'єкта Спостереження.

![image](https://miro.medium.com/max/600/1*XopupLvHC-i6ntailoLEQw.png)

Якщо в Об'єкта Спостереження виникає помилка, вона позначається символом X. Після появи помилки Об'єкт Спостереження більше не видає жодних значень.

![image](https://miro.medium.com/max/600/1*ZvJ9aD8k3ywo4BTs28HLEA.png)

І, нарешті, ці кольорові кульки представляють значення та можуть з’являтися будь-де на часовій шкалі стрілки. Ці значення можуть бути рядками, числами, логічними значеннями або будь-яким іншим базовим типом.

![image](https://miro.medium.com/max/600/1*ZtA0Up-0zKXxcLwh7zc-cg.png)

Нижче ви можете побачити анатомію кулькової діаграми.
![image](https://user-images.githubusercontent.com/64136789/185767990-556eb075-31f2-4a78-ab26-67a16dd555a4.png)

### New

Пам’ятайте, кулькові діаграми допомагають нам зрозуміти оператори. І оператори бувають двох форм:

1. Оператори створення (`of`, `from`, `timer` тощо)
2. Конвеєрні оператори (`map`, `take`, `filter` тощо)

Оператори створення є автономними (вони створюють власні значення), що означає, що їхні кулькові діаграми є лише однією стрілкою:

![image](https://miro.medium.com/max/700/1*a6BMBG_UFmZ-9iTVJDdX8A.png)

```javascript
import { interval, take } from 'rxjs';

const numbers = interval(1000);

numbers.subscribe((x) => console.log('Next: ', x));

// Logs:
// Next: 0
// Next: 1
// Next: 2
// Next: 3
// Next...
```

І Pipable операторам потрібен «Input Observable» як джерело, оскільки вони самі не видають значень. Вони просто «оперують» цими значеннями. Таким чином, ви побачите кулькові діаграми для Pipable операторів з 1 або більше «Вхідними Об'єктами Спостереження», самим оператором і «Вихідним Об'єктом Спостереження».

Просто подумайте про це як про звичайні функції (технічно «чисті функції»), за винятком того, що їхні аргументи є Об'єктами Спостереження і значення, які вони повертають також є Об'єктами Спостереження.

Ось приклад:

![image](https://miro.medium.com/max/600/1*n7qbIBNC8jqpokRQ0BY-jg.png)

Важливо зауважити, що в деяких випадках порядок Об'єктів Спостереження має значення. Хоча деякі оператори повертають той самий вихідний Об'єкт Спостереження незалежно від порядку двох вхідних Об'єктів Спостереження, деякі оператори фактично використовують порядок цих вхідних даних для формування виводу.

Наведений вище оператор `concat()` є чудовим прикладом цього. Зверніть увагу, як вихідний Об'єкт Спостереження повертає три значення, випущені з вхідного Об'єкта Спостереження #1, перш ніж повернути два значення, випущені з вхідного Об'єкта Спостереження #2, навіть якщо обидва значення Об'єкта Спостереження #2 були випущені раніше.

У RxJS ми зазвичай називаємо вхідний Об'єкт Спостереження №1 «зовнішнім», а вхідний Об'єкт Спостереження №2 — «внутрішнім».

Як я вже сказав, порядок не завжди має значення. Візьмемо, наприклад, оператор `merge()`:

![image](https://miro.medium.com/max/700/1*kE77SCCW6HEdpMwvixthdQ.png)

Незалежно від того, в якому порядку викликаються два вхідних Об'єкти Спостереження, вихідний Об'єкт Спостереження завжди видаватиме однакові значення.

Щоб зрозуміти цю публікацію надалі, вам потрібно розібратися з певною термінологією:

**Зовнішній Об'єкт Спостереження**: або те, що я назвав «вхідний Об'єкт Спостереження №1», є Об'єктом Спостереження, який знаходиться у верхній частині кожної діаграми. Його називають «зовнішнім», тому що він зазвичай виглядає так під час написання коду:

```javascript
outerObservable().pipe(mergeMapTo(innerObservable(), (x, y) => x + y));
```

**Внутрішній Об'єкт Спостереження**: або те, що я назвав «вхідний Об'єкт Спостереження #2», є Об'єктом Спостереження нижче зовнішнього Об'єкта Спостереження, але перед оператором на кожній діаграмі. Його називають «внутрішнім» з тієї ж причини, що й вище.

**Вихідний Об'єкт Спостереження**: під час використання операторів RxJS іноді існує багато шарів між вхідними Об'єктом(ами) Спостереження і вихіднимм Об'єктами Спостереження, але ви можете думати про вихідний Об'єкт Спостереження як про «повернене значення».

**Вхідний Об'єкт Спостереження**: Це загальний термін для визначення будь-якого Об'єкта Спостереження, який НЕ є «вихідним Об'єктом Спостереження». Іншими словами, як внутрішні, так і зовнішні Об'єкти Спостереження вважаються «вхідними» Об'єктами Спостереження.

І нарешті, не всі оператори дотримуються концепції «внутрішніх» і «зовнішніх» Об'єктів Спостереження. Для деяких операторів, таких як `combineLatest` (ми побачимо це пізніше), усі Об'єкти Спостереження обробляються однаково, і тому ми називаємо кожен Об'єкт Спостереження «вхідним Об'єктом Спостереження».

## Оператори Створення

Оператори створення — це функції, які можна використовувати для створення Об'єктів Спостереження із певною загальною попередньо визначеною поведінкою або шляхом приєднання до інших Об'єктів Спостереження.

Прикладом оператора створення може бути інтервальна функція. Він приймає число (не Об'єкт Спостереження) як вхідний аргумент і створює Об'єкт Спостереження на виході:

```javascript
import { interval } from 'rxjs';

const observable = interval(1000 /* number of milliseconds */).subscribe(console.log);

setTimeout(() => observable.unsubscribe(), 5000);

// Output will be:
// 0
// 1
// 2
// 3
// 4
```

Тут немає ніякої магії. Всі ці оператори використовують `new Observable()` всередині. Перевага їх в тому, що нам не потрібно власноруч писати одне і те ж знову і знову.

Наприклад, ось як працює оператор `interval()` всередині:

```javascript
import { Observable } from 'rxjs';

function interval(intervalTime) {
	let number = 0;
	return new Observable((subscriber) => {
		const intervalId = setInterval(() => subscriber.next(number++), intervalTime);

		return () => {
			clearInterval(intervalId);
		};
	});
}

const observable = interval(1000).subscribe(console.log);

setTimeout(() => observable.unsubscribe(), 5000);

// Output will be:
// 0
// 1
// 2
// 3
// 4
```

Це ще не повний приклад того, що є всередині `interval()`, але загальну суть ви розумієте.

> Замість того, щоб писати свої Об'єкти Спостереження з нуля, в нас є набір операторів створення, які роблять це за нас

Ось список всіх операторів створення:

-   ajax
-   create
-   defer
-   empty
-   from
-   fromEvent
-   generate
-   interval
-   of
-   range
-   throw
-   timer

Ми вже подивилися на роботу `interval()`, давайте ще розглянемо найбільш вживані оператори в цій статті.

`from` - створює Об'єкт Спостерження з масиву, Promise, iterable.

```javascript
import { from } from 'rxjs';

const array = [10, 20, 30];
const result = from(array);

result.subscribe((x) => console.log(x));

// Logs:
// 10
// 20
// 30
```

![image](https://rxjs.dev/assets/images/marble-diagrams/from.png)

`of` - створює Об'єкт Спостережння, який віддає через `next` передані дані по черзі, а потім завершається.

```javascript
import { of } from 'rxjs';

of(10, 20, 30).subscribe({
	next: (value) => console.log('next:', value),
	error: (err) => console.log('error:', err),
	complete: () => console.log('the end'),
});

// Outputs
// next: 10
// next: 20
// next: 30
// the end
```

`fromEvent` - створює Об'єкт Спостереження з події(кліки/ввід даних, тощо).

```javascript
import { fromEvent } from 'rxjs';

const clicks = fromEvent(document, 'click');
clicks.subscribe((x) => console.log(x));

// Results in:
// MouseEvent object logged to console every time a click
// occurs on the document.
```

### Join Creation Operators

Це оператори створення Об'єктів Спостереження, які також мають функцію об’єднання — видають значення кількох вихідних Об'єктів Спостереження.

-   combineLatest
-   concat
-   forkJoin
-   merge
-   partition
-   race
-   zip

Перш ніж почати, я пояснюю, що всі приклади використовують ці три Об'єкти Спостереження як вхідні дані.

```javascript
import { from, Observable } from 'rxjs';

async function* hello() {
	const wait = async (time: number) => new Promise((res) => setTimeout(res, time));
	yield 'Hello';
	await wait(1000);
	yield 'from';
	await wait(500);
	yield 'iterator';
}

export const iterator$ = from(hello());
export const arrayFrom$ = from(['Hello', 'from', 'array']);
export const arrayOfWithDelay$ =
	new Observable() <
	number >
	((subscriber) => {
		let counter = 10;
		const id = setInterval(() => {
			if (counter > 0) {
				subscriber.next(counter--);
			} else {
				clearInterval(id);
				subscriber.complete();
			}
		}, 500);
	});
```

`combineLatest` - Об’єднує кілька Об'єктів Спостереження для створення нового, значення якого обчислюються з останніх значень кожного з його вхідних Об'єктів Спостереження.

```javascript
import { combineLatest } from 'rxjs';
import { arrayFrom$, arrayOfWithDelay$, iterator$ } from '../sources';

console.log(new Date().toLocaleTimeString(), `[combineLatest] start`);

combineLatest([iterator$, arrayFrom$, arrayOfWithDelay$]).subscribe({
	next: (res) => console.log(new Date().toLocaleTimeString(), `[combineLatest]`, res),
	complete: () => console.log(new Date().toLocaleTimeString(), `[combineLatest] complete`),
});
```

```
12:38:22 [combineLatest] start
12:38:22 [combineLatest] [ 'Hello', 'array', 10 ]
12:38:23 [combineLatest] [ 'from', 'array', 10 ]
12:38:23 [combineLatest] [ 'from', 'array', 9 ]
12:38:23 [combineLatest] [ 'iterator', 'array', 9 ]
12:38:23 [combineLatest] [ 'iterator', 'array', 8 ]
12:38:24 [combineLatest] [ 'iterator', 'array', 7 ]
12:38:24 [combineLatest] [ 'iterator', 'array', 6 ]
12:38:25 [combineLatest] [ 'iterator', 'array', 5 ]
12:38:25 [combineLatest] [ 'iterator', 'array', 4 ]
12:38:26 [combineLatest] [ 'iterator', 'array', 3 ]
12:38:26 [combineLatest] [ 'iterator', 'array', 2 ]
12:38:27 [combineLatest] [ 'iterator', 'array', 1 ]
12:38:27 [combineLatest] complete
```

![image](https://res.cloudinary.com/practicaldev/image/fetch/s--XIXZCEAy--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0ixjrhbs6o70piea384a.jpg)

У цьому прикладі ви можете побачити, як цей оператор віддає масив значень кожного разу, коли Об'єкт Спостереження видає одне значення.
Важливо пам’ятати, що оператор видає перше значення тільки коли всі залежні Об'єкти Спостереження видають перше значення.
Як ви бачите, результатом оператора `combineLatest` є масив, де елементи зберігають порядок.

`forkJoin` - Приймає масив Об'єктів Спостереження і повертає новий, який видає або масив значень у тому самому порядку, що й переданий масив.

```javascript
import { forkJoin } from 'rxjs';
import { arrayFrom$, arrayOfWithDelay$, iterator$ } from '../sources';

console.log(new Date().toLocaleTimeString(), `[forkJoin] start`);

forkJoin([iterator$, arrayFrom$, arrayOfWithDelay$]).subscribe({
	next: (res) => console.log(new Date().toLocaleTimeString(), `[forkJoin]`, res),
	complete: () => console.log(new Date().toLocaleTimeString(), `[forkJoin] complete`),
});
```

```
14:38:58 [forkJoin] start
14:39:04 [forkJoin] [ 'iterator', 'array', 1 ]
14:39:04 [forkJoin] complete
```

![image](https://res.cloudinary.com/practicaldev/image/fetch/s--oUyntFs8--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/633cbiicb2ag0n6u1lm9.jpg)

`forkJoin` схожий на оператор `combineLatest`, відмінність полягає в тому, що оператор `forkJoin` видає лише одне значення, коли всі Об'єкти Спостереження завершені. Простіше кажучи, оператор `forkJoin` видає лише останнє значення оператора `combineLatest`.

`concat` - Створює вихідний Об'єкт Спостереження, який послідовно видає всі значення з першого заданого Об'єкту Спостереження, а потім переходить до наступного.

```javascript
import { concat } from 'rxjs';
import { arrayFrom$, arrayOfWithDelay$, iterator$ } from '../sources';

console.log(new Date().toLocaleTimeString(), `[concat] start`);

concat(iterator$, arrayFrom$, arrayOfWithDelay$).subscribe({
	next: (res) => console.log(new Date().toLocaleTimeString(), `[concat]`, res),
	complete: () => console.log(new Date().toLocaleTimeString(), `[concat] complete`),
});
```

```
14:44:23 [concat] start
14:44:23 [concat] Hello
14:44:24 [concat] from
14:44:24 [concat] iterator
14:44:24 [concat] Hello
14:44:24 [concat] from
14:44:24 [concat] array
14:44:25 [concat] 10
14:44:25 [concat] 9
14:44:26 [concat] 8
14:44:26 [concat] 7
14:44:27 [concat] 6
14:44:27 [concat] 5
14:44:28 [concat] 4
14:44:28 [concat] 3
14:44:29 [concat] 2
14:44:29 [concat] 1
14:44:30 [concat] complete
```

![image](https://res.cloudinary.com/practicaldev/image/fetch/s--Q69cz2lJ--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/cmyi538qlfm44raobivz.jpg)

Як бачите, цей оператор віддає всі значення Об'єктів Спостереження у послідовності.
`concat`, на відміну від `combineLatest`, не запускає всі Об'єкти Спостереження паралельно,а робить це послідовно, починаючи з першого і не переходячи до наступного, доки поточний не буде завершено.

`merge` - Створює вихідний Об'єкт Спостереження, який одночасно видає всі значення з кожного даного вхідного Об'єкту Спостереження.

```javascript
import { merge } from 'rxjs';
import { arrayFrom$, arrayOfWithDelay$, iterator$ } from '../sources';

console.log(new Date().toLocaleTimeString(), `[merge] start`);

merge(iterator$, arrayFrom$, arrayOfWithDelay$).subscribe({
	next: (res) => console.log(new Date().toLocaleTimeString(), `[merge]`, res),
	complete: () => console.log(new Date().toLocaleTimeString(), `[merge] complete`),
});
```

```
14:58:48 [merge] start
14:58:48 [merge] Hello
14:58:48 [merge] from
14:58:48 [merge] array
14:58:48 [merge] Hello
14:58:48 [merge] 10
14:58:49 [merge] from
14:58:49 [merge] 9
14:58:49 [merge] iterator
14:58:49 [merge] 8
14:58:50 [merge] 7
14:58:50 [merge] 6
14:58:51 [merge] 5
14:58:51 [merge] 4
14:58:52 [merge] 3
14:58:52 [merge] 2
14:58:53 [merge] 1
14:58:53 [merge] complete
```

![image](https://res.cloudinary.com/practicaldev/image/fetch/s--eVeMNJl7--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/1m8e0zqtn8ymn7nwgrq3.jpg)

Оператор `merge` подібний до оператора `concat`, на відміну від того, що оператор `merge` запускає всі Об'єкти Спостереження в режимі одночасного виконання, тому в цьому випадку всі Об'єкти Спостереження починаються разом, і кожного разу, коли Об'єкт Спостереження видає значення, оператор `merge` видає це останнє значення.

`race` - Повертає перший Об'єкт Спостереження, який видасть елемент.

```javascript
import { race } from 'rxjs';
import { arrayFrom$, arrayOfWithDelay$, iterator$ } from '../sources';

console.log(new Date().toLocaleTimeString(), `[race] start`);

race([iterator$, arrayFrom$, arrayOfWithDelay$]).subscribe({
	next: (res) => console.log(new Date().toLocaleTimeString(), `[race]`, res),
	complete: () => console.log(new Date().toLocaleTimeString(), `[race] complete`),
});
```

```
15:09:03 [race] start
15:09:03 [race] Hello
15:09:03 [race] from
15:09:03 [race] array
15:09:03 [race] complete
```

![image](https://res.cloudinary.com/practicaldev/image/fetch/s--CRsl-Y19--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/c6xo26df399om1tfj3lc.jpg)

Цей оператор особливий, він видає перший Об'єкт Спостереження, який віддасть перше значення. Іншими словами, він бере швидший Об'єкт Спостереження і ігнорує інші.

`zip` - Об’єднує кілька Об'єктів Спостереження для створення нового, значення якого обчислюються зі значень у порядку кожного вхідного Об'єкту Спостереження.

```javascript
import { zip } from 'rxjs';
import { arrayFrom$, arrayOfWithDelay$, iterator$ } from '../sources';

console.log(new Date().toLocaleTimeString(), `[zip] start`);

zip([iterator$, arrayFrom$, arrayOfWithDelay$]).subscribe({
	next: (res) => console.log(new Date().toLocaleTimeString(), `[zip]`, res),
	complete: () => console.log(new Date().toLocaleTimeString(), `[zip] complete`),
});
```

```
15:09:27 [zip] start
15:09:27 [zip] [ 'Hello', 'Hello', 10 ]
15:09:28 [zip] [ 'from', 'from', 9 ]
15:09:28 [zip] [ 'iterator', 'array', 8 ]
15:09:28 [zip] complete
```

![image](https://res.cloudinary.com/practicaldev/image/fetch/s--S9YG5WMo--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/4wk2khoruej9urxlm1de.jpg)

Цей оператор може здатися дивним, але він може використовуватися для об'єднання впорядкованих значень різних Об'єктів Спостереження.
У цьому прикладі ми маємо 3 спостережувані:

iterator$: ['Hello', 'from', 'iterator', '!']
arrayFrom$: ['Hello', 'from', 'array', '!']
arrayOfWithDelay$: [10, 9, 8, 7, 6, 5, 4, 3, 2, 1]
За допомогою оператора `zip` ми об’єднуємо значення в порядку їх індексу:

['Hello', 'Hello', 10]
['from', 'from', 9]
['iterator', 'array', 8]

Як ви бачите, оператор припиняє видавати значення на індексі першого завершеного Об'єкта Спостереження.

## Pipable оператори

Ми не будемо перелічувати тут всі pipable оператори, тому що їх досить багато.
Нижче наведено деякі з найпоширеніших операторів і способи тлумачення їхніх кулькових діаграм.

Ми почнемо з знайомого нам оператора `map()`.

![image](https://miro.medium.com/max/700/1*RUZNpZqArzCfc0UTjSZJ6A.png)

Верхня стрілка представляє наш вхідний Об'єкт Спостереження і видає три значення. Це досить просто, якщо ви працювали з функцією `map` використовуючи масиви JavaScript. Все, що ви робите, це трансформуєте значення, що надходять із вхідного Об'єкта Спостереження, у 10 разів. Ось кулькова діаграма, відтворена в коді:

```javascript
import { of } from 'rxjs';
import { map } from 'rxjs/operators';

const inputObservable = of(1, 2, 3);

inputObservable()
	.pipe(map((x) => 10 * x))
	.subscribe((value) => console.log(value));
```

![image](https://miro.medium.com/max/700/1*4Jg3vWN82T4R2N1QW4WJPg.gif)

Нижче наведено оператор `take()`.

![image](https://miro.medium.com/max/700/1*cP6hRnkMKXtHsqvEu0qo5A.png)

На наведеній вище діаграмі вхідний Об'єкт Спостереження віддає чотири цілі числа — `1`, `2`, `3` і `4`. Якби ви підписалися на цей вхідний Об'єкт Спостереження напряму, ви б отримали ці чотири значення.

Але якщо ви передаєте оператор `take(2)` по каналу, новий вихідний Об'єкт Спостереження захопить перші два виданих значення, а потім завершиться. Вхідний Об'єкт Спостереження все ще видаватиме останні два значення, але наш вихідний Об'єкт Спостереження їх не бачитиме, оскільки він завершився після двох значень.

Нижче наведено код і візуалізацію:

```javascript
import { interval } from 'rxjs';
import { map, take } from 'rxjs/operators';

const inputObservable = interval(1000);

inputObservable().pipe(take(2));
```

![image](https://miro.medium.com/max/700/1*k1suhVSk3QsosfOiyqBB3A.gif)

## Cтворення своїх операторів

Це може вас здивувати, але оператор — це всього лише функція, яка повертає Об'єкт Спостереження.
Візьмемо такий приклад:

```js
import { interval } from 'rxjs';
import { map } from 'rxjs/operators';

interval(1000).pipe(
  map(num => num + 1)
).subscribe(...)
```

У нас є функція `interval`, яка повертає Об'єкт Спостереження. Використовуючи його як джерело, ми використовуємо метод `pipe()`, передаючи йому функцію `map`, яка повертає оператор. Давайте подивимося реалізацію методу `pipe()`, щоб ми могли краще зрозуміти, як він працює:

```ts
class Observable {
	pipe(...operators): Observable<any> {
		return operators.reduce((source, next) => next(source), this);
	}
}
```

Метод `pipe` приймає масив операторів, проходить по ньому циклом і кожного разу викликає наступний оператор, передаючи йому результат попереднього як джерело. Якщо ми використаємо це в нашому прикладі, ми отримаємо такий вираз:

```
map(interval): Observable
```

Вивчаючи цей код, ми можемо зрозуміти, що побудувати оператор так само просто, як написати функцію, яка приймає `source Observable` як вхідні дані та повертає новий Об'єкт Спостереження:

```ts
function myOperator<T>(source: Observable<T>) {
	return source;
}
```

Ми щойно створили свій оператор! Так, в ньому немає сенсу, оскільки він повертає той самий Об'єкт Спостереження, що й отримує, але, тим не менш, це оператор:

```ts
interval(1000)
	.pipe(myOperator)
	.subscribe((value) => console.log(value));
```

А тепер давайте зупинимося на секунду та поговоримо про поширену помилку щодо цієї теми. Дивлячись на наш приклад, ви можете почути, як люди описують його як «підписка на Об'єкт Спостереження `interval`». Це помилковий опис, ви _завжди підписані на останнього оператора в ланцюжку_ (тобто останнього в списку операторів).

Отже, у цьому прикладі ми підписані на Об'єкт Спостереження, який повертає функція `myOperator`. Дозвольте мені продемонструвати це. Давайте повернемо інший Об'єкт Спостереження з нашого оператора:

```ts
function myOperator<T>(source: Observable<T>) {
	return new Observable((subscriber) => {
		subscriber.next(`🦄`);
		subscriber.complete();
	});
}
```

Якщо ми повторимо наш приклад, усе, що ми коли-небудь отримаємо, це один 🦄. Це тому, що ми отримуємо `source Observable` (у нашому випадку згенерований інтервалом), але нічого з ним не робимо (наприклад, підписуємося на нього).

Це доводить, що ми підписані на Об'єкт Спостереження, який повертається з `myOperator`, а не на той, який повертається з `interval`.

Це підводить мене до наступної теми — ланцюга Об'єкта Спостереження. Візьмемо наш перший приклад, `interval` із `map`:

```ts
import { interval } from 'rxjs';
import { map } from 'rxjs/operators';

interval(1000).pipe(
  map(num => num + 1)
).subscribe(...)
```

Якщо, як ми зазначали, ми фактично підписані на Об'єкт Спостереження, який повертається з оператора `map()`, як ми досягаємо Об'єкта Спостереження, який повертається з `interval()`?

Відповідь проста — ніяк! Кожен оператор отримує аргументом вхідний Об'єкт Спостереження, який йде в ланцюгу перед ним, і цей оператор (у більшості випадків) підписується на вхідний Об'єкт Спостереження.

Коли ми викликаємо `subscribe()`, ми виконуємо `map()`,який повертає Об'єкт Спостереження, який сам підписується на Об'єкт Спостереження від `interval`. Кожного разу, коли Об'єкт Спостереження інтервалу віддає нове значення, значення досягає функції підписки `map`.

Потім, після застосування функції проекції `map` до значення, Об'єкт Спостереження повернутий `map` видає результат цієї функції до останньої підписки в ланцюжку.

Тепер, коли ми розуміємо, як працює цей механізм, давайте створимо кілька операторів.

### Створення оператору filterNill

У нас може бути джерело, яке може видавати `null` або `undefined`, і ми хочемо ігнорувати ці значення. Давайте створимо оператор, який керуватиме цим завданням для нас, і ми зможемо застосувати його до будь-якого джерела:

```ts
function filterNil() {
	return function <T>(source: Observable<T>): Observable<T> {
		return new Observable((subscriber) => {
			source.subscribe({
				next(value) {
					if (value !== undefined && value !== null) {
						subscriber.next(value);
					}
				},
				error(error) {
					subscriber.error(error);
				},
				complete() {
					subscriber.complete();
				},
			});
		});
	};
}
```

У цьому випадку, на відміну від попереднього, ми створюємо функцію, яка повертає оператор. Причина в тому, що це робить наш код масштабованим, оскільки його можна легко розширити для отримання аргументів у майбутньому.

Отримавши джерело, ми підписуємося на нього. Коли джерело видає значення, ми перевіряємо, чи є значення `undefined` або `null` — якщо це так, ми його ігноруємо; В іншому випадку ми передаємо значення наступному підписнику.

Те саме стосується сповіщень про помилки та завершення. Пропускаємо їх через ланцюжок; інакше решта ланцюга не оброблятиме їх належним чином. Якщо ми запустимо наступний приклад, ми побачимо, що ми не отримуємо сповіщення про перше значення `0`:

```ts
interval(1000)
	.pipe(
		map((value) => (value === 0 ? undefined : value)),
		filterNil()
	)
	.subscribe((value) => console.log(value));
```

Ви могли помітити важливу проблему — ми створили витік пам’яті. Пам’ятайте, що кожен спостережуваний повертає функцію скасування підписки, яка відповідає за виконання будь-якого необхідного очищення. Ця функція викликається кожного разу, коли хтось викликає `unsubscribe()`. У нашому прикладі ми підписалися на джерело, але ніде не скасовували підписку на нього.

Це означає, що джерело інтервалу продовжуватиме працювати навіть після того, як ми викличемо метод `unsubscribe` для отриманого Об'єкту Спостереження.

Отже, як ми можемо це виправити? Рішення досить просте, необхідно викликати метод `unsubscribe` у джерела:

```ts
function filterNil() {
	return function <T>(source: Observable<T>): Observable<T> {
		return new Observable((subscriber) => {
			const subscription = source.subscribe({
				next(value) {
					if (value !== undefined && value !== null) {
						subscriber.next(value);
					}
				},
				error(error) {
					subscriber.error(error);
				},
				complete() {
					subscriber.complete();
				},
			});

			return () => subscription.unsubscribe(); // <-- here
		});
	};
}
```

У нашому випадку є більш короткий шлях — ми можемо безпосередньо повернути результат вихідного виклику підписки:

```ts
function filterNil() {
  return function<T>(source: Observable<T>): Observable<T> {
    return new Observable(subscriber => {
      return source.subscribe({
        next(value) {
          if(value !== undefined && value !== null) {
            subscriber.next(value);
          }
        },
        ...
      });
    });
  }
}
```

Тепер, коли ми викликаємо `unsubscribe()` для результату, він фактично викликає `unsubscribe()` для джерела.

### Створення нового оператора з вже існуючого

Тепер, коли ми дізналися про основи створення операторів і про те, наскільки це легко, ми можемо скористатися коротким і в більшості випадків рекомендованим способом їх створення — створити їх із існуючих.

Давайте повторно створимо оператор `filterNil`, використовуючи, як ви могли здогадатися, оператор `filter`:

```ts
function filterNil() {
	return function <T>(source: Observable<T>) {
		return source.pipe(filter((value) => value !== undefined && value !== null));
	};
}
```

Непогано, але ми можемо зробити ще один крок далі. Оскільки pipable оператори повертають функції, які, як ми дізналися, отримують Об'єкт Спостереження як джерело, ми можемо зробити `filterNil` ще коротшим:

```ts
function filterNil() {
	return filter((value) => value !== undefined && value !== null);
}
```

## Об'єкти Спостереження вищого порядку

```javascript
const gameData = [
	{
		title: 'Mega Man 2',
		bosses: [
			{
				name: 'Bubble Man',
				weapon: 'Bubble Beam',
			},
			{
				name: 'Metal Man',
				weapon: 'Metal Blade',
			},
		],
	},
	{
		title: 'Mega Man 3',
		bosses: [
			{
				name: 'Gemini Man',
				weapon: 'Gemini Laser',
			},
			{
				name: 'Top Man',
				weapon: 'Top Spin',
			},
		],
	},
];

// return an array of all bosses

const bossesArray = gameData.map((game) => {
	return game.bosses;
});

// uh oh, those are nested arrays!
// [[{},{}],[{},{}]]
```

Ми відображаємо масив gameData. `map` завжди повертає масив. Завжди.
Однак усередині виклику `map` ми повертаємо `game.bosses`, який також є масивом, тому ми отримуємо вкладений масив. `[[{}, {}], [{}, {}]]`.

Напевно, це не те, чого ми хотіли. В ідеалі ми хотіли б мати все зведене в єдиний масив. Нам просто потрібен метод, який візьме масив і зменшить його глибину до одиниці. Оскільки у нас є один рівень вкладеності в цьому масиві, ми хотіли б застосувати цей метод зменшення глибини один раз.

На жаль,`Array.prototype` наразі не має хорошого способу зробити це. `Array.prototype.concat` можна використовувати, хоча він не ідеальний для цієї ситуації. У майбутньому ми матимемо доступ до `flat` і `flatMap`, але зараз це не принесе нам особливої користі, тому давайте реалізуємо нашу власну функцію `flatten`. Він поверне новий масив зі зменшеною глибиною в одиницю.

```javascript
Array.prototype.flatten = function () {
	let retVal = [];

	this.forEach((a) => {
		retVal = retVal.concat(a);
	});

	return retVal;
};

let arr = [[1, 2], [3], 4, [5, 6], [[7], 8]];

console.log(arr.flatten()); // [1, 2, 3, 4, 5, 6, [7], 8]
```

Тепер ми можемо повернутися до нашого прикладу вище та застосувати зведення до вкладеного масиву `game.bosses`, отриманого з `map`:

```javascript
const bossesArray = gameData
	.map((game) => {
		return game.bosses;
	})
	.flatten();

// returns a flattened array of boss objects [{}, {}, {}, {}]
```

`map`, за якою слідує `flatten`, є настільки поширеним поєднанням операцій, що більшість мов поєднують обидві в оператор `flatMap`.

```javascript
Array.prototype.flatMap = function (fn) {
	return this.map(fn).flatten();
};

// usage
const bossesArray = gameData.flatMap((game) => {
	return game.bosses;
}); // [{}, {}, {}, {}]
```

Здатність занурюватися в шар за допомогою `map`, а потім повертатися до шару за допомогою `flatteb` — це дуже корисна навичка під час роботи з вкладеними структурами даних, такими як масиви. Це також виникає на регулярній основі під час роботи з Об'єктами Спостереження вищого порядку (Об'єкт Спостереження всередині іншого Об'єкту Спостереження).

Часто у нас виникає необхідність поєднувати більше однієї асинхронної дії разом. Прикладом може бути пошук, який слідкує за тим, що користувач вводить в поле пошуку, а потім робить http запит на сервер, щоб повернути знайдені елементи користувачу.

Спочатку ми подивимося на інтуітивний, але **неправильний** спосіб це зробити.

```javascript
import { of } from 'rxjs';
import { ajax } from 'rxjs/ajax';

// Для простоти ми не будемо створювати поле вводу і слухати його, а замінимо просто 3 умовними значеннями пошуку
const userInputs$ = of('users', 'addresses');

userInputs$.subscribe((searchText) => {
	ajax(`https://random-data-api.com/api/v2${searchText}`).subscribe((result) => {
		console.log(result);
	});
});
```

Ми імітуємо дані від користувача використовуючи `of()`, підписуємося на цей потік і всередині підписки підписуємося на потік, який робить http-запит на сервер і віддає нам результати.
Даний код беззаперечно вважається **поганою практикою**.

1. Це складно читається. Можливо ви чули вираз `callback hell`, так ось, це явище не унікальне для зворотніх функцій і ви можете зробити те саме з Об'єктами Спостереження.
2. Це складно підтримувати. Нам потрібно відписуватися від підписок, які ми створюємо. Чим більше ми підписуємося самостійно, тим важче нам це підтримувати.

> Якщо у вас виникає необхідність створити підписку всередині іншої підписки, то скоріше за все, ви ще не знайшли оператора, який зробить це за вас.

Що ж, давайте спробуємо використовувати оператори:

```javascript
import { of } from 'rxjs';
import { ajax } from 'rxjs/ajax';
import { map } from 'rxjs/operators';

const userInputs$ = of('users', 'addresses');

userInputs$.pipe(map((searchText) => ajax(`https://random-data-api.com/api/v2/${searchText}`))).subscribe((result) => {
	console.log(result);
});
// We will get:
// Observable {...}
// Observable {...}
// Observable {...}
```

В даному випадку ми отримали текст для пошуку і через `map` повернули Об'єкт Спостереження з нашим запитом на сервер. Нічого дивного в тому, що на виході ми замість даних отримали Об'єкт Спостереження. Для того, щоб він почав працювати, нам потрібно підписатися на нього.

Підписуватися на нього самостійно всередині `pipe()` ідея не краща, виглядатиме це ще складніше.

```javascript
import { mergeAll, Observable, of } from 'rxjs';
import { ajax } from 'rxjs/ajax';
import { map } from 'rxjs/operators';

const userInputs$ = of('users', 'addresses');

userInputs$
	.pipe(
		map((searchText) => {
			return new Observable((subscriber) => {
				ajax(`https://random-data-api.com/api/v2/${searchText}`).subscribe({
					next: (result) => subscriber.next(result),
				});
			});
		}),
		mergeAll()
	)
	.subscribe((result) => {
		console.log(result);
	});
```

В даному випадку ми не можемо просто підписатися вдруге на `ajax()`, як ми це зробили в першому неправильному прикладі. Все тому, що нам потрібно повернути значення, яке віддасть сервер далі до `subscribe`. Це можна зробити створивши новий Об'єкт Спостереження, який просто віддасть через `subscriber.next(result)` отримане значення, коли воно з'явиться. І це ще не все. Так як ми знову повертаємо Об'єкт Спостереження, а не сам результат, нам потрібно використати один з `flattening` операторів, для того, щоб отримати саме значення. І ще ніхто не відміняв відписування від підписок всередині `pipe()`.

Враховуючи все вищесказане, нам потрібні рішення, які дозволять нам зручно працювати з Об'єктами Спостереження всередині Об'єктів Спостереження.

Познайомтеся з операторами розрівнювання:

1. `mergeMap`
2. `concatMap`
3. `exhaustMap`
4. `switchMap`

---

Об'єкти Спостереження найчастіше віддають прості значення, такі як рядки та числа, але напрочуд часто необхідно обробляти так звані Об'єкти Спостереження вищого порядку.
Наприклад, уявіть, що у вас є Об'єкт Спостереження, що видає рядки, які є URL-адресами файлів, які ви хочете переглянути. Код може виглядати так:

```javascript
const fileObservable = urlObservable.pipe(map((url) => http.get(url)));
```

`http.get()` повертає Об'єкт Спостереження (імовірно рядок або масиви рядків) для кожної окремої URL-адреси. Тепер у вас є Об'єкт Спостереження всередині Об'єкту Спостереження, або **Об'єкт Спостереження вищого порядку**.

Але як працювати з Об'єктом Спостереження вищого порядку? Як правило, шляхом розрівнювання: (якимось чином) перетворюючи Об'єкт Спостереження вищого порядку на звичайний Об'єкт Спостереження. Наприклад:

```javascript
const fileObservable = urlObservable.pipe(
	map((url) => http.get(url)),
	concatAll()
);
```

Оператор `concatAll()` підписується на кожен «внутрішній» Об'єкт Спостереження, який виходить із «зовнішнього» Об'єкту Спостереження,
і копіює всі випущені значення, доки цей Об'єкт Спостереження не завершиться, і не переходить до наступного.
Усі значення об’єднані таким чином. Інші корисні оператори зведення (які називаються операторами з’єднання).

-   `mergeAll()` — підписується на кожен внутрішній Об'єкт Спостереження, щойно він надходить, а потім видає кожне значення, коли воно надходить
-   `switchAll()` — підписується на перший внутрішній Об'єкт Спостереження, коли він надходить, і видає кожне значення, коли воно надходить, але коли наступний внутрішній Об'єкт Спостереження надходить, скасовує підписку на попередній і підписується на новий.
-   `exhaustAll()` — підписується на перший внутрішній Об'єкт Спостереження, коли він надходить, і видає кожне значення, коли воно надходить, відкидаючи всі нові внутрішні Об'єкти Спостереження, доки перший не завершиться, а потім чекає на наступний внутрішній Об'єкт Спостереження.

Подібно до того, як багато бібліотек поєднують `map()` і `flat()` (або `flatten()`) в один `flatMap()`, існують еквіваленти відображення всіх операторів розрівнювання RxJS `concatMap()`, `mergeMap()`, `switchMap()` і `exhaustMap()`. -->
