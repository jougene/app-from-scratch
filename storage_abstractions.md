# Абстракции хранилища

Прежде чем переходить к сценариям использования необходимо разобраться как хранить данные.

Итак, мы определились, что будем строить приложение по мотивам clean architecture.
И поняли принцип интверсии зависимости: модули верхнего уровня не зависят от модулей нижнего и оба зависят от абстракций.

Есть 2 подхода к работе с хранилищем:
[Active record](https://www.martinfowler.com/eaaCatalog/activeRecord.html)
и [Data mapper](https://www.martinfowler.com/eaaCatalog/dataMapper.html)

Как правило, фреймворки реализуют патерн Active record и предоставляют базовый класс для наших сущностей.
Такой подход идет вразрез с принципом инверсии зависимости.
Получается, что наша сущность зависит от деталей реализации.

Есть различные способы соблюсти этот принцип и использовать Active record.
Например, реализовать сущность как класс с абстрактными методами доступа к данным(find, save, udate), конкретные инстансы получать с помощью абстрактной фабрики.
Уровнями ниже должен быть объявлен класс, потомок нашей сущности, реализующий методы доступа к данным.

Считается, что более сложным является паттерн Data mapper.
При таком подходе наши сущности ничего не знают про persistence слой.
Это просто обычные объекты, в завсисимости от языка их называют
POJO (Plain Old Java Object).

При реализации Data mapper помогают шаблоны [Единица работы(Unit of work)](https://www.martinfowler.com/eaaCatalog/unitOfWork.html) и [Identity map](https://www.martinfowler.com/eaaCatalog/identityMap.html).

Unit of work отслеживает изменения объектов в памяти.
В конце бизнес транзакции происходит сохранение созданных/удаленных/измененных объектов.

Identity map задает соответствие между идентификатором сущности и ссылкой на объект.
Таким образом, повторно запрашивая сущность из хранилища вы получите тот же самый объект.





## Хранилище

Хранилище - это нечто, что сохраняет состояние сущностей.
Я очень долго думал и искал различные варианты того, как работать с хранилищем.


Модель предметной области находится в модулях самого высокого уровня.
Сущности не могут зависеть от

то сущности не должны зависеть
Сущности не должны зависеть от абстракции


Существуют

Для работы с сущностями нужна абстракция со следующими свойствами:

+ извлечение сущности по ее идентификатору
+ повторное извлечение сущности по идентификатору
  возвращает тот же объект что и предыдущее извлечение
  (Identity map)
+ сущности ничего не знают про хранилище,
  мы работаем с сущностаями как с простыми структурами данных
  (Datamapper)
+ поддержка ACID транзакций


Есть несколько требований


Хранилище нужно для сохраниения состояния сущностей.
Это может быть mysql, postgres, mongodb, elasticsearch, redis, файл, оперативная память.