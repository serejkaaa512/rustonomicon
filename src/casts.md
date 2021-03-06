% Явное приведение типа

Явное приведение - это надстройка над неявным приведением: каждое неявное
приведение можно вызвать посредством явного приведения. Некоторые
преобразования типов требуют явного приведения. В то время как неявные
приведения распространены и в основном безвредны, эти "настоящие явные
приведения типов" редки и потенциально опасны. Поэтому явные приведения должны
явно вызываться с помощью ключевого слова `as`: `expr as Type`.

Настоящие явные приведения обычно крутятся вокруг сырых указателей и примитивных
числовых типов. Даже при том, что они опасны, эти приведения не могут сломаться
во время выполнения. Если явное приведение вызовет какую-то проблему, это
невозможно будет обнаружить. Явное приведение просто выполнится. Несмотря на
это, явные приведения должны быть правильны на уровне типов, или они будут
предотвращены статически. Например, `7u8 as bool` не компилируется.

При этом явные приведения не `unsafe`, потому что они вообще не могут нарушить
безопасность памяти *сами по себе*. Например, преобразование целого в сырой
указатель может легко привести к ужасным вещам. Но само действие по созданию
указателя безопасно, потому что на самом деле использование сырого указателя уже
помечено `unsafe`.

Вот полный список всех правильных явных приведений. Для краткости используем
`*`, чтобы обозначить `*const` или  `*mut`, и `integer` - для любого целого
примитива:

 * `*T as *U` где `T, U: Sized`
 * `*T as *U` TODO: объяснить ситуацию с безразмерными типами
 * `*T as integer`
 * `integer as *T`
 * `number as number`
 * `C-like-enum as integer`
 * `bool as integer`
 * `char as integer`
 * `u8 as char`
 * `&[T; n] as *const T`
 * `fn as *T` где `T: Sized`
 * `fn as integer`

Заметьте, что длины не корректируются при явном приведении сырых срезов -
`*const [u16] as *const [u8]` создаст срез, который состоит из половины
оригинальной памяти.

Явное приведение не транзитивно, это означает, что, даже если `e as U1 as U2`
правильное выражение, `e as U2` не обязано быть таким же.

В случае с числами нужно пояснить несколько моментов:

* явное приведение между двумя целыми одного размера (e.g. i32 -> u32) это 
пустая операция
* явное приведение большего целого к меньшему (e.g. u32 -> u8) обрежет большее
* явное приведение меньшего целого к большего (e.g. u8 -> u32) будет
    * дополнять нулями если источник беззнаковый
    * дополнять знаком если источник знаковый
* явное приведение дробного к целому осуществляется округлением дробного к нулю
    * **[ВНИМАНИЕ: на данный момент может вызвать Неопределенное Поведение, если
     округленное значение не может быть представлено целевым целым типом]
     [float-int]**.  Включает в себя Inf и NaN. Это ошибка и она будет исправлена.
* явное приведение целого к дробному осуществляется созданием дробного числа, 
округленного при необходимости (стратегия округления не указана)
* явное приведение f32 к f64 выполняется отлично и без потерь
* явное приведение f64 к f32 создаст ближайшее возможное значение
  (стратегия округления не указана)
    * **[ВНИМАНИЕ: на данный момент может вызвать Неопределенное Поведение, если
      значение конечно, но больше или меньше самого маленького или самого 
      большого конечного числа, представляемым f32][float-float]**. 
      Это ошибка и она будет исправлена.


[float-int]: https://github.com/rust-lang/rust/issues/10184
[float-float]: https://github.com/rust-lang/rust/issues/15536
