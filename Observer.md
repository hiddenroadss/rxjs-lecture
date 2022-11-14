Спостерігач — це споживач даних, які надає Об'єкт Спостереження. Фактично, це просто набір зворотних викликів(callbacks), по одному для кожного типу сповіщень,
які доставляє Об'єкт Спостереження: `next`, `error` і `complete`. Нижче наведено приклад Спостерігача:

```javascript
const observer = {
	next: (x) => console.log('Observer got a next value: ' + x),
	error: (err) => console.error('Observer got an error: ' + err),
	complete: () => console.log('Observer got a complete notification'),
};
```

Щоб використовувати Спостерігача, передайте його в метод `subscribe` Об'єкту Спостереження:

```javascript
observable.subscribe(observer);
```

Вам не обов'язково передавати всі три зворотніх виклики. Якщо ви не надасте один із них (або взагалі ні одного), виконання Об'єкту Спостереження все одно
відбуватиметься нормально, за винятком того, що деякі типи сповіщень ігноруватимуться, оскільки вони не мають відповідного зворотного виклику в Спостерігачі.

Наведений нижче приклад є Спостерігачем без повного зворотного виклику:

```javascript
const observer = {
	next: (x) => console.log('Observer got a next value: ' + x),
	error: (err) => console.error('Observer got an error: ' + err),
};
```

Підписуючись на Об'єкт Спостереження, ви також можете просто надати зворотний виклик `next` як аргумент, не передаючи об'єкт Спостерігача, наприклад так:

```javascript
observable.subscribe((x) => console.log('Observer got a next value: ' + x));
```

Всередині `observable.subscribe` він створить об’єкт Спостерігача, використовуючи аргумент зворотного виклику як обробник виклику `next`.
