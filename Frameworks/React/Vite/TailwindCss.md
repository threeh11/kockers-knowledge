Накатываем tailwind версии =="^4.1.7"== в проект vite на пресете ==react-ts==.

---
1. После развёртывания проекта устанавливаем необходимые для работы tailwind библиотеки:
`npm install -D @tailwindcss/postcss postcss autoprefixer`
2. Создаём [[tailwind.config.js]]  и [[postcss.config.js]]
3. Настраиваем [[index.scss]], чтобы подружить наш подключенный css к tailwind
Чтобы наш IDE подсказывал нам tailwind классы, также должно быть установлено расширение [Tailwind CSS IntelliSense](https://marketplace.cursorapi.com/items?itemName=bradlc.vscode-tailwindcss)


> [!NOTE] Autoprefixer
> Добавляет вендорные префиксы (`-webkit-`, `-moz-`) для кроссбраузерности.

> [!NOTE] PostCSS
> Инструмент, который обрабатывает CSS (как Babel — JS). Tailwind работает через него.
