# Case-study оптимизации

## Заметки от себя

1. Для реализации local_production — я решил все запускать также в режиме Development, но с другими параметрами.
   То есть, на метод development? Будет отвечать true

   Почему выбрал режим development.

   Если посмотреть код, то есть места, где если окружение отлично от development, то идут обращения к различным сервисам, для которых нужны свои API KEY. Также CarrierWave
   при production использует storage от AWS. Во избежание этого, основа - это development

   От создания своего дополнительного окружения я тоже отказался. Так как в этом случае придется добавлять, где идут проверки окружения, проверку на свое окружение

   ```
   if Rails.evn.development? || Rails.env.local_production?
   ```

   Поэтому, я решил сделать путем добавления файла в config/initializers.

   Файл называется 1_local_production_config.rb. Единицы в названии файла для того, чтобы Рельсы грузили файл одним из первых(ноль был уже занят =) )

   В этом файле мы проверяем, установлен ли у нас необходимый ENV, если да, переопределяются ключевые моменты конфига.

   Для того, чтобы это отработало, необходимо запускать рейл-сервер с допонительным EVN -

   ```
   CUSTOM_ENVIRONMENT=local_production bin/rails s -p 3000
   ```

   Можно придумать, конечно, более лаконичное название ENV, и более логичное значение, но пока в голову ничего не пришло =)

   Далее, я внес соответствующие правки в bin/startup. Для того, чтобы запустить через bin/startup режим local_production, мы вызываем команду

   ```
   bin/startup local_production
   ```

   После, идет вызов нужного нам Procfile.local_production и все работает, так, как задумано

2. К сожалению, таких проблемных мест, по типу single_story, больше найти не удалось. Есть места, где можно зафризить строки (например, articles/index.html.erb), но погоды бы эта оптимизация не сделала бы.

3. Сам по себе проект, на первый взгляд, как мне показалось, написал с уклоном на оптимизацию. Даже избавились от вызовов хелпера `link_to` и все ссылки прописаны обычным html

## Актуальная проблема

В проекте обнаружилась проблема, связанная с отображением списка постов.

И я попробую этот момент оптимизировать

## Формирование метрики

Для метрики я использовал benchmark AB (от Apache) и Skylight

## Вникаем в детали системы, чтобы найти главные точки роста

Для того, чтобы найти "точки роста" для оптимизации я воспользовался Skylight.

Вот какие проблемы удалось найти и решить

### Ваша находка №1

Я запустил siege и через Skylight посмотрел результат.
Вот что выдало
![sk first](https://i.ibb.co/Y2h8mr7/sk-1.png)

Заходим в index и смотрим
![sk second](https://i.ibb.co/qy9Msmf/sk-2.png)

Видим, что больше всех кушает single_story

Запускаем benchmark AB 100 запросов, выдает следующее

```
Time taken for tests: 17.17
Requests per second:    5.82
```

Я решил для начала, попробовать переписать на использование collection и посмотреть, будет ли прирост производительности.
Реализовав это, AB показал следующие результаты

```
Времени потребовалось — 15.45
Requests per second:    6.47
```

Как видим, производительность все же это повысило, хоть и нельзя назвать это значительным достижением.
А можно даже списать и на погрешность

В данных представлениях практически нечего фризить. А те фризы строк, что можно сделать, не сыграют никакой роли.
И единственным вариантом осталось закешировать все это дело

После кеширования (добавление cache: true в collection)

В итоге такие результаты

```
Time taken for tests:   6.535 seconds
Requests per second:    15.30
```

Результат существенный.

Хотя, если взять и запустить еще раз, то выполнится, практически, мгновенно

```
Time taken for tests:   3.255 seconds
Requests per second:    30.72
```

Сила кэша =)

Дальше, если посмотреть, то видим, что у tags тоже есть подобный вызов

![sk_third](https://i.ibb.co/HBvhKKg/sk-3.jpg)

поэтому приведу его к такому же виду(collection) и закеширую

Особо больше ничего "выдающегося" найти не удалось.
Тут практически все вьюхи обмазаны кэшем, и в таком случае сложнее что-то увидеть.
Также, судя по контроллерам, запросам и тому, что говорит bullet - все в норме.

За исключением моментов, когда bullet говорит, что есть лишний includes(:user) или наоброт просит добавить includes(:organization) - но, например, при попытке добавить preload(:organization) к stories, жалуется, что надо его убрать =)

В таком случае, думаю, можно было бы сделать LEFT JOIN и в select добавить нужные поля по организации.
Но опять же, тут все кэшируется

## Результаты

В результате проделанной работы удалось существенно повысить производительность страницы StoriesController#index

Было

```
Time taken for tests: 17.17
Requests per second:    5.82
```

Стало

```
Time taken for tests:   6.535 seconds
Requests per second:    15.30
```

И когда кэщ "прогреется", то

```
Time taken for tests:   3.255 seconds
Requests per second:    30.72
```

Также, познакомился со Skylight, siege и AB - очень удобные инструменты
