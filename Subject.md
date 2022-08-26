# Subject

Що таке предмет? RxJS Subject — це особливий тип Observable, який дозволяє передавати значення багатьом спостерігачам. Хоча звичайні Observable є одноадресними (кожен підписаний Observer володіє незалежним виконанням Observable), Subjects є багатоадресними.

Subject схожий на Observable, але може багаторазово передавати багато спостерігачів. Суб’єкти схожі на EventEmitters: вони підтримують реєстр багатьох слухачів.

Кожен предмет є спостережуваним. Маючи предмет, ви можете підписатися на нього, надавши спостерігача, який почне нормально отримувати значення. З точки зору Спостерігача, він не може визначити, чи виконання Observable походить від простого одноадресного Observable чи Subject.

Всередині суб’єкта підписка не викликає нового виконання, яке доставляє значення. Він просто реєструє даний спостерігач у списку спостерігачів, подібно до того, як addListener зазвичай працює в інших бібліотеках і мовах.

Кожен суб'єкт є спостерігачем. Це об’єкт із методами next(v), error(e) і complete(). Щоб передати нове значення суб’єкту, просто викличте next(theValue), і воно буде передано багатоадресним спостерігачам, зареєстрованим для прослуховування суб’єкта.

У наведеному нижче прикладі ми маємо двох спостерігачів, приєднаних до суб’єкта, і ми передаємо деякі значення суб’єкту:
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
Оскільки Subject є спостерігачем, це також означає, що ви можете надати Subject як аргумент для підписки будь-якого Observable, як показано в прикладі нижче:
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

З наведеним вище підходом ми, по суті, просто перетворили одноадресне виконання Observable на багатоадресне через Subject. Це демонструє, як Subjects є єдиним способом зробити будь-яке виконання Observable доступним для кількох спостерігачів.

Існує також кілька спеціалізацій типу Subject: BehaviorSubject, ReplaySubject і AsyncSubject.

## Multicasted Observables

«Multicasted Observable» передає сповіщення через Subject, який може мати багато передплатників, тоді як простий «unicast Observable» надсилає сповіщення лише одному спостерігачу.

Багатоадресний Observable використовує Subject під капотом, щоб кілька спостерігачів бачили те саме виконання Observable.

Під капотом ось як працює оператор багатоадресної передачі: спостерігачі підписуються на базовий Subject, а Subject підписується на джерело Observable. Наступний приклад схожий на попередній, у якому використовувався observable.subscribe(subject):
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

multicast повертає Observable, який виглядає як звичайний Observable, але працює як Subject, коли справа доходить до підписки. multicast повертає ConnectableObservable, який є просто Observable з методом connect().

Метод connect() важливий, щоб точно визначити, коли почнеться виконання спільного Observable. Оскільки connect() виконує source.subscribe(subject) під капотом, connect() повертає підписку, від якої ви можете скасувати підписку, щоб скасувати спільне виконання Observable.

### Reference counting

Виклик connect() вручну та обробка підписки часто громіздкі. Зазвичай ми хочемо автоматично підключатися, коли приходить перший спостерігач, і автоматично скасовувати спільне виконання, коли останній спостерігач скасовує підписку.

Розглянемо наступний приклад, коли підписки відбуваються, як зазначено в цьому списку:
First Observer підписується на багатоадресний Observable
Багатоадресний Observable підключено
Наступне значення 0 доставляється першому спостерігачу
Другий спостерігач підписується на багатоадресний Observable
Наступне значення 1 доставляється першому спостерігачу
Наступне значення 1 передається другому спостерігачу
First Observer скасовує підписку на багатоадресний Observable
Наступне значення 2 доставляється другому спостерігачу
Другий спостерігач скасовує підписку на групову передачу Observable
Підписку на підключення до багатоадресного Observable скасовано

Щоб досягти цього за допомогою явних викликів connect(), ми пишемо такий код:
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

Якщо ми хочемо уникнути явних викликів connect(), ми можемо використати метод refCount() ConnectableObservable (підрахунок посилань), який повертає Observable, який відстежує кількість передплатників. Коли кількість підписників зросте з 0 до 1, він викличе для нас функцію connect(), яка розпочне спільне виконання. Лише коли кількість підписників зменшиться з 1 до 0, підписку буде повністю скасовано, припиняючи подальше виконання.

refCount змушує багатоадресний Observable автоматично починати виконання, коли приходить перший передплатник, і припиняти виконання, коли останній передплатник залишає.

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
Метод refCount() існує лише для ConnectableObservable і повертає Observable, а не інший ConnectableObservable.

## Behaviour Subject
Одним із варіантів Subjects є BehaviorSubject, який має поняття «поточне значення». Він зберігає останнє значення, надіслане своїм споживачам, і щоразу, коли новий Спостерігач підписується, він негайно отримуватиме «поточне значення» від BehaviorSubject.

BehaviorSubjects корисні для представлення «цінностей у часі». Наприклад, потік подій днів народження є Subject, але потік віку людини буде BehaviorSubject.

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

## Replay Subject

ReplaySubject схожий на BehaviorSubject тим, що він може надсилати старі значення новим підписникам, але також може записувати частину виконання Observable.

ReplaySubject записує кілька значень із виконання Observable і відтворює їх новим передплатникам.

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

## Async Subject
AsyncSubject — це варіант, у якому спостерігачам надсилається лише останнє значення виконання Observable, і лише після завершення виконання.
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

AsyncSubject подібний до оператора last() тим, що він очікує на повне сповіщення, щоб доставити одне значення.

## Void Subject

Іноді видане значення не має такого значення, як факт, що значення було випущено.

Наприклад, наведений нижче код сигналізує, що минула одна секунда.
```javascript
const subject = new Subject<string>();
setTimeout(() => subject.next('dummy'), 1000);
```
Передача фіктивного значення таким чином є незграбною та може заплутати користувачів.

Оголошуючи недійсну тему, ви сигналізуєте, що значення не має значення. Важлива лише сама подія.
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