# JavaScript — ООП и прототипы

> **Теги:** #frontend #javascript #oop #prototype #конспект

> [!abstract] Навигация
> [[main]] | [[main Frontend]] | [[JS_Index]]

---

## 🔹 Прототипная цепочка

В JS каждый объект имеет скрытое свойство `[[Prototype]]` — ссылку на другой объект (прототип). При обращении к свойству JS ищет его вверх по цепочке.

```javascript
const animal = {
  breathe() { return 'дышу' }
}

const dog = Object.create(animal)  // animal — прототип dog
dog.bark = function() { return 'гав' }

dog.bark()     // 'гав' — собственное свойство
dog.breathe()  // 'дышу' — из прототипа
dog.__proto__ === animal  // true

// Цепочка: dog → animal → Object.prototype → null
```

---

## 🔹 Class синтаксис (ES6+)

```javascript
class Animal {
  #name  // приватное поле (ES2022)
  static count = 0  // статическое поле

  constructor(name, sound) {
    this.#name = name
    this.sound = sound
    Animal.count++
  }

  // Геттер / сеттер
  get name() { return this.#name }
  set name(value) {
    if (!value) throw new Error('Имя не может быть пустым')
    this.#name = value
  }

  speak() {
    return `${this.#name} говорит: ${this.sound}`
  }

  static create(name, sound) {  // фабричный метод
    return new Animal(name, sound)
  }
}

class Dog extends Animal {
  #tricks = []

  constructor(name) {
    super(name, 'Гав')  // super — конструктор родителя
  }

  learn(trick) {
    this.#tricks.push(trick)
  }

  speak() {  // переопределение
    return `${super.speak()} (и умеет: ${this.#tricks.join(', ')})`
  }
}

const rex = new Dog('Рекс')
rex.learn('сидеть')
rex.speak()  // 'Рекс говорит: Гав (и умеет: сидеть)'
```

---

## 🔹 Миксины — множественное поведение

JS не поддерживает множественное наследование. Паттерн миксинов:

```javascript
const Serializable = (Base) => class extends Base {
  serialize() { return JSON.stringify(this) }
  static deserialize(json) { return Object.assign(new this(), JSON.parse(json)) }
}

const Validatable = (Base) => class extends Base {
  validate() {
    return Object.keys(this).every(key => this[key] !== null)
  }
}

class User extends Serializable(Validatable(Animal)) {
  constructor(name) {
    super(name, '')
    this.email = null
  }
}
```

---

## 🔹 Полезные паттерны объектов

```javascript
// Иммутабельный объект
const config = Object.freeze({ host: 'localhost', port: 3000 })
config.port = 4000  // silently ignored (strict mode: TypeError)

// Object.assign vs spread
const merged = Object.assign({}, defaults, overrides)  // мутирует первый!
const merged = { ...defaults, ...overrides }            // новый объект

// Динамические ключи
const key = 'name'
const obj = { [key]: 'Дмитрий', [`${key}_upper`]: 'ДМИТРИЙ' }

// Проверки
'key' in obj                    // включая прототип
obj.hasOwnProperty('key')       // только собственные
Object.keys(obj)                // массив собственных ключей
Object.entries(obj)             // [[key, value], ...]
Object.fromEntries(entries)     // обратно в объект
```

---

## 🔗 Смотри также
> - [[JS_Functions]] — this, bind, arrow functions
> - [[JS_Basics]] — типы данных, примитивы vs объекты
