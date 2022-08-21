# Підписка
Що таке підписка? Підписка — це об’єкт, який представляє одноразовий ресурс??, як правило, виконання Observable. 
Підписка має один важливий метод `unsibscribe`, який не приймає аргументів і просто позбавляє ресурс, який містить підписка. 
```javascript
import { interval } from 'rxjs';

const observable = interval(1000);
const subscription = observable.subscribe(x => console.log(x));
// Later:
// This cancels the ongoing Observable execution which
// was started by calling subscribe with an Observer.
subscription.unsubscribe();
```
> По суті, підписка має лише функцію unsubscribe(), щоб звільнити ресурси або скасувати роботу Спостережуваного.

Підписки також можна поєднувати, щоб виклик `unsubscribe()` однієї підписки міг скасувати свою підписку та додані нами підписки. 
Це можна зробити, «додавши» одну підписку до іншої:
```javascript
import { interval } from 'rxjs';

const observable1 = interval(400);
const observable2 = interval(300);

const subscription = observable1.subscribe(x => console.log('first: ' + x));
const childSubscription = observable2.subscribe(x => console.log('second: ' + x));

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
