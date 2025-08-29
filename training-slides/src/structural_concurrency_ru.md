# Структурная конкурентность

Незабываемые типы

## Дисклеймер

Этот доклад требует продвинутого понимания языка Rust и в первую очередь сделан для любителей языка.

## Работать в асинхронном Rust тяжело.

Notes:

В контрасте с многопоточным кодом возникают проблемы с заимствованием (borrowing) и тред-безопасыми (thread-safe) типами.

## Асинхронные задачи (async tasks)

```rust [1-7|3-5|7]
use tokio::task;
let s0 = "Hello, world!".to_string();
let t0 = task::spawn(async {
    do_work(s0).await
});
do_other_work().await;
let r0 = t0.await;
```

Notes:

Пример использования асинхронных задач (async tasks) из tokio.

## Заимствование в задачах

```rust [1-7|2,4|3]
let s0 = "Hello, world!".to_string();
let s1 = &s0[..];
let t0 = task::spawn(async { // ERROR: `s1` must be `'static`
    do_work(s1).await
});
do_other_work().await;
let r0 = t0.await;
```

## Задачи могут быть только `'static`

```rust []
pub fn spawn<F>(future: F) -> JoinHandle<F::Output>
where
    F: Future + Send + 'static,
    F::Output: Send + 'static,
{ /* ... */ }
```

Notes:

Исполнение задачи, может занять сколько угодно времени, даже если какие-то заимствующие ссылки (borrows) станут недействительными.

## Решение: ждать завершение задачи

```rust []
pub fn spawn_scoped<'a, F>(future: F) -> ScopedJoinHandle<'a, F::Output>
where
    F: Future + Send + 'a,
    F::Output: Send + 'a,
{ /* ... */ }

impl<T> Drop for ScopedJoinHandle<'_, T> {
    fn drop(&mut self) {
        block_on(self);
    }
}
```

## Rust не гарантирует вызова drop

```rust [1-8|1,7|3,5,8]
use std::mem::forget;
let mut s0 = "Hello, world!".to_string();
let s1 = &s0[..];
let t0 = task::spawn_scoped(async {
    do_work(s1).await;
});
forget(t0);
s0.clear(); // `t0` could use now invalid `s1`
```

Notes:

Существует такая функция `std::mem::forget`, которая просто убивает переменную, не вызывая на ней деструктор.

Можно просто забыть о ранее запущенных задач, даже если они заимствуют локальные данные на стэке, которые впоследствии станут недействительными.

## Заимствующие потоки (threads)

```rust []
use std::thread;
let mut s0 = "Hello, world!".to_string();
let s1 = &s0[..];
let t0 = thread::scoped(|| {
    do_work(s1);
});
forget(t0);
s0.clear(); // `t0` could use now invalid `s1`
```

## `std::mem::forget`

```rust []
struct Leak<T>(mpsc::Receiver<Leak<T>>, T);

pub fn forget<T>(data: T) {
    let (tx, rx) = mpsc::channel();
    tx.send(Leak(rx, data)).unwrap(); // Create a cycle
}
```

Notes:

На самом деле сегодня Rust не может гарантировать, чтобы вызов drop вообще произошёл.
Такую функцию можно реализовать с помощью mpsc каналов.

Об этом факте узнали незадолго до релиза Rust 1.0.
Раньше такая гарантия присутствовала в языке, но потом от неё решили отказаться из-за временных рамок.

## `std::thread::scope`

```rust [1-9|1-3|4-6|7|8]
let s0 = String::from("Hello, world!");
let s1 = &s0[..];
std::thread::scope(|scope| {
    let t0 = scope.spawn(|| {
        do_work(s1);
    });
    do_other_work();
    let r0 = t0.join().unwrap();
})
```

Notes:

В случае с потоками эту проблему обошли с помощью `std::thread::scope`.
Данный интерфейс работает с помощью подпрограмм (subroutines), то есть обычных функций, а точнее с помощью гарантированного порядка исполнения вложенных подпрограмм.

## `tokio::task::scope`?

```rust [1-9|9]
let s0 = String::from("Hello, world!");
let s1 = &s0[..];
task::scope(async |scope| {
    let t0 = scope.spawn(async {
        do_work(s1).await;
    });
    do_other_work().await;
    let r0 = t0.await;
}).await
```

## Контрпример

```rust [1-15|4-10|3,11|12|12,5-7|12-13,8|14|15]
let mut s0 = String::from("Hello, world!");
let s1 = &s0[..];
let mut fut = Box::pin(async {
    task::scope(async |scope| {
        let t0 = scope.spawn(async {
            do_work(s1).await;
        });
        task::yield_now().await;
        let r0 = t0.await;
    }.await
}));
assert_eq!(poll_once(fut.as_mut()), Poll::Pending);
// `fut` progress is now on line 8
forget(fut);
s0.clear(); // `t0` could use now invalid `s1`
```

Notes:

В контрасте, асинхронный Rust реализован с помощью сопрограмм (coroutines), вследствие чего вышеупомянутый подход просто так не работает.

Тут мы используем гипотетический интерфейс внутри футуры, которую мы выполним только наполовину, а затем забудем.
Для этого мы воспользуемся `yield_now`.

## `Forget` маркер

```rust []
unsafe auto trait Forget {}

impl<T: Forget> Arc<T> {
    pub fn new(data: T) -> Self {
        // ...
    }
}
```

Notes:

Одно из напрашивающихся решений - добавить новый маркер-трейт по аналогии с `Send`, который даст использовать `forget`, `Rc` и другие интерфейсы по отношению, не ко всем, а только к "забываемым" типам.

## Вложенные незабываемые типы

```rust []
pub struct JoinGuard<'a, T> {
    // ...
}
// JoinGuard<'a, T>: !Forget => Foo<'a, T>: !Forget
struct Foo<'a, T> {
    inner: JoinGuard<'a, T>,
    other_data: i32,
}
```

Notes:

Нам важно чтобы тип `JoinGuard` не смог попасть в `forget` или `Rc`, даже если он вложен внутри другой структуры.
Если какое-то поле структуры (`inner`) имеет незабываемый тип, то и тип самой структуры (`Foo`) считается незабываемым.

## `Forget` не предотвращает утечки памяти

```rust []
fn forget<T: 'static>(data: T) {
    std::thread::spawn(move || {
        let _data = data;
        loop {} // drop(_data) is never called
    });
}
```

Notes:

Даже если мы определим тип как незабываемый (`!Forget`), это не даёт гарантию, что drop объекта когда-либо произойдёт.
Это не позволяет нам так просто избавиться от утечек памяти, как до этого люди долго представляли себе решение этой проблемы.
Поэтому в ранних упоминаниях `Forget` именуется `Leak`.

## ¯\\_ (ツ)_/¯

Notes:

Но раз утечки памяти - безопасные, то какая разница?
Главное, что `!Forget` заставляет компилятор всегда ассоциировать конец жизни объекта с вызовом drop.
В случае с `JoinGuard<’a, T>` это действительно предотвращает инвалидацию заимствованных ссылок в купе с borrow checker.

## `T: 'static` означает `T: Forget`?

```rust []
fn forget<T: 'static>(data: T) {
    std::thread::spawn(move || {
        let _data = data;
        loop {} // drop(_data) is never called
    });
}
```

Notes:

В таком случае, ничто не мешает нам утверждать, что `T: ’static` типы автоматически предполагают и `T: Forget`.

## Рандеву каналы

```rust [|1-2|5,9]
// let (tx, rx) = std::sync::mpsc::sync_channel(0);
let (tx, rx) = rendezvois_channel();
let t0 = std::thread::spawn(move || {
    do_work();
    let data = rx.recv().unwrap();
    process_data(data);
});
let data = fetch_data();
tx.send(data).unwrap();
```

Notes:

Для начала мне нужно уточнить что такое рандеву канал.
Рандеву канал является частным случаем mpsc каналов с очередью нулевого размера.

## Циклы владения тредами

```rust [|4,5,8]
let mut s0 = "Hello, world!".to_string();
let s1 = &s0[..];
let (tx, rx) = rendezvois_channel();
let t0 = std::thread::scoped(move || {
    let t0 = rx.recv().unwrap();
    do_work(s1);
});
tx.send(t0).unwrap();
s0.clear(); // `t0` could use now invalid `s1`
```

Notes:

Но можно поместить заимствующий тред сам в себя с помощью рандеву канала, хотя казалось бы обычными рекурсивными структурами такой цикл невозможно создать.

## `std::thread::scoped<F: Forget>(f: F)`

```rust []
let mut s0 = "Hello, world!".to_string();
let s1 = &s0[..];
let (tx, rx) = rendezvois_channel();
let t0 = std::thread::scoped(move || {
    let t0 = rx.recv().unwrap();
    do_work(s1);
});
tx.send(t0).unwrap(); // ERROR: `t0` must be `Forget`
s0.clear();
```

## `task::spawn_scoped<F: Forget>(f: F)`

```rust [|1,5]
let t0 = task::spawn_scoped(async { // ERROR: `t1` must be `Forget`
    let t1 = task::spawn_scoped(async {
        do_work().await;
    });
    do_other_work().await; // because `t1` must stay alive across this await point
    t1.await;
});
```

## `JoinGuard<'a, T>: !Send`

```rust []
let mut s0 = "Hello, world!".to_string();
let s1 = &s0[..];
let (tx, rx) = rendezvois_channel();
let t0 = std::thread::scoped(move || { // ERROR: `rx` must be `Send`
    let t0 = rx.recv().unwrap();
    do_work(s1);
});
tx.send(t0).unwrap();
s0.clear();
```

## `ScopedTask<'a, T>: !Send`

```rust [|1,5]
let t0 = task::spawn_scoped(async { // ERROR: `t1` must be `Send`
    let t1 = task::spawn_scoped(async {
        do_work().await;
    });
    do_other_work().await; // because `t1` must stay alive across this await point
    t1.await;
});
```

Notes:

`spawn_scoped` может запустить задачу на другом треде, в отличии от `spawn_local_scoped`.

## `rendezvois_channel<T: Forget>()`

```rust []
let mut s0 = "Hello, world!".to_string();
let s1 = &s0[..];
let (tx, rx) = rendezvois_channel();
let t0 = std::thread::scoped(move || {
    let t0 = rx.recv().unwrap();
    do_work(s1);
});
tx.send(t0).unwrap(); // ERROR: `t0` must be `Forget`
s0.clear();
```

Notes:

Несмотря на видимое исправление, это решение основывается на множестве предположений, не являющиеся гарантиями языка как таковыми, однако для которых пока никто не смог придумать контрпример.
Грубо говоря неясна фундаментальная разница между предачей данных в тред до или после создания `JoinGuard` объекта.
Раз на уровне реализации такой разницы может и не существовать, то мы говорим об особенности системы типов.

Так или иначе на данный момент это наиболее гибкое решение для внедрения в Rust, поэтому я больше всего склоняюсь к нему.

## Спасибо за внимание

Вопросы?

## Примечания

- Исторический пост о проблеме: [Leakpocalypse](https://cglab.ca/~abeinges/blah/everyone-poops/#leakpocalypse)
- Что такое структурная конкурентность: <br> [Structured concurrency](https://blog.yoshuawuyts.com/tree-structured-concurrency/)
- Мой пост о Forget: [Myosotis](https://zetanumbers.github.io/book/myosotis.html)
- RFC с Forget маркером: [rust-lang/rfcs#3782](https://github.com/rust-lang/rfcs/pull/3782)
