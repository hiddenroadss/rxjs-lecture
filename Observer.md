Що таке спостерігач? Спостерігач — це споживач цінностей, які надає Observable. Спостерігачі — це просто набір зворотних викликів, по одному для кожного типу сповіщень, 
які доставляє Observable: `next`, `error` і `complete`. Нижче наведено приклад типового об’єкта Observer:
```javascript
const observer = {
  next: x => console.log('Observer got a next value: ' + x),
  error: err => console.error('Observer got an error: ' + err),
  complete: () => console.log('Observer got a complete notification'),
};
```
Щоб використовувати Observer, надайте його підписнику Observable:
```javascript
observable.subscribe(observer);
```
> Спостерігачі — це лише об’єкти з трьома зворотними викликами, по одному для кожного типу сповіщень, які може доставити Observable.

Спостерігачі в RxJS також можуть бути частковими. Якщо ви не надасте один із зворотних викликів, виконання Observable все одно
відбуватиметься нормально, за винятком того, що деякі типи сповіщень ігноруватимуться, оскільки вони не мають відповідного зворотного виклику в Observer.

Наведений нижче приклад є спостерігачем без повного зворотного виклику:
```javascript
const observer = {
  next: x => console.log('Observer got a next value: ' + x),
  error: err => console.error('Observer got an error: ' + err),
};
```
Підписуючись на Observable, ви також можете просто надати наступний зворотний виклик як аргумент, не будучи приєднаним до об’єкта Observer, наприклад так:
```javascript
observable.subscribe(x => console.log('Observer got a next value: ' + x));
```
Всередині `observable.subscribe` він створить об’єкт Observer, використовуючи аргумент зворотного виклику як наступний обробник.
