# Persistence

Есть разные способы работы с базой данных.

## Active Record

Примеры для ruby и ActiveRecord из Ruby on Rails.

Есть проблема с отслеживанием изменений.

```ruby
user = User.first
user.skills << "codding"
user.save
```

В ruby массивы мутабельны, соответственно ORM не может
отследить добавление нового навыка и не сохранит это изменение.
Можно конечно сразу после загрузки делать deep copy,
и при сохранении сравнивать текущее состояние с изначальным,
но не всегда это возможно и приемлемо.

Доступна загрузка ассоциаций по требованию:

```ruby
user.posts
```

Однако вполне возможна рассинхронизация состояния
базы данных и программы:

```ruby
user.posts.length #=> 2
Post.create user: user, other_attr: ""
user.posts.length #=> 2
```

Вы можете загрузить одну и ту же сущность в разные объекты:

```ruby
o1 = User.first
o2 = User.find(o1.id)

o1 != o2
```

Ваши сущности зависят от фреймворка (см. Dependency Inversion Principle)

```
class User < ActiveRecord::Base
  has_many :posts
end
```

Разумеется есть и другие особенности, но нам достаточно приведенных.

В целом, для своей ниши это отличная ORM,
но в сложных проектах она начинает откровенно вредить.

## Commands & Queries

Наиболее простой механизм.
Объявляются функции,
которые только извлекают данные и только изменяют данные.
Задавая вопрос, не меняй ответ.

```clojure
(defn perform [params]
  ...
  (let [user (queries/get-user-by-id some-id)
        post (post/build params)
        post (assoc post :author-id (:id user))]
    (commands/put-post post)
    ...))
```

Тут уже нет изменяемых объектов, user и post - просто структуры данных вроде map или record.
Таким образом вы не зависите от деталей реализации.

Естественно, не получится ходить про связям `user.posts`.

Вы по прежнему можете отобразить одну сущность в несколько объектов в памяти:

```clojure
(let [user (queries/get-user-by-id 1)
      user (update user :achievements conj :fishing)
      ...
      author (queries/get-user-by-id 1)
      author (update author :achievements conj :writing)]
  (commands/put-user user)
  ...
  (commands/put-user author))
```

В данном примере мы теряем часть изменений, а именно изменения "автора"
перетрут изменения пользователя.

Если используются транзакции,
и эти транзакции занимают некоторое время,
то при большом потоке изменений будут возникать
дедлоки и придется вручную расставлять блокировки.

Этот подход хорошо работает в функциональных языках и
просто языках без развитой инфраструткуры ORM.

## Data Mapper & Identity map & Unit of Work

+ https://martinfowler.com/eaaCatalog/dataMapper.html
+ https://martinfowler.com/eaaCatalog/identityMap.html
+ https://martinfowler.com/eaaCatalog/unitOfWork.html

Clojure имеет неизменяемые структуры данних и ссылочные типы.

Мы можем моделировать наши сущности используя record и ref:

```clojure
(defrecord Entity [id attr])

(let [entity (ref (->Entity 1 "some data"))]
  (dosync (alter entity assoc :attr "new data")))
```

При этом становится тривиальным реализация Identity map:

```clojure
(let [tx (db/build-tx)
      e1 (db/fetch tx 1)
      e2 (db/fetch tx 1)]
  (identical? e1 e2))
```

`tx` внутри себя хранит отобажение идентификаторов сущностей
на объекты в памяти:

```clojure
(defn fetch [tx id]
  (let [identity-map (:identity-map tx)
        from-memory  (get @identity-map id)]
    (if (some? from-memory)
      from-memory
      (let [state  (sql-fetch id)
            entity (ref state)]
        (swap! identity-map assoc id entity)
        entity))))
```

Когда приходит запрос на извлечении сущности, `db/fetch` сначала смотрит свою карту,
и если в ней есть нужный объект то возвращает его, иначе делает запрос в базу данных,
создает объект сущности из извлеченного состояния, добавляеет сущность в карту и
возвращает сущность.

Теперь мы хотим сохранить наши изменения:

```clojure
(let [tx (db/build-tx)
      e1 (db/fetch tx 1)
      e2 (db/fetch tx 2)]
  (dosync
   (let [attr1 (:attr @e1)
         attr2 (:attr @e2)]
     (alter e1 assoc :attr attr2)
     (alter e2 assoc :attr attr1)))
  (db/commit tx))
```

Как понять, что сущность изменилась? В этом поможет Единица работы(Unit of Work).
`tx` хранит изначальное сотояние сущности, и сравнивает его с текущим,
если оно отличается то идет запрос в базу.

Это позволяет не открывать транзакцию базы данных на все время выполнения бизнес транзакции.
В конце бизенс транзакции мы начинаем короткую транзакцию БД и быстро сохраняем измения.
В том числе это позволяет устранить дедлоки.

Да, предложенная схема работает только для выборок по первичному ключу.
Очень сложно реализовать Identity map имея возмоможность делать любые выборки.
Допустим мы выбираем по идентификатору пользователя с ролью Модератор
и меняем ее на Админ. В той же бизнес транзакции мы выбираем всех пользователей с ролью Модератор.
Должен ли первый пользователь попасть в выборку?

Мы можем воспользоваться Запросами(Query) и извлекать любые данные(состояние) вне транзакции
и перечитать данные находясь в транзакции:

```
(let [moderators-ids (queries/fetch-moderators)
      tx (db/build-tx)
      moderators (for [id moderators-ids]
                   (db/fetch tx id))]
  ...)
```

При этом происходит разделение.
Данные могут храниться на мастере, ассинхронно реплиципроваться в реплики,
через очередь индексироваться в поисковый движок.
API работы с транзакциями работает всегда с мастер базой,
а запросы могут исполняться где угодно и возвращать устаревшие данные.

Например, для задач массового изменения данных этот подход очевидно не подходит.
Но для этих редких случаев мы можем использовать функцию Command,
котрая выполнит необходимый запрос.

Мы не коснулись многих вопросов, и разберем их, когда будем проектировать
абстракции.

## Datomic / Datascript

+ https://www.datomic.com/
+ https://docs.datomic.com/cloud/whatis/data-model.html
+ https://github.com/tonsky/datascript