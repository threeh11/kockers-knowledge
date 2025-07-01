useMemo - это один из самых важных хуков для оптимизации производительности. Давайте разберем его досконально с практическими примерами.

## Что такое useMemo?

useMemo - это хук, который мемоизирует (кэширует) результат вычисления и пересчитывает его только тогда, когда изменяются зависимости.

```
const memoizedValue = useMemo(() => {
  // Дорогое вычисление
  return expensiveCalculation(a, b);
}, [a, b]); // Пересчитывается только при изменении a или b
```

## Когда НЕ нужно использовать useMemo

 ❌ Простые вычисления

```
// ПЛОХО - бессмысленная оптимизация
const simpleSum = useMemo(() => a + b, [a, b]);

// ХОРОШО - простое вычисление
const simpleSum = a + b;
```
 
 ❌ Вычисления, которые выполняются быстро

```
// ПЛОХО - строки склеиваются мгновенно
const fullName = useMemo(() => `${firstName} ${lastName}`, [firstName, lastName]);

// ХОРОШО
const fullName = `${firstName} ${lastName}`;
```
## Когда НУЖНО использовать useMemo

 ✅ 1. Дорогие вычисления

```
// Пример: фильтрация и сортировка большого массива
const ExpensiveList = ({ items, searchTerm, sortBy }) => {
  // БЕЗ useMemo - выполняется при каждом рендере
  const processedItems = items
    .filter(item => item.name.toLowerCase().includes(searchTerm.toLowerCase()))
    .map(item => ({ ...item, computed: heavyCalculation(item) }))
    .sort((a, b) => a[sortBy] - b[sortBy]);

  // С useMemo - выполняется только при изменении зависимостей
  const processedItems = useMemo(() => {
    return items
      .filter(item => item.name.toLowerCase().includes(searchTerm.toLowerCase()))
      .map(item => ({ ...item, computed: heavyCalculation(item) }))
      .sort((a, b) => a[sortBy] - b[sortBy]);
  }, [items, searchTerm, sortBy]);
  
  return <div>{/* рендер списка */}</div>;
};
```
 
 ✅ 2. Создание объектов/массивов для пропсов

```
const ParentComponent = ({ users, filter }) => {
  // БЕЗ useMemo - новый объект при каждом рендере
  // Дочерний компонент будет перерендериваться даже если данные не изменились
  const config = {
    showNames: true,
    allowEdit: false,
    theme: 'dark'
  };

  // С useMemo - объект создается только при изменении зависимостей
  const config = useMemo(() => ({
    showNames: true,
    allowEdit: false,
    theme: 'dark'
  }), []); // Пустой массив = создается один раз

  return <UserList users={users} config={config} />;
};
```

✅ 3. Наш случай: фильтрация по условию

Разберем наш код:

```
const filteredItems = useMemo(() => {
  return filterFunction ? items.filter(filterFunction) : items;
}, [items, filterFunction]);
```

Почему здесь нужен useMemo?

1. Метод filter() создает новый массив - это относительно дорогая операция для больших массивов

2. Функция может вызываться часто - при каждом изменении состояния компонента

3. Результат используется в других вычислениях - влияет на allSelected

## Практические примеры из реальных проектов

### Пример 1: Поиск и фильтрация

```
const ProductList = ({ products, searchTerm, category, priceRange }) => {
  // Без оптимизации - выполняется при каждом рендере
  const filteredProducts = useMemo(() => {
    console.log('Фильтруем продукты...'); // Вы увидите это только при изменении зависимостей
    
    return products
      .filter(product => {
        // Поиск по названию
        const matchesSearch = product.name
          .toLowerCase()
          .includes(searchTerm.toLowerCase());
        
        // Фильтр по категории
        const matchesCategory = !category || product.category === category;
        
        // Фильтр по цене
        const matchesPrice = product.price >= priceRange.min && 
                            product.price <= priceRange.max;
        
        return matchesSearch && matchesCategory && matchesPrice;
      })
      .sort((a, b) => a.name.localeCompare(b.name)); // Сортировка
  }, [products, searchTerm, category, priceRange]);

  return (
    <div>
      {filteredProducts.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  );
};
```
### Пример 2: Группировка данных

```
const ScheduleView = ({ schedules }) => {
  // Группируем расписания по датам
  const groupedSchedules = useMemo(() => {
    console.log('Группируем расписания...');
    
    return schedules.reduce((groups, schedule) => {
      const date = dayjs(schedule.scheduled_time).format('YYYY-MM-DD');
      if (!groups[date]) {
        groups[date] = [];
      }
      groups[date].push(schedule);
      return groups;
    }, {});
  }, [schedules]);

  // Сортируем даты
  const sortedDates = useMemo(() => {
    return Object.keys(groupedSchedules).sort();
  }, [groupedSchedules]);

  return (
    <div>
      {sortedDates.map(date => (
        <div key={date}>
          <h3>{date}</h3>
          {groupedSchedules[date].map(schedule => (
            <ScheduleCard key={schedule.id} schedule={schedule} />
          ))}
        </div>
      ))}
    </div>
  );
};
```
### Пример 3: Вычисляемые свойства

```
const Statistics = ({ orders }) => {
  const stats = useMemo(() => {
    console.log('Вычисляем статистику...');
    
    const total = orders.reduce((sum, order) => sum + order.amount, 0);
    const average = total / orders.length;
    const maxOrder = Math.max(...orders.map(o => o.amount));
    const minOrder = Math.min(...orders.map(o => o.amount));
    
    return { total, average, maxOrder, minOrder };
  }, [orders]);

  return (
    <div>
      <div>Общая сумма: {stats.total}</div>
      <div>Средний чек: {stats.average}</div>
      <div>Максимальный заказ: {stats.maxOrder}</div>
      <div>Минимальный заказ: {stats.minOrder}</div>
    </div>
  );
};
```
## Как понять, нужен ли useMemo?

### 🔍 Способ 1: Добавьте console.log

```
// Добавьте лог для отслеживания частоты выполнения
const expensiveResult = useMemo(() => {
  console.log('Выполняется дорогое вычисление');
  return doExpensiveCalculation();
}, [dependency]);
```

Если лог появляется слишком часто без изменения зависимостей - useMemo не работает.

### 🔍 Способ 2: Замерьте производительность

```
const expensiveResult = useMemo(() => {
  console.time('expensive-calculation');
  const result = doExpensiveCalculation();
  console.timeEnd('expensive-calculation');
  return result;
}, [dependency]);
```
### 🔍 Способ 3: React DevTools Profiler

Используйте профайлер для измерения времени рендера компонентов.

## Распространенные ошибки

 ❌ Ошибка 1: Забыли зависимости

```
const filteredItems = useMemo(() => {
  return items.filter(item => item.status === currentStatus);
}, [items]); // ❌ Забыли currentStatus!

// ✅ Правильно
const filteredItems = useMemo(() => {
  return items.filter(item => item.status === currentStatus);
}, [items, currentStatus]);
```
### ❌ Ошибка 2: Слишком много зависимостей

```
const result = useMemo(() => {
  return expensiveCalc(a);
}, [a, b, c, d, e, f]); // ❌ Только 'a' используется в вычислении!

// ✅ Правильно
const result = useMemo(() => {
  return expensiveCalc(a);
}, [a]);
```
### ❌ Ошибка 3: Объекты в зависимостях

```
const config = { theme: 'dark', size: 'large' };

const result = useMemo(() => {
  return processData(data, config);
}, [data, config]); // ❌ config - новый объект каждый раз!

// ✅ Правильно - мемоизируем конфиг
const config = useMemo(() => ({ 
  theme: 'dark', 
  size: 'large' 
}), []);

const result = useMemo(() => {
  return processData(data, config);
}, [data, config]);
```
## Правила принятия решения

### ✅ Используйте useMemo когда:

1. Дорогие вычисления (фильтрация больших массивов, сложные расчеты)

2. Создание объектов/массивов для пропсов дочерних компонентов

3. Зависимость для других хуков (useEffect, useMemo, useCallback)

4. Результат влияет на рендер многих компонентов

### ❌ НЕ используйте useMemo когда:

1. Простые вычисления (сложение, конкатенация строк)

2. Вычисления выполняются быстро (< 1ms)

3. Зависимости меняются при каждом рендере

4. Добавляете "на всякий случай"

## Заключение

В нашем хуке useSelectAll:

```
const filteredItems = useMemo(() => {
  return filterFunction ? items.filter(filterFunction) : items;
}, [items, filterFunction]);
```

Это оправдано, потому что:

1. filter() может быть дорогой операцией для больших массивов расписаний

2. Результат используется в других вычислениях (allSelected)

3. Хук может вызываться часто при изменении состояния родительского компонента

4. Фильтрация выполняется только при реальном изменении данных или функции фильтрации

Помните: useMemo - это оптимизация. Сначала напишите рабочий код, потом оптимизируйте там, где это действительно нужно!