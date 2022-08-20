Джаваскрипт - дуже активно використовує асинхронний код. В нас є тільки один головний потік, тому все, що може його заблокувати виконується асинхронно.
Давайте згадаємо, як ми можемо писати асинхронний код:
setTimeout, setInterval
Promises
Event Listeners
Http Requests
etc
Якщо нам потрібно зробити щось просте, то ці способи нам добре підходять.
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
Але набагато частіше нам потрібно комбінувати асинхронні події разом. Наприклад, по кліку на кнопку, нам потрібно почекати 2 секунди, а потім зробити запит на сервер.
```javascript
const button = document.querySelector('button');
button.addEventListener('click', () => {
  setTimeout(() => {
    fetch('https://jsonplaceholder.typicode.com/todos/1')
      .then((data) => data.json())
      .then(console.log);
  }, 2000);
});
```
Виглядає вже не так круто. А це ще відносно проста задача. Часто в реальному житті нам потрібно дочекатися кліка на кнопку, відправити запит на сервер, якщо користувач натисне ще раз - потрібно відмінити попередній запит і почати новий. А ще треба додати обробку помилок, які можуть з'являтися на різних рівнях.
