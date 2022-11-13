Javascript - дуже активно використовує асинхронний код. В нас є тільки один головний потік, тому все, що може його заблокувати виконується асинхронно.
Давайте згадаємо, де ми використовуємо асинхронний код:

-   `setTimeout`, `setInterval`
-   `Promises`
-   `Event Listeners`
-   `Http Requests`
-   etc

Якщо нам потрібно зробити щось просте і тривіальне, то робота з цими API не викликає дискомфорту.
Наприклад, нам потрібно отримати дані з серверу:

```javascript
fetch('https://jsonplaceholder.typicode.com/todos/1')
	.then((data) => data.json())
	.then(console.log);
```

Або реагувати, коли користувач щось натискає:

```javascript
const button = document.querySelector('button');
button.addEventListener('click', () => {
	console.log('clicked');
});
```

Або почекати, перед тим, як виконати якусь дію:

```javascript
setTimeout(() => console.log('timeout, 2s'), 2000);
```

Більшість з цих Web API з'явилися досить давно(`setTimeout`, `XMLHttpRequest`, `addEventListener`), деякі - недавно(`fetch`, `Promises`, `async/await`). Javascript активно розвивався останні 20 років і якщо на початку задачі JS були накшталт "Анімувати кнопку", "Зробити бургер-меню", то зараз - це можуть бути досить складні програми, які можуть мати 10к+ рядків коду.

За весь цей період деякі речі змінилися, наприклад ми зараз набагато більше використовуємо `Promise` або` async/await` замість зворотніх викликів(callbacks). Їх створили для того, щоб вирішувати проблеми, які ми маємо при роботі з асинхронним кодом і зворотніми викликами.

Але ми не можемо уніфікувати всю роботу з Web Api для роботи з `Promise`, наприклад. Тому що в нас залишаються `addEventListener`, який працює з зворотніми викликами. Залишається `XmlHttpRequest`, який ми використовуємо для `ajax` запитів на сервер і який має своє API:

```js
const req = new XMLHttpRequest();

req.onload = (e) => {
	const arraybuffer = req.response;
	/* ... */
};
req.open('GET', url);
req.responseType = 'arraybuffer';
req.send();
```

Звісно, зараз з'явився метод `fetch`, який базується на `XMLHttpRequest` і `Promise`, тому і має більш зручне API.

Але важливо зрозуміти те, що на даний момент у нас немає абстракції в нативному JS, яка б дала нам змогу використовувати всі ці речі з однаковим підходом.

Це одна з переваг RxJS:

> Бібліотека дає нам змогу абстрагуватися від імплементації конкретного API і працювати з ними однаково.

Приклад створення потоку з різних джерел:

```js
import { from, fromEvent, interval } from 'rxjs';
import { ajax } from 'rxjs/ajax';

interval(1000).subscribe(console.log);

fromEvent(document, 'click').subscribe(console.log);

ajax('https://random-data-api.com/api/v2/users').subscribe(console.log);

from(new Promise((res, rej) => res(10))).subscribe(console.log);
```

Друга проблема витікає з першої.

Якщо в нас немає одного API для всіх асинхронних операцій в JS, значить нам потрібно комбінувати різні підходи разом. Часто це буває не зручно і не читабельно.

Давайте подивимося на приклад того, як приблизно може виглядати рішення задачі "Створення поля пошуку" нативними методами:

```html
<!DOCTYPE html>
<html lang="en">
	<head>
		<meta charset="UTF-8" />
		<meta http-equiv="X-UA-Compatible" content="IE=edge" />
		<meta name="viewport" content="width=device-width, initial-scale=1.0" />
		<title>Document</title>
		<script src="index.js" defer></script>
	</head>
	<body>
		<div>
			<input type="text" id="search" />
			<div id="container"></div>
		</div>
	</body>
</html>
```

```js
const searchInput = document.getElementById('search');
const container = document.getElementById('container');
let timeoutId = null;
let searchValue;
let controller = new AbortController();

const inputCb = (e) => {
	if (timeoutId) {
		clearTimeout(timeoutId);
	}
	timeoutId = setTimeout(() => {
		if (e.target.value === searchValue) {
			return;
		}
		controller.abort();
		controller = new AbortController();

		searchValue = e.target.value;

		fetch(`https://random-data-api.com/api/v2/${e.target.value}`, {
			method: 'get',
			signal: controller.signal,
		})
			.then((value) => value.json())
			.then((value) => (container.innerText = `Username is: ${value.username}`));
	}, 1000);
};

searchInput.addEventListener('input', inputCb);
```

Це умовний приклад і я впевнений, що його можна написати краще, але для цілей цієї статті він підійде. Що тут відбувається? Одразу важко сказати, але давайте розберемося:

1. У нас є поле пошуку, яке ми будемо слухати за допомогою `addEventListener` і зворотнього виклику `inputCb`. Кожного разу, як ми будемо щось вводити в поле - `inputCb` буде викликатися
2. Нам не потрібно, щоб на кожну нову літеру ми робили запит на сервер, тому що це непотрібне навантаження на сервер. Тому ми використовуємо `setTimeout` для того, щоб затримати виконання запиту на секунду після кожного натискання клавіші. Фактично, ми зробимо запит, коли користувач перестане друкувати(або якщо він буде друкувати повільно).
3. Далі ми робимо запит на сервер через `fetch`. Він використовує `Promise` всередині, тому ми підписуємося через `then` на дані і відображаємо їх на сторінці.
4. Також, якщо користувач введе якесь слово, а потім почне його змінювати, але поверне назад("users"->"userssome"->"users") не роблячи паузу(працює наш `setTimeout`), то ми не будемо робити такий же самий запит з `users`, тому що ми вже отримали данні за таким запитом.
5. І нарешті, якщо в нас буде запит на сервер і повільна мережа, ми не хочемо, щоб старий запит прислав відповідь пізніше нового(ніхто не гарантує те, що відповідь на перший HTTP запит прийде першою). Для цього ми використовуємо `AbortController`, який буде відміняти попередній запит, якщо почав виконуватися новий.

Якщо розбити цей механізм на кроки і прочитати - то все досить логічно, але в коді вище розбиратися досить складно.
Ось як це можна переписати з RxJS:

```js
import { fromEvent } from 'rxjs';
import { ajax } from 'rxjs/ajax';
import { distinctUntilChanged, debounceTime, switchMap } from 'rxjs/operators';

const searchInput = document.getElementById('search');
const container = document.getElementById('container');

fromEvent(searchInput, 'input')
	.pipe(
		debounceTime(1000),
		distinctUntilChanged(),
		switchMap((inputText) => {
			return ajax(`https://random-data-api.com/api/v2/${inputText}`);
		})
	)
	.subscribe((value) => (container.innerText = `Username is: ${value.username}`));
```

При цьому RxJs не створює якісь інші способи, як робити ті ж самі речі. Всередині будуть відбуватися ті ж самі `setTimeout`, `addEventListener`, `AbortController`, тощо. Він надає нам додатковий рівень абстракції, який допомагає писати нам декларативний, а не імперативний код. А декларативний код набагато легше читати і розуміти.
