---
layout: post
title:  "Знакомимся с Redis"
date:   2018-03-25 09:13:30 +0300
categories: technologies
tags: [technologies, redis, cache, nosql]
image:
  feature: 2018-03-25/redis.png
  teaser: 2018-03-25/redis-teaser.png
---

Redis — NoSQL база данных типа ключ-значение. Redis хранит данные в оперативной памяти, что является ключевой особенностью этого хранилища. Поэтому он очень быстрый, но не самый надежный. Периодически Redis сбрасывает все данные на диск, но если сервер упадет в момент между добавлением новой информации и сохранением на диск, данные будут потеряны. По этим причинам Redis часто используют не как основное хранилище, а в качестве кэша, системы управления сессиями или для решения другой задачи, где не страшно потерять данные. Сегодня мы познакомимся с основными возможностями этой базы данных.


# Особенности Redis

Перечислим главные принципы Redis:

- **Производительность**. Redis действительно быстрый. Утилита *redis-benchmark* (вы найдете ее в папке вместе с остальными бинарными файлами) покажет насколько. Например, операции *get* и *set* в количестве 100 тысяч запросов выполняются чуть больше чем за секунду.
- **Персистентность**. Redis сохраняет снимки базы данных на диск. Можно настроить период сохранения данных в зависимости от количества обновленных значений. Также можно использовать режим дозаписи. Для случаев когда, например, Redis используется в качестве кэша, сохранение на диск можно отключить вовсе.
- **Структуры данных**. Это то, из-за чего многие не хотят называть Redis хранилищем ключ-значение. Дело в том, что он умеет хранить не только пары строка-строка. Redis поддерживает шесть типов данных и различные операции над ними. В следующем разделе мы обсудим каждый из этих типов.
- **Распределенность**. Оперативной памяти одного сервера может не хватить для использования Redis в серьезном приложении. Тут вам поможет Redis Cluster. Он поддерживает шардирование, репликацию, отказоустойчивость и другие полезные возможности из мира распределенных систем. Настройкой кластера Redis мы займемся в другой раз.

# Типы данных и команды

Redis хранит в себе пары ключ-значение. Ключом должен быть строковый литерал, но в качестве значения можно использовать один из шести доступных типов. Чтобы понимать для каких сценариев подходит Redis, нужно разобраться в том, что это за типы и какие операции с ними возможны.

## Ключи

Ключ — всегда строка. Тем не менее есть несколько важных моментов и рекомендаций, которые нужно знать при работе с ключами:

- Не следует использовать очень длинные ключи. Чем ключ длиннее, тем дольше производится поиск по нему. Мегабайтный массив -- плохой вариант для ключа.
- Не стремитесь сократить ключ, во что бы то не стало. Используйте читаемые имена, чтобы можно было понять, какую сущность хранит этот ключ.
- Старайтесь придерживать схемы. Хорошая практика -- использовать единую схему для ваших ключей. Разделяйте сущности в названии двоеточиями, а названия из нескольких слов -- дефисами или точками. Пример: *users:1000:best-friends*.
- На ключ есть ограничение по размеру: 512 Mb.

## Строки

В качестве строк может использоваться все, что угодно. Сохраняйте числа, JSON, байты изображения или что-нибудь еще. Тем не менее набор операций для работы с этим типом данных ограничен, поэтому, например, извлечь нужное значение из JSON вам не удастся.

Чтобы добавить в хранилище новое строковое значение, используйте команду *set*.
{% highlight redis %}
  set key value
{% endhighlight %}

Для того, чтобы получить строку по ключу — команду *get*.
{% highlight redis %}
  get key
{% endhighlight %}

Отметим, что *set* обновит значение, если такой ключ существует, причем вне зависимости от того, значение какого типа хранилось до этого. Это поведение можно изменить, используя опции *nx* — создать только, если такого ключа нет, и *xx* — создать (обновить) только, если ключ существует.
{% highlight redis %}
  set key value nx/xx
{% endhighlight %}

Если в качестве строки используется число, то можно использовать команды *incr*, *decr* и *incrby*, *decrby*. Первая пара увеличивает и уменьшает число на единицу, вторая делает это на заданное число.
{% highlight redis %}
  incr counter
  incrby counter 10
{% endhighlight %}

Если строка не является числом, то операция вернет ошибку.


## Общие команды

В Redis есть набор команд, которые можно применять к любому типу данных:

- *exists* — возвращает 1, если ключ существует и 0, если нет.
- *del* - удаляет ключ.
- *type* - возвращает тип значения по ключу.
- *expire* - удаляет ключ по прошествии указанного времени. 
- *ttl* - возвращает число: сколько времени осталось ключу до удаления.

С полным перечнем команд Redis можно ознакомиться на [официальном сайте](https://redis.io/commands).

## Списки

Для того, чтобы понимать, для каких операций лучше подходит этот тип данных, нужно понимать как он устроен. Список в Redis реализован как связанный список, в отличие от многих других реализаций, использующих массив. В связи с этим, со списками быстро работают команды добавления элементов в начало или конец и извлечения последовательных элементов. Если есть необходимость в доступе к элементам из середины большой коллекции, то лучше использовать упорядоченные множества.

С помощью команд *lpush* и *rpush* элементы добавляются в начало и конец списка. Если ключ не существует, то сначала создается пустой список с таким ключом, а затем в него добавляется элемент. Такое поведение предусмотрено для всех видов коллекций.
{% highlight redis %}
  lpush mylist 1 2 3
{% endhighlight %}

Команда *lrange* возвращает подмножество списка по заданным индексам. При этом знать длину списка не обязательно, так как можно использовать отрицательные индексы. Например, чтобы вернуть весь список, используйте команду:
{% highlight redis %}
  lrange mylist 0 -1
{% endhighlight %}

Команда *lrange* не удаляет элементы из списка. Чтобы извлечь, при этом удалив элемент, используйте команды *lpop* и *rpop*. При удалении последнего элемента из коллекции список также удаляется. Если после этого использовать команду *llen*, которая возвращает длину списка, то она вернет такое значение, как если бы коллекция была пустой.
{% highlight redis %}
  lpop mylist
{% endhighlight %}

*ltrim* — еще одна полезная команда при работе со списками. Она принимает аргументы, так же как и *lrange*, но не возвращает, а обрезает список. Если формальнее, то она создает новый список, ассоциированный с этим ключом, и заполняет его элементами из исходного списка в соответствии с переданными индексами.
{% highlight redis %}
  ltrim mylist 0 2
{% endhighlight %}

## Хеши

С помощью хешей можно хранить объекты, так как он содержит набор пар ключ-значение. В роли ключей могут использоваться только строковые значения. Для добавления элементов в хеш используйте команду *hmset* вместе с набором пар.
{% highlight redis %}
  hmset hash a 10 b 20
{% endhighlight %}

Получение значения по ключу осуществляется с помощью команды *hget*, получить все пары же можно командой *hgetall*.
{% highlight redis %}
  hget hash a
  hgetall hash
{% endhighlight %}

Хеши — мощный инструмент, они подходят для самых разных сценариев работы с Redis.

## Множества

Множества в Redis — неупорядоченные наборы строк. Они поддерживают стандартные операции между множествами, такие как пересечение, объединение, разность и другие. Для добавления элементов в множество используйте команду *sadd*.
{% highlight redis %}
  sadd myset 1 2 3
{% endhighlight %}

Некоторые операции над множествами:
{% highlight redis %}
  smember myset 2
  sunion store myset myset2
{% endhighlight %}

## Упорядоченные множества

Упорядоченные множества — симбиоз обычных множеств и списков. Дело в том, что они содержат только уникальные значения, но каждому значению соответствуют число (score). В результате для это типа данных вводится порядок:

- *A > B*, если *A.score > B.score*
- если *A.score = B.score*, то *A* и *B* упорядочиваются в соответствии с лексикографическим порядком значений. Так как они уникальны, равенство двух различных элементов в упорядоченном множестве невозможно.

Чтобы добавить новые элементы используйте команду *zadd*, а для получения всех элементов в порядке возрастания — *zrange*:
{% highlight redis %}
  zadd authors 1902 Tolkien
  zrange authors 0 -1
{% endhighlight %}

Если нужен обратный порядок для элементов упорядоченного множества, то есть команда *zrevrange*:
{% highlight redis %}
  zrevrange authors 0 -1
{% endhighlight %}

Мы обсудили пять типов данных в Redis. Есть также шестой тип, он называется *HyperLogLogs*. Он необходим для быстрого подсчета уникальных элементов. Пример использования этого типа: подсчет уникальных запросов в поисковой системе.

# Реализуем кэширование с Redis

Как уже было сказано, хранить важную информацию только в Redis рискованно, поэтому в большинстве сценариев это хранилище используется в качестве кэша. В учебных целях я решил прикрутить кэширование с помощью Redis к одному из своих домашних проектов. Для этого была раскопана десктопная пародия на Instagram. Проект написан на платформе .NET: UI на WPF, REST API на ASP.NET и база данных MS SQL Server. Код можно посмотреть на [GitHub](https://github.com/AlexeyKalina/Digital-Design-School/tree/master/InstagramKiller).

Задача состояла в том, чтобы добавить уровень кэша для SQL-запросов к основному хранилищу. При реализации кэширования нужно учитывать несколько идей:

- *Не пытаться закэшировать все*. Кэширование нужно, чтобы сократить количество обращений к основному хранилищу. Однако, совсем от них избавиться не удастся. Поэтому не нужно пытаться поместить в кэш ответы на любой запрос. Пусть там хранятся ответы на самые частые запросы -- нужно выдерживать грань.
- *Не бояться дублирующихся данных в кэше*. Подход к хранению информации в кэше отличается от применяющегося в реляционной модели, где данные должны быть нормализованы и быть в единственном экземпляре. В некоторых случаях засчет дублирования информации можно значительно выиграть в скорости, а это принципиально важно для кэша.
- *Инвалидация кэша*. Нужно понимать, в какой момент данные из кэша должны исчезнуть или обновиться. Redis поддерживает идею TTL (Time To Live), в которой ключи удаляются по прошествии определенного времени. В реальных проектах стоит использовать эту парадигму; в своем проекте я буду удалять данные, при соответствующих операциях в базе данных. Проблема такого подхода в том, что за всем не уследишь, и в какой-то момент в кэше станет слишком много невалидных данных. Но для учебного проекта это не страшно.

В роли клиента для Redis на языке C# была выбрана популярная библиотека [StackExchange.Redis](https://github.com/StackExchange/StackExchange.Redis) (5 миллионов скачиваний в Nuget). Подключение к хранилищу осуществляется с помощью класса *ConnectionMultiplexer*:
{% highlight C#  %}
public RedisCacheProvider(string connectionString)
{
    _redisConnection = ConnectionMultiplexer.Connect(connectionString);
}
{% endhighlight %}

Чтобы выполнять команды в конкретной базе данных (в нашем случае дефолтной), используется метод *GetDatabase()*
{% highlight C#  %}
private IDatabase GetRedisDb()
{
    return _redisConnection.GetDatabase();
}
{% endhighlight %}

Обертки для команд реализуются с помощью клиента Redis очень просто. Вот пара примеров:
{% highlight C#  %}
public void Add<T>(string key, T value)
{
    var db = GetRedisDb();
    var redisValue = SerializeToJson(value);
    db.StringSet(key, redisValue);
}

public void Delete(string key)
{
    var db = GetRedisDb();
    db.KeyDelete(key);
}
{% endhighlight %}

Не забудем, что хорошей практикой является выбор читаемых названий для ключей. Для этого составим простую схему:
{% highlight C#  %}
public static class Scheme
{
    public static string Users(Guid id) => $"users:{id}";
    public static string Posts(Guid id) => $"posts:{id}";
    public static string Comments(Guid id) => $"comments:{id}";
    public static string Hashtags(Guid postId) => $"posts:{postId}:hashtags";
    public static string LatestPosts() => "posts:latest";
}
{% endhighlight %}

Теперь используем созданный класс в методах обращения к базе данных. Сначала добавим кэширование для получения объектов по идентификатору:
{% highlight C#  %}
public Post GetPost(Guid postId)
{
    if (_cacheProvider.TryGet(Scheme.Posts(postId), out Post cachedPost))
        return cachedPost;
    
    /* Находим нужный пост SELECT запросом к базе данных */

    _cacheProvider.Add(Scheme.Posts(postId), post);

    return post;
}
{% endhighlight %}

Аналогичный код будет и для других сущностей в системе: пользователи, комментарии — все, что имеет идентификатор. При удалении объекта нужно очистить кэш: удаляем все ключи, в которых этот объект может использоваться.

Кэширование ленты новостей можно реализовать с помощью списка. Список хранит только идентификаторы объектов, поэтому при их обновлении не нужно менять список. При каждом создании поста новый айди добавляется в начало списка и производится операция trim, обрезающая ленту до необходимого числа элементов. При удалении постов кэш для ленты сбрасывается.
{% highlight C#  %}
public List<Post> GetLatestPosts(int count = 5)
{
    if (_cacheProvider.TryGetGuidList(Scheme.LatestPosts(), count, out List<Guid> guids))
    {
        var cachedPosts = new List<Post>();
        foreach (var guid in guids)
        {
            if (_cacheProvider.TryGet(Scheme.Posts(guid), out Post cachedPost))
                cachedPosts.Add(cachedPost);
        }
        if (cachedPosts.Count == guids.Count)
            return cachedPosts;
    }
    
    /* Получаем список постов SELECT запросом к базе данных */

    _cacheProvider.AddGuidListAndTrim(Scheme.LatestPosts(), count, posts.Select(p => p.Id).ToArray());
    return posts;
}
{% endhighlight %}

Кэширование хэштегов удобно реализовать через множества, так как для них важен не порядок, а факт наличия для поста.
{% highlight C#  %}
private void AddHashtagsToPost(Post post)
{
    /* Добавление связи между хэштегами и постами в базу данных */

    _cacheProvider.AddSet(Scheme.Hashtags(post.Id), post.Hashtags);
}
    
private List<string> GetHashtags(Guid postId)
{
    if (_cacheProvider.TryGetSet(Scheme.Hashtags(postId), out List<string> cachedHashtags))
        return cachedHashtags;

    /* Выборка хэштегов, соответствующих запросу, с помощью оператора JOIN */
    
    _cacheProvider.AddSet(Scheme.Hashtags(postId), hashtags);
    return hashtags;
}
{% endhighlight %}

# Заключение

Мы познакомились с новой технологией с интересной парадигмой: высокая скорость работы, но большая вероятность потери данных. Основываясь на Redis, мы реализовали простой кэш и теоретически повысили производительность существующего приложения. Тем не менее, ограничение оперативной памяти состоит не только в возможной потере информации, но и в объеме. В случае высоконагруженного приложения оперативной памяти одного сервера может не хватить для поддержки кэширования. Поэтому в следующий раз мы обсудим настройку кластера Redis. 

Пока же вы можете почитать в [блоге на Хабре](https://habrahabr.ru/company/digdes/blog/351002/), как мы использовали другую NoSQL базу данных, которая называется Elasticsearch, для реализации полнотекстового поиска.
