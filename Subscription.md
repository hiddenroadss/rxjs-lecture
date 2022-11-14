# Підписка

Підписка — це об’єкт, який повертає виклик методу `Observable.prototype.subscribe`.

Підписка має один важливий метод `unsibscribe`, який не приймає аргументів і просто позбавляє ресурс, який містить підписка.

Фактично, це дуже схоже на те, як ми підписуємося на якусь гахету/журнал/тощо. Ми підписуємося і нам віддають підписку - документ, який підтверджує нашу дію. І коли ми потім захочемо скасувати нашу підписку - нам потрібно буде використати цей документ??

```javascript
import { interval } from 'rxjs';

const observable = interval(1000);
const subscription = observable.subscribe((x) => console.log(x));
// Later:
// This cancels the ongoing Observable execution which
// was started by calling subscribe with an Observer.
subscription.unsubscribe();
```

> По суті, підписка має лише функцію unsubscribe(), щоб звільнити ресурси або скасувати роботу Об'єкту Спостереження.

Підписки також можна поєднувати, щоб виклик `unsubscribe()` однієї підписки міг скасувати свою підписку та додані нами підписки.
Це можна зробити, «додавши» одну підписку до іншої:

```javascript
import { interval } from 'rxjs';

const observable1 = interval(400);
const observable2 = interval(300);

const subscription = observable1.subscribe((x) => console.log('first: ' + x));
const childSubscription = observable2.subscribe((x) => console.log('second: ' + x));

subscription.add(childSubscription);

setTimeout(() => {
	// Unsubscribes BOTH subscription and childSubscription
	subscription.unsubscribe();
}, 1000);
```

Після виконання ми побачимо в консолі:

```
second: 0
first: 0
second: 1
first: 1
second: 2
```

Підписки також мають метод `remove(otherSubscription)`, щоб скасувати додавання дочірньої підписки.
