## Основные понятия:

#### 1. Декоратор @ObjectType()

```typescript
@ObjectType({ description: "User model" })
class User { ... }
```

- Что это: Превращает класс в GraphQL тип
- Зачем: Создание типов для GraphQL схемы

#### 2. Декоратор @Field()

```typescript
@Field(() => String, { nullable: true })
public name?: string;
```

- Что это: Превращает поле класса в GraphQL поле
- Зачем: Контроль того, какие поля видны в GraphQL

#### 3. Декоратор @InputType()

```typescript
@InputType()
class CreateUserInput {
  @Field(() => String)
  name!: string;
}
```
- Что это: Тип для входных данных (аргументы мутаций)
- Зачем: Валидация и типизация входящих данных

#### 4. Декоратор @Resolver()

```typescript
@Resolver(() => User)
class UserResolver {
  @Query(() => [User])
  async users(): Promise<User[]> { ... }
}
```

- Что это: Класс с методами для обработки GraphQL запросов
- Зачем: Логика получения и изменения данных
### Опции @Field() декоратора:

| Опция                 | Тип     | Описание              | Пример                                  |
| --------------------- | ------- | --------------------- | --------------------------------------- |
| **nullable**          | boolean | Может быть null       | `{ nullable: true }`                    |
| **description**       | string  | Описание поля         | `{ description: "User name" }`          |
| **defaultValue**      | any     | Значение по умолчанию | `{ defaultValue: "Guest" }`             |
| **deprecationReason** | string  | Пометка устаревшего   | `{ deprecationReason: "Use fullName" }` |

### Типы GraphQL полей:

```typescript
// Скалярные типы
@Field(() => String)
public name!: string;

@Field(() => Int)
public age!: number;

@Field(() => Float)
public price!: number;

@Field(() => Boolean)
public isActive!: boolean;

@Field(() => Date)
public createdAt!: Date;

@Field(() => ID)
public id!: string;

// Массивы
@Field(() => [String])
public tags!: string[];

// Вложенные типы
@Field(() => Address)
public address!: Address;

// Перечисления
@Field(() => UserRole)
public role!: UserRole;
```