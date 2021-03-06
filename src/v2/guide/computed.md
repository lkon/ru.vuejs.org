---
title: Вычисляемые свойства и слежение
type: guide
order: 5
---

## Вычисляемые свойства

Выражения, встраиваемые в шаблоны, удобны, но предназначены они только для простых операций. При усложнении логики, они быстро становятся трудноподдерживаемыми. Например:

``` html
<div id="example">
  {{ message.split('').reverse().join('') }}
</div>
```

Этот шаблон уже не выглядит просто и декларативно. С первого взгляда и не скажешь, что он всего лишь отображает `message` задом наперёд. Ситуация станет ещё хуже, если эту логику в шаблоне потребуется использовать в нескольких местах.

На помощь приходят **вычисляемые свойства**.

### Простой пример

``` html
<div id="example">
  <p>Изначальное сообщение: "{{ message }}"</p>
  <p>Сообщение задом наперёд: "{{ reversedMessage }}"</p>
</div>
```

``` js
var vm = new Vue({
  el: '#example',
  data: {
    message: 'Привет'
  },
  computed: {
    // геттер вычисляемого значения
    reversedMessage: function () {
      // `this` указывает на экземпляр vm
      return this.message.split('').reverse().join('')
    }
  }
})
```

Результат:

{% raw %}
<div id="example" class="demo">
  <p>Изначальное сообщение: "{{ message }}"</p>
  <p>Сообщение задом наперёд: "{{ reversedMessage }}"</p>
</div>
<script>
var vm = new Vue({
  el: '#example',
  data: {
    message: 'Привет'
  },
  computed: {
    reversedMessage: function () {
      return this.message.split('').reverse().join('')
    }
  }
})
</script>
{% endraw %}

Здесь мы определили вычисляемое свойство `reversedMessage`. Написанная нами функция будет использоваться как геттер свойства `vm.reversedMessage`:

``` js
console.log(vm.reversedMessage) // => 'тевирП'
vm.message = 'Пока'
console.log(vm.reversedMessage) // => 'акоП'
```

Вы можете открыть консоль и поиграть с примером сами. Значение `vm.reversedMessage` всегда зависит от значения `vm.message`.

В шаблонах вы можете привязываться к вычисляемым свойствам ровно таким же образом, как и к обычным. Vue знает, что `vm.reversedMessage` зависит от `vm.message`, так что при обновлении `vm.message` обновятся и все элементы, зависящие от `vm.reversedMessage`. И самое главное, что эту зависимость мы указали декларативно: геттер вычисляемого свойства не имеет побочных эффектов, что упрощает как понимание кода, так и тестирование.

### Кеширование вычисляемых свойств

Вы могли заметить, что того же результата можно достигнуть при помощи метода:

``` html
<p>Сообщение задом наперёд: "{{ reverseMessage() }}"</p>
```

``` js
// в компоненте
methods: {
  reverseMessage: function () {
    return this.message.split('').reverse().join('')
  }
}
```

Вместо вычисляемого свойства, мы можем указать ту же самую функцию в качестве метода. С точки зрения конечного результата, оба подхода действительно делают одно и то же. Но есть важное различие: **вычисляемые свойства кешируются, основываясь на своих зависимостях**. Вычисляемое свойство будет пересчитано только тогда, когда изменится одна из его зависимостей. Поэтому, пока `message` остаётся неизменным, многократное обращение к `reversedMessage` будет каждый раз незамедлительно возвращать единожды вычисленное значение, не запуская функцию вновь.

Кроме того, это значит, что следующее вычисляемое свойство никогда не обновится, так как `Date.now()` не является реактивной зависимостью:

``` js
computed: {
  now: function () {
    return Date.now()
  }
}
```

Использование метода, напротив, запускает функцию **всегда, при каждом обращении к нему**.

Зачем нужно кеширование? Представьте, что у вас есть дорогое вычисляемое свойство **A**, требующее цикла по огромному массиву и выполняющее множество вычислений. А ещё пусть будут другие вычисляемые свойства, в свою очередь, зависящие от **A**. Без кеширования геттер **A** будет запускаться куда чаще необходимого! В тех же случаях, когда кеширования нужно избежать — используйте методы.

### Вычисляемые свойства и слежение

Vue также предоставляет и более общий способ наблюдения и реагирования на изменения данных в экземпляре: **слежение за свойствами**. Когда у вас есть данные, которые должны быть обновлены при изменении других данных, возникает соблазн избыточно использовать этот подход, особенно если вы привыкли к Angular. Но, как правило, гораздо лучше использовать вычисляемые свойства, а не императивный коллбэк в `watch`. Рассмотрим пример:

``` html
<div id="demo">{{ fullName }}</div>
```

``` js
var vm = new Vue({
  el: '#demo',
  data: {
    firstName: 'Foo',
    lastName: 'Bar',
    fullName: 'Foo Bar'
  },
  watch: {
    firstName: function (val) {
      this.fullName = val + ' ' + this.lastName
    },
    lastName: function (val) {
      this.fullName = this.firstName + ' ' + val
    }
  }
})
```

Код выше — императивный и избыточный. Сравните с версией с использованием вычисляемого свойства:

``` js
var vm = new Vue({
  el: '#demo',
  data: {
    firstName: 'Foo',
    lastName: 'Bar'
  },
  computed: {
    fullName: function () {
      return this.firstName + ' ' + this.lastName
    }
  }
})
```

Так же гораздо лучше, не правда ли?

### Сеттеры вычисляемых свойств

По умолчанию вычисляемые свойства работают только на чтение, но в случае необходимости вы можете также указать и сеттер:

``` js
// ...
computed: {
  fullName: {
    // геттер:
    get: function () {
      return this.firstName + ' ' + this.lastName
    },
    // сеттер:
    set: function (newValue) {
      var names = newValue.split(' ')
      this.firstName = names[0]
      this.lastName = names[names.length - 1]
    }
  }
}
// ...
```

Теперь запись `vm.fullName = 'Иван Иванов'` вызовет сеттер, и `vm.firstName` и `vm.lastName` будут соответствующим образом обновлены.

## Методы-наблюдатели

Хотя в большинстве случаев лучше использовать вычисляемые свойства, иногда пользовательские методы-наблюдатели всё же необходимы. Поэтому Vue предоставляет более общий путь для реагирования на изменения в данных через опцию `watch`. Полезнее всего эта возможность оказывается для дорогих или асинхронных операций, выполняемых в ответ на изменение данных.

Рассмотрим пример:

``` html
<div id="watch-example">
  <p>
    Задайте вопрос, на который можно ответить "да" или "нет":
    <input v-model="question">
  </p>
  <p>{{ answer }}</p>
</div>
```

``` html
<!-- Поскольку уже существует обширная экосистема ajax-библиотек  -->
<!-- и библиотек функций общего назначения, ядро Vue может        -->
<!-- оставаться маленьким и не изобретать их заново. Кроме того,  -->
<!-- это позволяет вам использовать только знакомые инструменты. -->
<script src="https://cdn.jsdelivr.net/npm/axios@0.12.0/dist/axios.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/lodash@4.13.1/lodash.min.js"></script>
<script>
var watchExampleVM = new Vue({
  el: '#watch-example',
  data: {
    question: '',
    answer: 'Пока вы не зададите вопрос, я не могу ответить!'
  },
  watch: {
    // эта функция запускается при любом изменении вопроса
    question: function (newQuestion, oldQuestion) {
      this.answer = 'Ожидаю, когда вы закончите печатать...'
      this.debouncedGetAnswer()
    }
  },
  created: function () {
    // _.debounce — это функция из lodash, позволяющая ограничить
    // то, насколько часто может выполняться определённая операция.
    // В данном случае мы хотим ограничить частоту обращений к yesno.wtf/api,
    // дожидаясь завершения печати вопроса перед тем как послать ajax-запрос.
    // Чтобы узнать больше о функции _.debounce (и её родственнице _.throttle),
    // см. документацию: https://lodash.com/docs#debounce
    this.debouncedGetAnswer = _.debounce(this.getAnswer, 500)
  },
  methods: {
    getAnswer: function () {
      if (this.question.indexOf('?') === -1) {
        this.answer = 'Вопросы обычно заканчиваются вопросительным знаком. ;-)'
        return
      }
      this.answer = 'Думаю...'
      var vm = this
      axios.get('https://yesno.wtf/api')
        .then(function (response) {
          vm.answer = _.capitalize(response.data.answer)
        })
        .catch(function (error) {
          vm.answer = 'Ошибка! Не могу связаться с API. ' + error
        })
    }
  }
})
</script>
```

Результат:

{% raw %}
<div id="watch-example" class="demo">
  <p>
    Задайте вопрос, на который можно ответить "да" или "нет":
    <input v-model="question">
  </p>
  <p>{{ answer }}</p>
</div>
<script src="https://cdn.jsdelivr.net/npm/axios@0.12.0/dist/axios.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/lodash@4.13.1/lodash.min.js"></script>
<script>
var watchExampleVM = new Vue({
  el: '#watch-example',
  data: {
    question: '',
    answer: 'Пока вы не зададите вопрос, я не могу ответить!'
  },
  watch: {
    question: function (newQuestion, oldQuestion) {
      this.answer = 'Ожидаю, когда вы закончите печатать...'
      this.debouncedGetAnswer()
    }
  },
  created: function () {
    this.debouncedGetAnswer = _.debounce(this.getAnswer, 500)
  },
  methods: {
    getAnswer: function () {
      if (this.question.indexOf('?') === -1) {
        this.answer = 'Вопросы обычно заканчиваются вопросительным знаком. ;-)'
        return
      }
      this.answer = 'Думаю...'
      var vm = this
      axios.get('https://yesno.wtf/api')
        .then(function (response) {
          vm.answer = _.capitalize(response.data.answer)
        })
        .catch(function (error) {
          vm.answer = 'Ошибка! Не могу связаться с API. ' + error
        })
    }
  }
})
</script>
{% endraw %}

В данном случае применение опции `watch` позволяет нам выполнять асинхронную операцию (обращение к API), ограничивать частоту выполнения этой операции и устанавливать промежуточные состояния до получения окончательного ответа. Ничего из этого не удалось бы достичь с помощью вычисляемых свойств.

В дополнение к опции `watch` вы можете также использовать [vm.$watch](../api/#vm-watch) в императивном стиле.
