## Основные понятия:

#### 1. Класс-модель

```typescript
class User {
  // Описание структуры данных
}
```

- Что это: TypeScript класс, описывающий структуру документа в MongoDB
- Зачем: Единое место описания всех полей и их типов
#### 2. Декоратор @prop()

```typescript
@prop({ type: String, required: true })
public name!: string;
```

- Что это: Декоратор, который превращает поле класса в поле MongoDB
- Зачем: Настройка валидации, типов, индексов для MongoDB

#### 3. Декоратор @modelOptions()

```typescript
@modelOptions({
  schemaOptions: {
    collection: "users",
    timestamps: true
  }
})
class User { ... }
```

- Что это: Настройки для всей модели
- Зачем: Имя коллекции, временные метки, индексы

#### 4. Функция getModelForClass()

``` typescript
const UserModel = getModelForClass(User);
```

- Что это: Превращает класс в Mongoose модель
- Зачем: Получить объект для работы с MongoDB (save, find, etc.)

## Опции @prop() декоратора:

| Опция                   | Тип     | Описание              | Пример                 |
| ----------------------- | ------- | --------------------- | ---------------------- |
| **type**                | Type    | Тип поля              | `{ type: String }`     |
| **required**            | boolean | Обязательное поле     | `{ required: true }`   |
| **default**             | any     | Значение по умолчанию | `{ default: 0 }`       |
| **unique**              | boolean | Уникальное значение   | `{ unique: true }`     |
| **index**               | boolean | Создать индекс        | `{ index: true }`      |
| **enum**                | Array   | Перечисление значений | `{ enum: ['A', 'B'] }` |
| **min/max**             | number  | Ограничения для чисел | `{ min: 0, max: 100 }` |
| **minlength/maxlength** | number  | Ограничения для строк | `{ minlength: 3 }`     |

## Типы данных в TypeGoose:

``` typescript
// Базовые типы
@prop({ type: String })
public name!: string;

@prop({ type: Number })
public age!: number;

@prop({ type: Boolean })
public isActive!: boolean;

@prop({ type: Date })
public createdAt!: Date;

// Массивы
@prop({ type: [String] })
public tags!: string[];

// Вложенные объекты
@prop({ type: () => Address })
public address!: Address;

// Перечисления
@prop({ type: String, enum: UserRole })
public role!: UserRole;
```

