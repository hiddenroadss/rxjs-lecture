# Об'єкти Спостереження

Об'єкти Спостереження — це ліниві Push-колекції кількох значень. Вони заповнюють пропущене місце в наступній таблиці:
| | Single | Multiple |
| --- | --- | --- |
| Pull | Функція | Ітератор |
| Push | Обіцянка | **Об'єкт Спостереження** |

Приклад. Нижче наведено Об'єкт Спостереження, який надсилає значення 1, 2, 3 негайно (синхронно) під час підписки, а значення 4 після того, як мине одна секунда після виклику підписки, а потім завершується:
```javascript
import { Observable } from 'rxjs';

const observable = new Observable(subscriber => {
  subscriber.next(1);
  subscriber.next(2);
  subscriber.next(3);
  setTimeout(() => {
    subscriber.next(4);
    subscriber.complete();
  }, 1000);
```
Саме по собі, створення Об'єкту Спостереження нічого не робить, для того, щоб почати отримувати дані, нам потрібно на нього підписатися:
```javascript
import { Observable } from 'rxjs';

const observable = new Observable(subscriber => {
  subscriber.next(1);
  subscriber.next(2);
  subscriber.next(3);
  setTimeout(() => {
    subscriber.next(4);
    subscriber.complete();
  }, 1000);
});

console.log('just before subscribe');
observable.subscribe({
  next: (x) => console.log('got value ' + x),
  error: (err) => console.error('something wrong occurred: ' + err),
  complete: () => console.log('done')
});
console.log('just after subscribe');
```
В консолі ми побачимо:
```
just before subscribe
got value 1
got value 2
got value 3
just after subscribe
got value 4
done
```
## Pull vs Push
Pull і Push — це два різні протоколи, які описують, як виробник даних може спілкуватися зі споживачем даних.

Що таке Pull? У системах Pull споживач визначає, коли він отримує дані від виробника. Сам Виробник не знає, коли дані будуть доставлені Споживачу.
Кожна функція JavaScript є системою Pull. Функція є виробником даних, і код, який викликає функцію, споживає їх, «витягуючи» єдине значення, що повертається з її виклику.

ES2015 представив функції-генератори та ітератори (function*), інший тип Pull-системи. Код, який викликає iterator.next(), є споживачем, який «витягує» кілька значень з ітератора (виробника).

| | Виробник | Споживач |
| --- | --- | --- |
| Pull | Пасивний: створює дані за запитом | **Активний:** вирішує, коли запитувати дані |
| Push | **Активний:** створює дані у власному темпі  | Пасивний: реагує на отримані дані |

Що таке Push? У системах Push виробник визначає, коли надсилати дані споживачеві. Споживач не знає, коли він отримає ці дані.
Обіцянки є найпоширенішим типом Push-системи в JavaScript сьогодні. Обіцянка (виробник) надає дозволене значення зареєстрованим зворотним викликам (споживачам), але, на відміну від функцій, саме проміс відповідає за те, щоб точно визначити, коли це значення «виштовхується» до зворотних викликів.

RxJS представляє Об'єкти Спостереження(Observables), нову систему Push для JavaScript. Об'єкт Спостереження є виробником колекції значень, «виштовхуючи» їх до спостерігачів (споживачів).
- Функція — це обчислення з лінивим виконанням, яке синхронно повертає одне значення під час виклику. 
- Генератор — це обчислення з лінивим виконанням, яке синхронно повертає від нуля до (потенційно) нескінченних значень ітеративно.
- Обіцянка — це обчислення, яке може (або не може) зрештою повернути одне значення. 
- Об'єкт Спостереження — це обчислення з лінивим виконанням, яке може синхронно або асинхронно повертати від нуля до (потенційно) нескінченної кількості значень з моменту його виклику.

## Об'єкти Спостереження як узагальнення функцій
Всупереч популярним твердженням, Об'єкти Спостереження не схожі на EventEmitters і не схожі на Promises для кількох значень. У деяких випадках Об'єкти Спостереження можуть діяти як EventEmitters, а саме, коли вони багатоадресні за допомогою RxJS Subjects, але зазвичай вони не діють як EventEmitters.

> Об'єкти Спостереження схожі на функції з нульовими аргументами, але узагальніть їх, щоб дозволити кілька значень.

Зверніть увагу на наступне:
```javascript
function foo() {
  console.log('Hello');
  return 42;
}

const x = foo.call(); // те ж саме, що і foo()
console.log(x);
const y = foo.call(); // те ж саме, що і foo()
console.log(y);
```
Виведе наступне:
```
"Hello"
42
"Hello"
42
```
Ви можете написати ту саму поведінку вище, але з Об'єктом Спостереження:
```javascript
import { Observable } from 'rxjs';

const foo = new Observable(subscriber => {
  console.log('Hello');
  subscriber.next(42);
});

foo.subscribe(x => {
  console.log(x);
});
foo.subscribe(y => {
  console.log(y);
});
```
Результат буде таким самим:
```
"Hello"
42
"Hello"
42
```
Це відбувається тому, що і функції, і Об'єкти Спостереження є ледачими обчисленнями. Якщо ви не викличете функцію, `console.log('Hello')` не відбудеться. Крім того, з Об'єктом Спостереження, якщо ви не «викликаєте» його (за допомогою `subscribe`), `console.log('Hello')` не відбудеться. Крім того, «виклик» або «підписка» є окремою операцією: два виклики функцій викликають два окремих побічних ефекту, а дві підписки Об'єктів Спостереження викликають два окремі побічні ефекти. На відміну від EventEmitters, які мають спільні побічні ефекти та нетерпляче виконуються незалежно від наявності підписників, Об'єкти Спостереження не мають спільного виконання та є ледачими.

> Підписування на Об'єкт Спостереження є аналогом виклику Функції.

Деякі люди стверджують, що Об'єкти Спостереження є асинхронними. Це не правда. Наприклад:
```javascript
console.log('before');
console.log(foo.call());
console.log('after');
```
Ви побачите такий результат:
```
"before"
"Hello"
42
"after"
```
Те ж саме з Об'єктом Спостереження:
```javascript
console.log('before');
foo.subscribe(x => {
  console.log(x);
});
console.log('after');
```
Виведе:
```
"before"
"Hello"
42
"after"
```
Що доводить, що підписка на foo була повністю синхронною, як і функція.

> Об'єкти Спостереження можуть віддавати значення як синхронно, так і асинхронно.

Яка різниця між Об'єкт Спостереження і функцією? Спостережувані можуть «повертати» кілька значень з часом, чого функції не можуть. Ви не можете зробити це:
```javascript
function foo() {
  console.log('Hello');
  return 42;
  return 100; // dead code. will never happen
}
```
Функції можуть повертати лише одне значення. Об'єкти Спостереження, однак, можуть це зробити:
```javascript
import { Observable } from 'rxjs';

const foo = new Observable(subscriber => {
  console.log('Hello');
  subscriber.next(42);
  subscriber.next(100); // "return" another value
  subscriber.next(200); // "return" yet another
});

console.log('before');
foo.subscribe(x => {
  console.log(x);
});
console.log('after');
```
Поверне:
```
"before"
"Hello"
42
100
200
"after"
```
Але ви також можете "повертати" значення асинхронно:
```javascript
import { Observable } from 'rxjs';

const foo = new Observable(subscriber => {
  console.log('Hello');
  subscriber.next(42);
  subscriber.next(100);
  subscriber.next(200);
  setTimeout(() => {
    subscriber.next(300); // happens asynchronously
  }, 1000);
});

console.log('before');
foo.subscribe(x => {
  console.log(x);
});
console.log('after');
```
Виведе:
```
"before"
"Hello"
42
100
200
"after"
300
```
висновок:
`func.call()` означає "дайте мені одне значення синхронно" 
`observable.subscribe()` означає "дати мені будь-яку кількість значень, синхронно чи асинхронно"

## Анатомія Об'єкту Спостереження
Об'єкти Спостереження **створюються** за допомогою `new Observable` або оператора створення(`of(), from(), interval()` etc), на них підписується Спостерігач, **виконуються** для доставки сповіщень `next`/`error`/`complete` до Спостерігача, і їхнє виконання може бути припинено. Усі ці чотири аспекти закодовані в екземплярі Об'єкту Спостереження, але деякі з цих аспектів пов’язані з іншими типами, наприклад Спостерігач і Підписка.
Основні завдання Об'єкту Спостереження:
- Створення Об'єктів Спостереження 
- Підписка на Об'єкти Спостереження 
- Виконання Об'єктів Спостереження 
- Утилізація Об'єктів Спостереження
### Створення Об'єктів Спостереження
Конструктор Observable приймає один аргумент: функцію `subscribe`.
У наступному прикладі створюється Observable, який щосекунди надсилає передплатнику рядок «hi».
```javascript
import { Observable } from 'rxjs';

const observable = new Observable(function subscribe(subscriber) {
  const id = setInterval(() => {
    subscriber.next('hi')
  }, 1000);
});
```
У наведеному вище прикладі функція підписки є найважливішою частиною для опису Observable. Давайте розглянемо, що означає підписка.
### Subscribing to Observables
Обсервабель Observable у прикладі можна підписатися на такий:
```javascript
observable.subscribe(x => console.log(x));
```
Це не випадково, що `observable.subscribe` і `subscribe` в `new Observable(function subscribe(subscriber) {...})` мають однакові назви. У бібліотеці вони різні, але для практичних цілей ви можете вважати їх концептуально рівноправними.
Це показує, як виклики підписки не розподіляються між декількома спостерігачами одного Observable. Під час виклику `observable.subscribe` за допомогою Observer функція `subscribe` у `new Observable(function subscribe(subscriber) {...})` виконується для цього передплатника. Кожен виклик `observable.subscribe` запускає власну незалежну установку для цього передплатника.
Підписка на Observable схожа на виклик функції, надаючи зворотні виклики, куди будуть доставлені дані.
Це суттєво відрізняється від API обробників подій, таких як addEventListener / removeEventListener. З `observable.subscribe` даний Observer не зареєстрований як слухач у Observable. Observable навіть не підтримує список приєднаних спостерігачів.
Виклик підписки — це просто спосіб розпочати «спостережуване виконання» та доставити значення або події спостерігачу цього виконання.
### Executing Observable
Код усередині `new Observable(function subscribe(subscriber) {...})` представляє «виконання Observable», відкладене обчислення, яке відбувається лише для кожного спостерігача, який підписався. Виконання створює кілька значень з часом, синхронно чи асинхронно.
Існує три типи значень, які може надати спостережуване виконання:
- «Наступне» сповіщення: надсилає таке значення, як число, рядок, об’єкт тощо. 
- Сповіщення про помилку: надсилає повідомлення про помилку JavaScript або виняток. 
- «Повне» сповіщення: не надсилає значення.
Сповіщення «Далі» є найважливішим і найпоширенішим типом: вони представляють фактичні дані, які доставляються абоненту. Сповіщення «Помилка» та «Завершено» можуть відбутися лише один раз під час спостережуваного виконання, і може бути лише одне з них.
У спостережуваному виконанні можуть бути доставлені наступні сповіщення від нуля до нескінченності. Якщо доставлено сповіщення про помилку або завершено, після цього більше нічого не буде доставлено.
Нижче наведено приклад виконання Observable, яке надсилає три сповіщення Next, а потім завершує:
```javascript
import { Observable } from 'rxjs';

const observable = new Observable(function subscribe(subscriber) {
  subscriber.next(1);
  subscriber.next(2);
  subscriber.next(3);
  subscriber.complete();
});
```
Observables суворо дотримуються Observable Contract, тому наступний код не доставить наступне сповіщення 4:
```javascript
import { Observable } from 'rxjs';

const observable = new Observable(function subscribe(subscriber) {
  subscriber.next(1);
  subscriber.next(2);
  subscriber.next(3);
  subscriber.complete();
  subscriber.next(4); // Is not delivered because it would violate the contract
});
```
Гарною ідеєю буде загорнути будь-який код у підписку блоком `try/catch`, який надсилатиме сповіщення про помилку, якщо виловить виняток:
```javascript
import { Observable } from 'rxjs';

const observable = new Observable(function subscribe(subscriber) {
  try {
    subscriber.next(1);
    subscriber.next(2);
    subscriber.next(3);
    subscriber.complete();
  } catch (err) {
    subscriber.error(err); // delivers an error if it caught one
  }
});
```
### Disposing  Observable Execution
Оскільки спостережувані виконання можуть бути нескінченними, а спостерігач зазвичай хоче перервати виконання за кінцевий час, нам потрібен API для скасування виконання. Оскільки кожне виконання є ексклюзивним лише для одного спостерігача, щойно спостерігач закінчить отримання значень, він повинен мати спосіб зупинити виконання, щоб уникнути марної витрати обчислювальної потужності або ресурсів пам’яті.
Коли викликається observable.subscribe, Observer приєднується до щойно створеного виконання Observable. Цей виклик також повертає об’єкт, Subscription:
```javascript
const subscription = observable.subscribe(x => console.log(x));
```
Підписка представляє поточне виконання та має мінімальний API, який дозволяє скасувати це виконання. Детальніше про тип підписки читайте тут. За допомогою `subscription.unsubscribe()` ви можете скасувати поточне виконання:
```javascript
import { from } from 'rxjs';

const observable = from([10, 20, 30]);
const subscription = observable.subscribe(x => console.log(x));
// Later:
subscription.unsubscribe();
```
Коли ви підписуєтеся, ви отримуєте назад підписку, яка представляє поточне виконання. Просто викличте `unsubscribe()`, щоб скасувати виконання.
Кожен Observable повинен визначати, як розпоряджатися ресурсами цього виконання, коли ми створюємо Observable за допомогою create(). Ви можете зробити це, повернувши спеціальну функцію скасування підписки з функції `subscribe()`.
Наприклад, ось як ми очищаємо набір інтервальних виконання за допомогою `setInterval`:
```javascript
const observable = new Observable(function subscribe(subscriber) {
  // Keep track of the interval resource
  const intervalId = setInterval(() => {
    subscriber.next('hi');
  }, 1000);

  // Provide a way of canceling and disposing the interval resource
  return function unsubscribe() {
    clearInterval(intervalId);
  };
});
```
Подібно до того, як observable.subscribe нагадує `new Observable(function subscribe() {...})`, `unsubscribe`, який ми повертаємо з `subscribe`, концептуально дорівнює `subscription.unsubscribe`. Насправді, якщо ми видалимо типи ReactiveX, що оточують ці концепції, ми залишимо досить простий JavaScript.
```javascript
function subscribe(subscriber) {
  const intervalId = setInterval(() => {
    subscriber.next('hi');
  }, 1000);

  return function unsubscribe() {
    clearInterval(intervalId);
  };
}

const unsubscribe = subscribe({next: (x) => console.log(x)});

// Later:
unsubscribe(); // dispose the resources
```
Причина, чому ми використовуємо типи Rx, такі як Observable, Observer і Subscription, полягає в тому, щоб забезпечити безпеку (наприклад, Observable Contract) і можливість комбінування з операторами.
