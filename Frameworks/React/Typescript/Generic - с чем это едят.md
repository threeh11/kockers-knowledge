## Что такое Generic?
Generic - это способ создавать переиспользуемые компоненты, функции и типы, которые могут работать с разными типами данных, сохраняя при этом типобезопасность.

## Зачем нужны Generic?
Без дженериков мы бы столкнулись с проблемами:
```
// Плохо: дублирование кода
function selectAllStrings(items: string[], selected: string[]): string[] {
  return items.length === selected.length ? [] : items;
}

function selectAllNumbers(items: number[], selected: number[]): number[] {
  return items.length === selected.length ? [] : items;
}

function selectAllSchedules(items: Schedule[], selected: Schedule[]): Schedule[] {
  return items.length === selected.length ? [] : items;
}

// Или еще хуже: потеря типобезопасности
function selectAllAny(items: any[], selected: any[]): any[] {
  return items.length === selected.length ? [] : items;
}
```
## На примере

```
// T - это параметр типа (generic parameter)
export function useSelectAll<T>({
  items: T[],           // T[] означает "массив элементов типа T"
  selectedItems: T[],   // Тот же тип T
  setSelectedItems: (items: T[]) => void,  // Функция принимает массив T
  filterFunction?: (item: T) => boolean,   // Функция принимает элемент типа T
}: UseSelectAllProps<T>) {
  // Весь код внутри работает с типом T
}
```

## Различные способы использования Generic
1.  Автовывод типов (Type Inference)
```
// TypeScript автоматически определяет тип
const schedules: Schedule[] = [...];
const { toggleSelectAll } = useSelectAll({
  items: schedules,        // T автоматически = Schedule
  selectedItems: [],       
  setSelectedItems: setSchedules,
});
```
2. Явное указание типа
```
// Мы явно указываем тип
const { toggleSelectAll } = useSelectAll<Schedule>({
  items: schedules,
  selectedItems: selectedSchedules,
  setSelectedItems: setSelectedSchedules,
});
```
3. Generic с ограничениями (Constraints)
```
// T должен иметь свойство id
interface HasId {
  id: string;
}

function useSelectAllWithId<T extends HasId>({
  items: T[],
  selectedItems: T[],
}: UseSelectAllProps<T>) {
  // Теперь мы можем использовать item.id
  const findById = (id: string) => items.find(item => item.id === id);
}
```

## Примеры использования в разных сценариях
1. Пример: Работа с пользователями
```
interface User {
  id: string;
  name: string;
  email: string;
}

const UserList = () => {
  const [users, setUsers] = useState<User[]>([]);
  const [selectedUsers, setSelectedUsers] = useState<User[]>([]);
  
  // Тип T автоматически определится как User
  const { allSelected, toggleSelectAll } = useSelectAll({
    items: users,
    selectedItems: selectedUsers,
    setSelectedItems: setSelectedUsers,
    filterFunction: (user) => user.email.includes('@gmail.com'), // только Gmail
  });
  
  return <SelectAllButton /* props */ />;
};
```
2. Пример: Работа с числами
```
const NumberSelector = () => {
  const [numbers] = useState<number[]>([1, 2, 3, 4, 5]);
  const [selectedNumbers, setSelectedNumbers] = useState<number[]>([]);
  
  // T = number
  const { allSelected, toggleSelectAll } = useSelectAll({
    items: numbers,
    selectedItems: selectedNumbers,
    setSelectedItems: setSelectedNumbers,
    filterFunction: (num) => num % 2 === 0, // только четные
  });
  
  return <div>{/* UI */}</div>;
};
```