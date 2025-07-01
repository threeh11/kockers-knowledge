useCallback - это близкий родственник useMemo, но для функций. Давайте разберем его также подробно!

## Что такое useCallback?

```
const memoizedCallback = useCallback(() => {
  // Логика функции
  doSomething(a, b);
}, [a, b]); // Функция пересоздается только при изменении a или b
```
## Ключевое отличие от useMemo

```
// useMemo мемоизирует РЕЗУЛЬТАТ вычисления
const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);

// useCallback мемоизирует САМУ ФУНКЦИЮ
const memoizedCallback = useCallback((x) => doSomething(a, b, x), [a, b]);

// Это эквивалентно:
const memoizedCallback = useMemo(() => (x) => doSomething(a, b, x), [a, b]);
```
## Когда НЕ нужно использовать useCallback

❌ Функции, которые не передаются как пропсы

```
const MyComponent = () => {
  // ПЛОХО - функция используется только внутри компонента
  const handleLocalClick = useCallback(() => {
    console.log('Clicked!');
  }, []);

  // ХОРОШО - простое объявление
  const handleLocalClick = () => {
    console.log('Clicked!');
  };

  return <button onClick={handleLocalClick}>Click me</button>;
};
```

 ❌ Функции без зависимостей, которые создаются редко

```
const MyComponent = () => {
  // ПЛОХО - бессмысленная оптимизация
  const simpleHandler = useCallback(() => {
    alert('Hello!');
  }, []);

  // ХОРОШО
  const simpleHandler = () => {
    alert('Hello!');
  };

  return <Button onClick={simpleHandler} />;
};
```
## Когда НУЖНО использовать useCallback

 ✅ 1. Функции, передаваемые в memo-компоненты

```
// Дочерний компонент обернут в memo
const ExpensiveChild = React.memo(({ onAction, data }) => {
  console.log('ExpensiveChild рендерится');
  return <div onClick={onAction}>{data}</div>;
});

const ParentComponent = ({ items }) => {
  const [count, setCount] = useState(0);

  // БЕЗ useCallback - новая функция при каждом рендере
  // ExpensiveChild будет перерендериваться даже при изменении count
  const handleAction = (id) => {
    console.log('Action for:', id);
    // Какая-то логика
  };

  // С useCallback - функция стабильна
  // ExpensiveChild рендерится только при изменении items
  const handleAction = useCallback((id) => {
    console.log('Action for:', id);
    processItem(id, items);
  }, [items]);

  return (
    <div>
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>
      {items.map(item => (
        <ExpensiveChild 
          key={item.id} 
          data={item.name}
          onAction={() => handleAction(item.id)} // ❌ Все равно новая функция!
        />
      ))}
    </div>
  );
};
```

Важно! В примере выше есть проблема - мы все равно создаем новую функцию в onAction={() => handleAction(item.id)}.
Правильное решение:

```
const ParentComponent = ({ items }) => {
  const [count, setCount] = useState(0);

  // Создаем стабильную функцию
  const handleAction = useCallback((id) => {
    console.log('Action for:', id);
    processItem(id, items);
  }, [items]);

  return (
    <div>
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>
      {items.map(item => (
        <ExpensiveChild 
          key={item.id} 
          data={item.name}
          itemId={item.id}
          onAction={handleAction} // ✅ Передаем стабильную функцию
        />
      ))}
    </div>
  );
};

// Дочерний компонент сам вызывает функцию с нужным ID
const ExpensiveChild = React.memo(({ onAction, data, itemId }) => {
  const handleClick = () => onAction(itemId);
  
  return <div onClick={handleClick}>{data}</div>;
});
```

✅ 2. Зависимости для других хуков

```
const DataComponent = ({ userId }) => {
  const [data, setData] = useState(null);

  // БЕЗ useCallback - useEffect перезапускается при каждом рендере
  const fetchData = async (id) => {
    const response = await api.fetchUser(id);
    setData(response.data);
  };

  useEffect(() => {
    fetchData(userId);
  }, [fetchData, userId]); // ❌ fetchData меняется каждый раз!

  // С useCallback - useEffect запускается только при изменении userId
  const fetchData = useCallback(async (id) => {
    const response = await api.fetchUser(id);
    setData(response.data);
  }, []); // Функция стабильна

  useEffect(() => {
    fetchData(userId);
  }, [fetchData, userId]); // ✅ fetchData стабильна

  return <div>{data?.name}</div>;
};
```

✅ 3. Наш случай: стабильная функция фильтрации

Посмотрим на код:

```
const filterFunction = React.useCallback((schedule: Schedule) => {
  if (isAcceptingMode) {
    return schedule.status === 'DRAFT';
  }
  return schedule.status !== 'DRAFT' && schedule.status === targetStatus;
}, [isAcceptingMode, targetStatus]);
```

Почему здесь нужен useCallback?

1. Функция передается в useSelectAll хук

2. useSelectAll использует эту функцию как зависимость для useMemo

3. Без useCallback функция пересоздавалась бы при каждом рендере

4. Это приводило бы к ненужным пересчетам в useMemo

## Практические примеры

### Пример 1: Обработчики событий для списков

```
const TodoList = ({ todos }) => {
  const [editingId, setEditingId] = useState(null);

  // ✅ Стабильные обработчики для оптимизации рендера дочерних компонентов
  const handleToggle = useCallback((id) => {
    setTodos(prev => prev.map(todo => 
      todo.id === id ? { ...todo, completed: !todo.completed } : todo
    ));
  }, []);

  const handleDelete = useCallback((id) => {
    setTodos(prev => prev.filter(todo => todo.id !== id));
  }, []);

  const handleEdit = useCallback((id) => {
    setEditingId(id);
  }, []);

  const handleSave = useCallback((id, newText) => {
    setTodos(prev => prev.map(todo => 
      todo.id === id ? { ...todo, text: newText } : todo
    ));
    setEditingId(null);
  }, []);

  return (
    <div>
      {todos.map(todo => (
        <TodoItem
          key={todo.id}
          todo={todo}
          isEditing={editingId === todo.id}
          onToggle={handleToggle}
          onDelete={handleDelete}
          onEdit={handleEdit}
          onSave={handleSave}
        />
      ))}
    </div>
  );
};

// Мемоизированный компонент задачи
const TodoItem = React.memo(({ todo, isEditing, onToggle, onDelete, onEdit, onSave }) => {
  console.log(`Рендер TodoItem ${todo.id}`); // Будет логироваться только при реальных изменениях
  
  return (
    <div>
      {/* UI компонента */}
    </div>
  );
});
```
### Пример 2: Дебаунс и троттлинг

```
const SearchComponent = () => {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  // Создаем стабильную debounced функцию
  const debouncedSearch = useCallback(
    debounce(async (searchQuery) => {
      if (searchQuery) {
        const data = await api.search(searchQuery);
        setResults(data);
      } else {
        setResults([]);
      }
    }, 300),
    [] // Функция создается один раз
  );

  // useEffect с стабильной зависимостью
  useEffect(() => {
    debouncedSearch(query);
  }, [query, debouncedSearch]);

  return (
    <div>
      <input 
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Поиск..."
      />
      <SearchResults results={results} />
    </div>
  );
};
```
### Пример 3: Кастомные хуки

```
// Кастомный хук для API вызовов
const useApi = (endpoint) => {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  // Стабильная функция для рефетча
  const refetch = useCallback(async () => {
    setLoading(true);
    setError(null);
    try {
      const response = await fetch(endpoint);
      const result = await response.json();
      setData(result);
    } catch (err) {
      setError(err);
    } finally {
      setLoading(false);
    }
  }, [endpoint]);

  // Первоначальная загрузка
  useEffect(() => {
    refetch();
  }, [refetch]);

  return { data, loading, error, refetch };
};

// Использование
const UserProfile = ({ userId }) => {
  const { data: user, loading, error, refetch } = useApi(`/users/${userId}`);

  // Передаем стабильную функцию refetch дочерним компонентам
  return (
    <div>
      {loading && <Spinner />}
      {error && <ErrorMessage onRetry={refetch} />}
      {user && <UserCard user={user} onRefresh={refetch} />}
    </div>
  );
};
```
## Распространенные ошибки

❌ Ошибка 1: Забыли зависимости

```
const MyComponent = ({ userId, filters }) => {
  const [data, setData] = useState([]);

  const fetchData = useCallback(async () => {
    const result = await api.getData(userId, filters);
    setData(result);
  }, []); // ❌ Забыли userId и filters!

  // ✅ Правильно
  const fetchData = useCallback(async () => {
    const result = await api.getData(userId, filters);
    setData(result);
  }, [userId, filters]);
};
```

❌ Ошибка 2: Излишнее использование

```
const MyComponent = () => {
  // ❌ Ненужный useCallback для внутренней функции
  const handleClick = useCallback(() => {
    console.log('Internal click');
  }, []);

  // ✅ Простое объявление
  const handleClick = () => {
    console.log('Internal click');
  };

  return <div onClick={handleClick}>Click me</div>;
};
```

❌ Ошибка 3: Объекты в зависимостях

```
const MyComponent = ({ config }) => {
  // ❌ config может быть новым объектом каждый раз
  const handleSubmit = useCallback((data) => {
    processData(data, config);
  }, [config]);

  // ✅ Деструктурируем нужные свойства
  const { apiUrl, timeout } = config;
  const handleSubmit = useCallback((data) => {
    processData(data, { apiUrl, timeout });
  }, [apiUrl, timeout]);
};
```
## Продвинутые паттерны

### 1. useCallback с useRef для доступа к актуальному состоянию

```
const ChatComponent = () => {
  const [messages, setMessages] = useState([]);
  const messagesRef = useRef(messages);
  
  // Обновляем ref при изменении messages
  useEffect(() => {
    messagesRef.current = messages;
  }, [messages]);

  // Стабильная функция, которая всегда имеет доступ к актуальным messages
  const handleNewMessage = useCallback((message) => {
    const currentMessages = messagesRef.current;
    setMessages([...currentMessages, message]);
  }, []); // Нет зависимостей, но функция работает с актуальными данными

  useEffect(() => {
    socket.on('message', handleNewMessage);
    return () => socket.off('message', handleNewMessage);
  }, [handleNewMessage]);
};
```
### 2. Условный useCallback

```
const DataTable = ({ data, sortable }) => {
  // Создаем callback только если нужна сортировка
  const handleSort = useCallback(
    sortable 
      ? (column) => {
          setSortConfig({ column, direction: 'asc' });
        }
      : undefined,
    [sortable]
  );

  return (
    <Table 
      data={data} 
      onSort={sortable ? handleSort : undefined}
    />
  );
};
```
## Как понять, нужен ли useCallback?

### 🔍 Способ 1: React DevTools Profiler

Посмотрите, какие компоненты рендерятся без необходимости.

### 🔍 Способ 2: Логирование в дочерних компонентах

```
const ChildComponent = React.memo(({ onClick, data }) => {
  console.log('ChildComponent рендер', data.id);
  return <div onClick={onClick}>{data.name}</div>;
});
```

### 🔍 Способ 3: Проверка ссылочного равенства

```
const MyComponent = () => {
  const callback1 = () => console.log('Without useCallback');
  const callback2 = useCallback(() => console.log('With useCallback'), []);

  useEffect(() => {
    console.log('callback1 changed'); // Будет логироваться каждый рендер
  }, [callback1]);

  useEffect(() => {
    console.log('callback2 changed'); // Логируется только один раз
  }, [callback2]);
};
```

## Правила принятия решения

### ✅ Используйте useCallback когда:

1. Функция передается в memo-компоненты

2. Функция является зависимостью для других хуков

3. Функция передается в дочерние компоненты, которые оптимизированы

4. Функция содержит дорогие операции или создает замыкания

### ❌ НЕ используйте useCallback когда:

1. Функция используется только внутри компонента

2. Дочерние компоненты не оптимизированы (не используют memo)

3. Зависимости функции меняются при каждом рендере

4. Добавляете "на всякий случай"

## Заключение

В нашем компоненте SelectAllButton:

```
const filterFunction = React.useCallback((schedule: Schedule) => {
  if (isAcceptingMode) {
    return schedule.status === 'DRAFT';
  }
  return schedule.status !== 'DRAFT' && schedule.status === targetStatus;
}, [isAcceptingMode, targetStatus]);
```

Это оправдано, потому что:

1. Функция передается в хук useSelectAll

2. Хук использует функцию как зависимость для useMemo

3. Без useCallback функция пересоздавалась бы каждый рендер

4. Это приводило бы к ненужным пересчетам фильтрации

Помните: useCallback - это оптимизация для предотвращения ненужных рендеров. Используйте его осознанно, когда видите реальную проблему производительности