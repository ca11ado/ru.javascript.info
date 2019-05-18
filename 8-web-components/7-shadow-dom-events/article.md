# Теневой DOM ("Shadow DOM") и события

Основная идея, стоящая за теневым деревом -- скрыть детали внутренней реализации компонента.

Скажем, внутри теневого DOM компонента `user-card` произошло событие клика. Но скрипты основного документа не имеют представления о том, что происходит внутри теневого DOM, особенно если компонент получен из сторонних библиотек или что-то типа того.

Поэтому, чтобы сохранять вещим простыми, браузер *перенаправляет* событие.

**Для событий, которые произошли в теневом DOM, целевым элементом выступает элемент-хозяин, когда их перехватывают снаружи компонента.**

Вот простой пример:

```html run autorun="no-epub" untrusted height=60
<user-card></user-card>

<script>
customElements.define('user-card', class extends HTMLElement {
  connectedCallback() {
    this.attachShadow({mode: 'open'});
    this.shadowRoot.innerHTML = `<p>
      <button>Click me</button>
    </p>`;
    this.shadowRoot.firstElementChild.onclick =
      e => alert("Внутренний целевой элемент: " + e.target.tagName);
  }
});

document.onclick =
  e => alert("Внешний целевой элемент: " + e.target.tagName);
</script>
```

Если вы кликните на кнопку, вы увидите следующие сообщения:

1. Внутренний целевой элемент: `BUTTON` -- внутренний обработчик события получит корректный целевой элемент -- элемент внутри теневого DOM.
2. Внешний целевой элемент: `USER-CARD` -- обработчик события в документе в качестве целевого элемента получит "хозяина" теневого DOM.

Очень хорошо, что у нас есть перенаправление событий, потому что внешний документ не должен знать о внутренних деталях компонента. С его точки зрения событие произошло на `<user-card>`.

**Перенаправление не происходит, если событие случилось на слотовом элементе, который физически живёт в легком DOM.**

Например, если в примере ниже пользователь кликнет на `<span slot="username">`, именно он станет целевым для события и в теневом и в легком обработчиках:

```html run autorun="no-epub" untrusted height=60
<user-card id="userCard">
*!*
  <span slot="username">John Smith</span>
*/!*
</user-card>

<script>
customElements.define('user-card', class extends HTMLElement {
  connectedCallback() {
    this.attachShadow({mode: 'open'});
    this.shadowRoot.innerHTML = `<div>
      <b>Имя:</b> <slot name="username"></slot>
    </div>`;

    this.shadowRoot.firstElementChild.onclick =
      e => alert("Внутренняя цель: " + e.target.tagName);
  }
});

userCard.onclick = e => alert(`Внешняя цель: ${e.target.tagName}`);
</script>
```

Если клик произойдёт на `"Johm Smith"`, и для внутреннего и для внешнего обработчика целевым будет `<span slot="username">`. Это элемент легкого DOM, так что перенаправления не будет.

С другой стороны, если клик произойдёт на элементе теневого DOM, например, на `<b>Имя</b>`, тогда, так как он всплывёт наружу из теневого DOM, его `event.target` будет изменён на  `<user-card>`.

## Всплытие, event.composedPath()

Для реализации всплытия используется уплощенный DOM.

Так что, если у нас есть слот-элемент, и событие произошло где-то внутри него, оно всплывёт до `<slot>` и вверх.

Полный путь к оригинальному целевому элементу со всеми корневыми теневыми элементами может быть получен, используя `event.composedPath()`. Как мы видим из имени метода, этот путь получается после композиции.

В примере выше, уплощённый DOM -- это:

```html
<user-card id="userCard">
  #shadow-root
    <div>
      <b>Имя:</b>
      <slot name="username">
        <span slot="username">John Smith</span>
      </slot>
    </div>
</user-card>
```


Так что для клика на `<span slot="username">` вызов `event.composePath()` вернёт массив: [`span`, `slot`, `div`, `shadow-root`, `user-card`, `body`, `html`, `document`, `window`]. Это точная цепочка родителей для целевого элемента в уплощённом DOM после композиции.

```warn header="Shadow tree details are only provided for `{mode:'open'}` trees"
Если теневое дерево было создано с `{mode: 'closed'}`, то составной путь начнётся с элемента-хозяина `user-card` и выше.

Это аналогичный принцип, как и в других методах работы с теневым DOM. Содержимое закрытых деревьев полностью скрыто.
```


## event.composed

Большинство событий успешно всплывают через границы теневого DOM. Но есть несколько исключений.

Это управляется свойством `composed` объекта события. Если его значение `true`, тогда событие пересекает границы. В противном случае, оно может быть поймано только внутри теневого DOM.

Если вы взгляните на [Спецификацию событий пользовательского интерфейса](https://www.w3.org/TR/uievents), у большинства событий установлено `composed: true`:

- `blur`, `focus`, `focusin`, `focusout`,
- `click`, `dblclick`,
- `mousedown`, `mouseup` `mousemove`, `mouseout`, `mouseover`,
- `wheel`,
- `beforeinput`, `input`, `keydown`, `keyup`.

У всех "touch" и "pointer" событий тоже установлено `composed: true`.

Хотя есть некоторые события, у которых установлено `composed: false`:

- `mouseenter`, `mouseleave` (они к тому же не всплывают),
- `load`, `unload`, `abort`, `error`,
- `select`,
- `slotchange`.

Эти события могут быть пойманы только в том же DOM, в котором находится целевой элемент.

## Пользовательские события

Когда мы отправляем пользовательские события, нам нужно установить оба свойства `bubbles` и `composed` в `true`, чтобы они могли всплывать наверх и наружу компонента.

Например, ниже мы создаём `div#inner` в теневом DOM `div#outer` и вызываем два события на нём. И только то, у которого установлено `composed: true`, всплывёт наружу, к документу.

```html run untrusted height=0
<div id="outer"></div>

<script>
outer.attachShadow({mode: 'open'});

let inner = document.createElement('div');
outer.shadowRoot.append(inner);

/*
div(id=outer)
  #shadow-dom
    div(id=inner)
*/

document.addEventListener('test', event => alert(event.detail));

inner.dispatchEvent(new CustomEvent('test', {
  bubbles: true,
*!*
  composed: true,
*/!*
  detail: "composed"
}));

inner.dispatchEvent(new CustomEvent('test', {
  bubbles: true,
*!*
  composed: false,
*/!*
  detail: "not composed"
}));
</script>
```

## Итого

События пересекают границы теневого DOM только тогда, когда их флаг `composed` установлен в `true`.

У большинства встроенных событий установлено `composed: true`, как описано в соответствующих спцификациях:

- Событие пользовательского интерфейса <https://www.w3.org/TR/uievents>.
- Тач события <https://w3c.github.io/touch-events>.
- Поинтер события <https://www.w3.org/TR/pointerevents>.
- ...и так далее.

Некоторые встроенные события, у которых установлено `composed: false`:

- `mouseenter`, `mouseleave` (к тому же не всплывают),
- `load`, `unload`, `abort`, `error`,
- `select`,
- `slotchange`.

Эти события могут быть пойманы только на элементах того же самого DOM.

Если мы отправляем `CustomEvent`, нужно явно установить `composed: true`.

Обратите внимание, что в случае наследуемых компонентов составные события всплывают сквозь все границы теневого DOM. Так что, если событие предназначено только для непосредственно вложенного компонента, мы так же можем отправить его на "хозяине" теневого DOM. Затем оно уже будет вне теневого DOM.

