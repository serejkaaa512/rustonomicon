% Проверяемая неинициализированная память

Как и в C, все переменные на стеке в Rust не инициализированы до тех пор пока им
явно не присвоено значение. В отличие от C, Rust статически ограничивает их
чтение, пока вы не сделаете это:

```rust,ignore
fn main() {
    let x: i32;
    println!("{}", x);
}
```

```text
src/main.rs:3:20: 3:21 error: use of possibly uninitialized variable: `x`
src/main.rs:3     println!("{}", x);
                                 ^
```

Все основывается на базовом анализе веток: каждая ветка должна присвоить `x`
значение до его первого использования. Интересно, что Rust не требует, чтобы
переменная была изменяемой, чтобы выполнить отложенную инициализацию, если
каждая ветка присваивает значение лишь однажды. Однако такой анализ не
использует анализ констант или что-либо подобное. Поэтому это компилируется:

```rust
fn main() {
    let x: i32;

    if true {
        x = 1;
    } else {
        x = 2;
    }

    println!("{}", x);
}
```

а это нет:

```rust,ignore
fn main() {
    let x: i32;
    if true {
        x = 1;
    }
    println!("{}", x);
}
```

```text
src/main.rs:6:17: 6:18 error: use of possibly uninitialized variable: `x`
src/main.rs:6   println!("{}", x);
```

хотя это тоже компилируется:

```rust
fn main() {
    let x: i32;
    if true {
        x = 1;
        println!("{}", x);
    }
    // Не обращайте внимания на то, что есть еще ветки, в которых x не инициализирована
    // ведь мы не используем в этих ветках ее значение 
}
```

Конечно, из-за того, что в анализе не участвуют настоящие значения, анализу
приходится обладать сравнительно сложным пониманием зависимостей и потока
выполнения. Например, это работает:

```rust
let x: i32;

loop {
    // Rust не понимает, что эта ветка безоговорочно выполнится,
    // потому что это зависит от настоящих значений.
    if true {
        // Но он понимает, что попадет сюда лишь один раз, потому что 
        // мы однозначно выходим отсюда. Поэтому `x` не надо помечать
        // изменяемым.
        x = 0;
        break;
    }
}
// Он также понимает, что невозможно добраться сюда, не достигнув break.
// И, следовательно, `x` должна быть уже инициализирована здесь!
println!("{}", x);
```

Если переменная перестает владеть значением, эта переменная становится логически
неинициализированной, если только тип значения не Copy. Это означает:

```rust
fn main() {
    let x = 0;
    let y = Box::new(0);
    let z1 = x; // x остается в силе из-за того, что i32 это Copy
    let z2 = y; // y теперь логически не инициализирована, потому что Box не Copy
}
```

Но переприсваивание `y` в этом примере *потребует*, чтобы `y` была помечена
изменяемой, дабы программа на Безопасном Rust могла заметить, что значение `y`
поменялось:

```rust
fn main() {
    let mut y = Box::new(0);
    let z = y; // y теперь логически не инициализирована, потому что Box не Copy
    y = Box::new(1); // переинициализация y
}
```

Иначе `y` станет абсолютно новой переменной.