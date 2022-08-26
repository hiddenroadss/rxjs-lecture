# Суб'єкт

Що таке Суб'єкт? RxJS Суб'єкт — це особливий тип Об'єкту Спостереження, який дозволяє передавати значення багатьом Спостерігачам. Хоча звичайні Об'єкти Спостереження є одноадресними (кожен підписаний Спостерігач володіє незалежним виконанням Об'єкту Спостереження), Суб'єкти є багатоадресними.

> Суб'єкт схожий на Об'єкт Спостереження, але може бути спільним для багатьох спостерігачів. Суб’єкти схожі на EventEmitters: вони підтримують реєстр багатьох слухачів.

**Кожен Суб'єкт є Об'єктом Спостереження**. Маючи Суб'єкт, ви можете підписатися на нього, передавши Спостерігача, який почне отримувати значення. З точки зору Спостерігача, він не може визначити, чи виконання Об'єкт Спостереження походить від простого одноадресного Об'єкта Спостереження чи Суб'єкта.

Всередині Суб'єкта `subscribe` не викликає нового виконання, яке доставляє значення. Він просто реєструє Спостерігача у списку Спостерігачів, подібно до того, як `addListener` зазвичай працює в інших бібліотеках і мовах.

**Кожен Суб'єкт є Спостерігачем**. Це об’єкт із методами `next(v)`, `error(e)` і `complete()`. Щоб передати нове значення Суб’єкту, просто викличте `next(theValue)`, і воно буде передано багатоадресним Спостерігачам, зареєстрованим для прослуховування Суб’єкта.

У наведеному нижче прикладі ми маємо двох Спостерігачів, приєднаних до Суб’єкта, і ми передаємо деякі значення Суб’єкту:
```javascript
import { Subject } from 'rxjs';

const subject = new Subject<number>();

subject.subscribe({
  next: (v) => console.log(`observerA: ${v}`),
});
subject.subscribe({
  next: (v) => console.log(`observerB: ${v}`),
});

subject.next(1);
subject.next(2);

// Logs:
// observerA: 1
// observerB: 1
// observerA: 2
// observerB: 2
```
Оскільки Суб'єкт є Спостерігачем, це також означає, що ви можете надати Суб'єкт як аргумент для підписки будь-якого Об'єкт Спостереження, як показано в прикладі нижче:
```javascript
import { Subject, from } from 'rxjs';

const subject = new Subject<number>();

subject.subscribe({
  next: (v) => console.log(`observerA: ${v}`),
});
subject.subscribe({
  next: (v) => console.log(`observerB: ${v}`),
});

const observable = from([1, 2, 3]);

observable.subscribe(subject); // You can subscribe providing a Subject

// Logs:
// observerA: 1
// observerB: 1
// observerA: 2
// observerB: 2
// observerA: 3
// observerB: 3

```

З наведеним вище підходом ми, по суті, просто перетворили одноадресне виконання Об'єкта Спостереження на багатоадресне через Суб'єкт. Це демонструє, як Суб'єкти є єдиним способом зробити будь-яке виконання Об'єкта Спостереження доступним для кількох Спостерігачів.

Існує також кілька спеціалізацій типу Суб'єкт: BehaviorSubject, ReplaySubject і AsyncSubject.

## Multicasted Observables

«Multicasted Observable» передає сповіщення через Суб'єкт, який може мати багато Спостерігачів, тоді як простий «unicast Observable» надсилає сповіщення лише одному Спостерігачу.

Багатоадресний Об'єкт Спостереження використовує Суб'єкт під капотом, щоб кілька Спостерігачів бачили те саме виконання Об'єкта Спостереження.

Під капотом ось як працює оператор багатоадресної передачі: Спостерігачі підписуються на базовий Суб'єкт, а Суб'єкт підписується на джерело Об'єкта Спостереження. Наступний приклад схожий на попередній, у якому використовувався `observable.subscribe(subject)`:
```javascript
import { from, Subject, multicast } from 'rxjs';

const source = from([1, 2, 3]);
const subject = new Subject();
const multicasted = source.pipe(multicast(subject));

// These are, under the hood, `subject.subscribe({...})`:
multicasted.subscribe({
  next: (v) => console.log(`observerA: ${v}`),
});
multicasted.subscribe({
  next: (v) => console.log(`observerB: ${v}`),
});

// This is, under the hood, `source.subscribe(subject)`:
multicasted.connect();
```

`multicast` повертає Об'єкт Спостереження, який виглядає як звичайний Об'єкт Спостереження, але працює як Суб'єкт, коли справа доходить до підписки. `multicast` повертає ConnectableObservable, який є просто Об'єкт Спостереження з методом `connect()`.

Метод `connect()` важливий, щоб точно визначити, коли почнеться виконання спільного Об'єкта Спостереження. Оскільки `connect()` виконує `source.subscribe(subject)` під капотом, `connect()` повертає підписку, від якої ви можете скасувати підписку, щоб скасувати спільне виконання Об'єкта Спостереження.

### Reference counting

Виклик `connect()` вручну та обробка підписки часто громіздкі. Зазвичай ми хочемо автоматично підключатися, коли приходить перший Спостерігач, і автоматично скасовувати спільне виконання, коли останній Спостерігач скасовує підписку.

Розглянемо наступний приклад, коли підписки відбуваються, як зазначено в цьому списку:
1. First Observer підписується на багатоадресний Об'єкт Спостереження
2. Багатоадресний Об'єкт Спостереження підключено
3. Наступне значення 0 доставляється першому спостерігачу
4. Другий спостерігач підписується на багатоадресний Об'єкт Спостереження
5. Наступне значення 1 доставляється першому спостерігачу
6. Наступне значення 1 передається другому спостерігачу
7. First Observer скасовує підписку на багатоадресний Об'єкт Спостереження
8. Наступне значення 2 доставляється другому спостерігачу
9. Другий спостерігач скасовує підписку на групову передачу Об'єкт Спостереження
10. Підписку на підключення до багатоадресного Об'єкт Спостереження скасовано

Щоб досягти цього за допомогою явних викликів `connect()`, ми пишемо такий код:
```javascript
import { interval, Subject, multicast } from 'rxjs';

const source = interval(500);
const subject = new Subject();
const multicasted = source.pipe(multicast(subject));
let subscription1, subscription2, subscriptionConnect;

subscription1 = multicasted.subscribe({
  next: (v) => console.log(`observerA: ${v}`),
});
// We should call `connect()` here, because the first
// subscriber to `multicasted` is interested in consuming values
subscriptionConnect = multicasted.connect();

setTimeout(() => {
  subscription2 = multicasted.subscribe({
    next: (v) => console.log(`observerB: ${v}`),
  });
}, 600);

setTimeout(() => {
  subscription1.unsubscribe();
}, 1200);

// We should unsubscribe the shared Observable execution here,
// because `multicasted` would have no more subscribers after this
setTimeout(() => {
  subscription2.unsubscribe();
  subscriptionConnect.unsubscribe(); // for the shared Observable execution
}, 2000);
```

Якщо ми хочемо уникнути явних викликів `connect()`, ми можемо використати метод `refCount()` ConnectableObservable (підрахунок посилань), який повертає Об'єкт Спостереження, який відстежує кількість підписників. Коли кількість підписників зросте з 0 до 1, він викличе для нас функцію `connect()`, яка розпочне спільне виконання. Лише коли кількість підписників зменшиться з 1 до 0, підписку буде повністю скасовано, припиняючи подальше виконання.

`refCount` змушує багатоадресний Об'єкт Спостереження автоматично починати виконання, коли приходить перший підписник, і припиняти виконання, коли останній підписник залишає.

Нижче наведено приклад:
```javascript
import { interval, Subject, multicast, refCount } from 'rxjs';

const source = interval(500);
const subject = new Subject();
const refCounted = source.pipe(multicast(subject), refCount());
let subscription1, subscription2;

// This calls `connect()`, because
// it is the first subscriber to `refCounted`
console.log('observerA subscribed');
subscription1 = refCounted.subscribe({
  next: (v) => console.log(`observerA: ${v}`),
});

setTimeout(() => {
  console.log('observerB subscribed');
  subscription2 = refCounted.subscribe({
    next: (v) => console.log(`observerB: ${v}`),
  });
}, 600);

setTimeout(() => {
  console.log('observerA unsubscribed');
  subscription1.unsubscribe();
}, 1200);

// This is when the shared Observable execution will stop, because
// `refCounted` would have no more subscribers after this
setTimeout(() => {
  console.log('observerB unsubscribed');
  subscription2.unsubscribe();
}, 2000);

// Logs
// observerA subscribed
// observerA: 0
// observerB subscribed
// observerA: 1
// observerB: 1
// observerA unsubscribed
// observerB: 2
// observerB unsubscribed
```
Метод `refCount()` існує лише для ConnectableObservable і повертає Об'єкт Спостереження, а не інший ConnectableObservable.

## Behaviour Суб'єкт
Одним із варіантів Суб'єкта є BehaviorSubject, який має поняття «поточне значення». Він зберігає останнє значення, надіслане своїм споживачам, і щоразу, коли новий Спостерігач підписується, він негайно отримуватиме «поточне значення» від BehaviorSubject.

> BehaviorSubjects корисні для представлення значень протягом часу». Наприклад, потік подій днів народження є Суб'єкт, але потік віку людини буде BehaviorSubject.

У наступному прикладі BehaviorSubject ініціалізується значенням 0, яке отримує перший спостерігач під час підписки. Другий спостерігач отримує значення 2, навіть якщо він підписався після того, як було надіслано значення 2.
```javascript
import { BehaviorSubject } from 'rxjs';
const subject = new BehaviorSubject(0); // 0 is the initial value

subject.subscribe({
  next: (v) => console.log(`observerA: ${v}`),
});

subject.next(1);
subject.next(2);

subject.subscribe({
  next: (v) => console.log(`observerB: ${v}`),
});

subject.next(3);

// Logs
// observerA: 0
// observerA: 1
// observerA: 2
// observerB: 2
// observerA: 3
// observerB: 3
```

## Replay Суб'єкт

ReplaySubject схожий на BehaviorSubject тим, що він може надсилати старі значення новим підписникам, але також може записувати частину виконання Об'єкту Спостереження.

ReplaySubject записує кілька значень із виконання Об'єкту Спостереження і відтворює їх новим підписникам.

Створюючи ReplaySubject, ви можете вказати, скільки значень відтворювати:
```javascript
import { ReplaySubject } from 'rxjs';
const subject = new ReplaySubject(3); // buffer 3 values for new subscribers

subject.subscribe({
  next: (v) => console.log(`observerA: ${v}`),
});

subject.next(1);
subject.next(2);
subject.next(3);
subject.next(4);

subject.subscribe({
  next: (v) => console.log(`observerB: ${v}`),
});

subject.next(5);

// Logs:
// observerA: 1
// observerA: 2
// observerA: 3
// observerA: 4
// observerB: 2
// observerB: 3
// observerB: 4
// observerA: 5
// observerB: 5
```

Ви також можете вказати час вікна в мілісекундах, крім розміру буфера, щоб визначити, скільки років можуть бути записані значення. У наступному прикладі ми використовуємо великий розмір буфера 100, але параметр часу вікна становить лише 500 мілісекунд.
```javascript
import { ReplaySubject } from 'rxjs';
const subject = new ReplaySubject(100, 500 /* windowTime */);

subject.subscribe({
  next: (v) => console.log(`observerA: ${v}`),
});

let i = 1;
setInterval(() => subject.next(i++), 200);

setTimeout(() => {
  subject.subscribe({
    next: (v) => console.log(`observerB: ${v}`),
  });
}, 1000);

// Logs
// observerA: 1
// observerA: 2
// observerA: 3
// observerA: 4
// observerA: 5
// observerB: 3
// observerB: 4
// observerB: 5
// observerA: 6
// observerB: 6
// ...
```

## Async Суб'єкт
AsyncSubject — це варіант, у якому Спостерігачам надсилається лише останнє значення виконання Об'єкту Спостереження, і лише після завершення виконання.
```javascript
import { AsyncSubject } from 'rxjs';
const subject = new AsyncSubject();

subject.subscribe({
  next: (v) => console.log(`observerA: ${v}`),
});

subject.next(1);
subject.next(2);
subject.next(3);
subject.next(4);

subject.subscribe({
  next: (v) => console.log(`observerB: ${v}`),
});

subject.next(5);
subject.complete();

// Logs:
// observerA: 5
// observerB: 5
```

AsyncSubject подібний до оператора `last()` тим, що він очікує на повне сповіщення, щоб доставити одне значення.

## Пустий Суб'єкт

Іноді видане значення не має такого значення, як факт, що значення було випущено.

Наприклад, наведений нижче код сигналізує, що минула одна секунда.
```javascript
const subject = new Subject<string>();
setTimeout(() => subject.next('dummy'), 1000);
```
Передача фіктивного значення таким чином є незграбною та може заплутати користувачів.

Оголошуючи Пустий Суб'єкт, ви сигналізуєте, що значення не має значення. Важлива лише сама подія.
```javascript
const subject = new Subject<void>();
setTimeout(() => subject.next(), 1000);
```
Повний приклад із контекстом наведено нижче:
```javascript
import { Subject } from 'rxjs';

const subject = new Subject(); // Shorthand for Subject<void>

subject.subscribe({
  next: () => console.log('One second has passed'),
});

setTimeout(() => subject.next(), 1000);
```