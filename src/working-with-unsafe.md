% Работа с небезопасным кодом

В общем случае инструменты языка Rust достаточно ограничены и сводят 
все ситуации к двум вариантам - безопасному и небезопасному. К сожалению, 
реальность оказывается невообразимо сложнее этого. Например, у нас есть
такая игрушечная функция:

```rust
fn index(idx: usize, arr: &[u8]) -> Option<u8> {
    if idx < arr.len() {
        unsafe {
            Some(*arr.get_unchecked(idx))
        }
    } else {
        None
    }
}
```

Ясно, что эта функция безопасна. Мы проверяем, что индекс находится внутри
границ, и если это так, обращаемся по нему к массиву в небезопасном виде. Но
даже в такой тривиальной функции, область действия небезопасного блока вызывает
вопросы. Поменяем `<` на `<=`:

```rust
fn index(idx: usize, arr: &[u8]) -> Option<u8> {
    if idx <= arr.len() {
        unsafe {
            Some(*arr.get_unchecked(idx))
        }
    } else {
        None
    }
}
```

Программа сломалась, а мы *только поменяли безопасный код*. Это фундаментальная
проблема безопасности: она не локальна. Устойчивость нашей небезопасной операции
обязательно зависит от состояния, полученного из другой, "безопасной" операции.

Безопасность является модульной, в том смысле, что использование небезопасного
кода не приведет к отбрасыванию других произвольных видов проблем. Например,
выполняя непроверенное индексирование в срезе не означает, что вам вдруг надо
беспокоиться о том, что срез станет нулем или будет содержать
неинициализированную память. Ничего фундаментально не меняется. Однако
безопасность *не* модульна в том смысле, что у программы есть состояние, и ваша
небезопасная операция может зависеть от другого произвольного состояния.

Всё ещё хитрее, когда мы имеем дело с настоящим контролем за состоянием. 
Представьте простую реализацию `Vec`:

```rust
use std::ptr;

// Помните, что это определение недостаточное. Смотрите секцию о реализации Vec.
pub struct Vec<T> {
    ptr: *mut T,
    len: usize,
    cap: usize,
}

// Обратите внимание, что эта реализация некорректно обрабатывает типы нулевого размера
// Здесь мы живем в прекрасном выдуманном мире положительных типов фиксированного размера.
impl<T> Vec<T> {
    pub fn push(&mut self, elem: T) {
        if self.len == self.cap {
            // неважно для этого примера
            self.reallocate();
        }
        unsafe {
            ptr::write(self.ptr.offset(self.len as isize), elem);
            self.len += 1;
        }
    }

    # fn reallocate(&mut self) { }
}

# fn main() {}
```

Этот код достаточно прост для разумного аудита и проверки. Добавим следующий
метод:

```rust,ignore
fn make_room(&mut self) {
    // наращиваем размерность
    self.cap += 1;
}
```

Это 100% Безопасный Rust, но он абсолютно неустойчив. Изменение размера нарушает
инварианты Vec (то, как `cap` отражает распределение памяти в Vec). И нет ничего
в остальном Vec, что могло бы защитить от этого. *Придется* доверять полю
размера, потому что проверить его никак нельзя.

`unsafe` не просто загрязняет всю функцию: он загрязняет весь *модуль*. 
В общем случае, ограничить небезопасность кода можно на границах модуля, с помощью 
контроля за видимостью (приватностью/публичностью) членов модуля. Указание 
публичности членов модуля (через ключевое слово pub) — это инструмент ограничения
небезопасности, который действует только на границе модуля (это особенность языка).

Однако это работает *идеально*. Существование `make_room` - это *не* проблема
устойчивости Vec, потому что мы не пометили его как публичный. Эту функцию можно
вызвать только внутри модуля, в котором она определена. Также `make_room`
напрямую получает доступ к приватным полям Vec, поэтому она может быть написана
только в том же модуле, что и Vec.

Таким образом мы можем написать полностью безопасную абстракцию, которая
опирается на сложные инварианты. Это *очень важно* для связи между
Безопасным Rust и Небезопасным. Мы уже видели, что Небезопасный код должен
доверять *некоторому* Безопасному, но он не может доверять *произвольному* Безопасному
коду. Он не может доверять тому, что произвольная реализация типажа или любой
функции, которые передаются ему, будут правильно себя вести в тех случаях, о
которых не заботится безопасный код.

Однако, если небезопасный код не сможет предотвратить то, что безопасный
клиентский код будет портить его состояние в произвольном случае, безопасность
будет потеряна. Спасибо, он хотя бы *может* предотвратить то, что произвольный код
будет портить его критическое состояние, благодаря приватности.

Живи безопасно!

