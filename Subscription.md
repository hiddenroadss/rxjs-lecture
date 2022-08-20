# Subscription
Що таке підписка? Підписка — це об’єкт, який представляє доступний ресурс, як правило, виконання Observable. 
Підписка має один важливий метод, скасування підписки, який не приймає аргументів і просто позбавляє ресурс, який містить підписка. 
У попередніх версіях RxJS підписка називалася «одноразова».
```javascript
import { interval } from 'rxjs';

const observable = interval(1000);
const subscription = observable.subscribe(x => console.log(x));
// Later:
// This cancels the ongoing Observable execution which
// was started by calling subscribe with an Observer.
subscription.unsubscribe();
```
> По суті, підписка має лише функцію unsubscribe(), щоб звільнити ресурси або скасувати спостережувані виконання.

Підписки також можна поєднувати, щоб виклик unsubscribe() однієї Підписки міг скасувати підписку на кілька Підписок. 
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
Після виконання ми бачимо в консолі:
```
second: 0
first: 0
second: 1
first: 1
second: 2
```
