## Описание

Компонент позволяет генерировать тестовые данные (персистентные сущности) с помощью интерфейса. Он будет полезен как для тестовых  и демо-проектов, так и для реальных проектов, в том случае, когда невозможно снять дамп.

## Возможности

- Разные типы генерации данных: 
  a) В случайном порядке
  b) Вручную
  c) Для генерации правдоподобных строк используется библиотека [Faker](https://github.com/serpro69/kotlin-faker). См. [data-providers](https://github.com/serpro69/kotlin-faker#data-providers).
- Массовая генерация (batch generation).
- Возможность просмотра и удаления сгенерированных сущностей.