---
layout: post
title:  "Elasticsearch и автокомплит"
date:   2017-10-28 20:43:30 +0300
categories: technologies
tags: [technologies, autocomplete, elasticsearch]
image:
  feature: 2017-10-28/elasticsearch-autocomplete.png
  teaser: 2017-10-28/elasticsearch-autocomplete-teaser.jpg
---

Пару недель назад мы начали изучать возможности, которые предоставляет поисковый движок Elasticsearch. Тогда мы познакомились с технологией suggest'ов, то есть подсказок в ответ на опечатки. Сегодня мы разберемся с родственным им понятием - автодополнением (autocomplete). Автокомплит подсказывает возможное продолжение строки по мере ее ввода пользователем. Без этой фичи трудно представить себе поисковую систему.

## Задача
Если вы не знакомы с Elasticsearch, советую прочитать мой [первый пост](https://alexeykalina.github.io/technologies/elasticsearch-suggesters.html) об этой технологии. В том же посте мы решили нашу первую задачу о предоставлении подсказок на примере датасета *Jeopardy*. Напомню, что это известная американская телевикторина на подобии *Своей игры*. Сегодня мы продолжим работать с этим набором данных, используя результаты, которых достигли в прошлый раз.
 
Продолжая идею сервиса для работы с вопросами Jeopardy, можно поставить себе новую задачу. Теперь благодаря suggest'ам пользователь может корректировать свои запросы в соответствии с существующими категориями. Следующим шагом станет добавление функциональности автодополнения, благодаря которому пользователь значительно ускорит ввод своего запроса.

## Completion Suggester
Для реализации автокомплита в Elasticsearch в дополнение к опечаточным suggester'ам существует еще один - [Completion Suggester](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-suggesters-completion.html). Он был добавлен для решения именно этой проблемы и имеет ряд преимуществ перед другими фичами системы:
1. Главное условие в работе автодополнения - это скорость. Если варианты подсказок не успевают появляться до того, как пользователь закончит печатать, их смысл абсолютно пропадает. Для хранения и поиска подсказок такого типа, ES использует *конечный преобразователь* (FST). Он представим в виде направленного графа, где дугами являются буквы, а полный путь содержит искомое слово. Такая структура данных позволяет находить suggest'ы с огромной скоростью (с бенчмарками можно ознакомиться на [официальном блоге Elasticsearch](https://www.elastic.co/blog/you-complete-me)).
2. При полнотекстовом поиске релевантность результатов определяется метрикой *TF-IDF* (то есть, совпадения, встретившиеся в меньшем числе документов, будут более приоритетны). Для автодополнения такой сценарий не подходит. Поэтому в completion suggester'е наиболее распространенные в индексе выражения будут более приоритетными в выдаче подсказок.
3. Еще одна фича этого suggester'а заключается в возможности сопоставлять несколько возможных вариантов ввода одному документу. Благодаря этому подсказки, например, могут учитывать разный порядок слов.

Похоже, completion suggester делает именно то, что нам нужно. Теперь перейдем к решению нашей задачи.

## Первая реализация
Для того, чтобы показать, какие данные будут использоваться для автодополнения, в Elasticsearch есть специальный тип - *completion*. Добавим в наш маппинг поле *category_suggest* с дефолтным анализатором:

{% highlight json %}
{
    "category_suggest": {
        "type": "completion",
        "analyzer": "autocomplete"
    }
}
{% endhighlight %}

Теперь при индексировании нам нужно класть в это поле данные, которые мы будем дополнять. При этом есть возможность индексировать в этом поле несколько вариантов и устанавливать для каждого свое значение *weight*, тем самым влияя на порядок результатов в выдаче.

Мы пойдем по простому пути и будем помещать в это поле ту же строку, что и в поле *category*. После индексации датасета сделаем тестовый поисковый запрос в поисках возможных продолжений строки *musica* (допустим, пользователя интересуют музыкальные категории). Для этого мы определяем новый suggest с именем *completion-suggest*, в котором указываем искомый текст, поле, по которому ведем поиск и количество результатов. Также будем использовать параметр *_source*, в котором отмечаются поля, которые будут предоставлены в ответе.

{% highlight json %}
{
  "_source": "category_suggest",
  "suggest": {
    "completion_suggest": {
        "text": "musica",
        "completion": {
            "field": "category_suggest",
            "size": 5
        }
    }
  }
}
{% endhighlight %}

Ответ выглядит следующим образом (часть полей и результатов пропущена):

{% highlight json %}
{
    "suggest": {
        "category_suggest": [
            {
                "text": "musica",
                "options": [
                    {
                        "text": "musical a to z",
                        "_score": 1,
                        "_source": {
                            "category_suggest": "musical a to z"
                        }
                    },
                    {
                        "text": "musical a to z",
                        "_score": 1,
                        "_source": {
                            "category_suggest": "musical a to z"
                        }
                    },
                    {
                        "text": "musical a to z",
                        "_score": 1,
                        "_source": {
                            "category_suggest": "musical a to z"
                        }
                    }
                ]
            }
        ]
    }
}
{% endhighlight %}

Очевидно, что у этого подхода есть недостатки. Об этом подробнее в следующем разделе.

## Неприятное обновление
Как вы видите по результатам предыдущего запроса, completion suggester не проводит дедупликацию подсказок и выводит столько повторяющихся результатов, в скольких документах он их встретил. Что интересно, в Elasticsearch версий 2.* это поведение отличалось и дедупликация проводилась. Создатели ES объясняют такие перемены тем, что они решили сделать completion suggester более документо-ориентированным.

Однако, в нашей задаче нет необходимости знать, в каких именно документах встретилась данная категория. Это изменение многими было воспринято весьма отрицательно. Для решения этой проблемы один из разработчиков, для которого отсутствие дедупликации является критичным в его приложении, написал плагин, который добавляет еще один тип поля, использующий старый suggester. Почитать о проблеме и в том числе об этом решении можно в [issue](https://github.com/elastic/elasticsearch/issues/22912).

Мы же воспользуемся более официальным способом обхода этой проблемы. Ребята из поддержки Elasticsearch предлагают использовать отдельный индекс со значениями для автокомплита и обрабатывать дубликаты самостоятельно в работе с такими сценариями как у нас. Кстати говоря, в новых версиях 6.x должна быть добавлена настройка с возможностью дедупликации.

## Добавим уникальности
Создадим новый индекс autocomplete с очень простым маппингом:

{% highlight json %}
{
  "properties": {
    "category_suggest": {
        "type": "completion"
    }
  }
}
{% endhighlight %}

Теперь при индексации в уникальный id документа будем также прописывать название категории, как следствие в индексе не будет дубликатов. Посмотрим на результаты запроса:

{% highlight json %}
{
    "suggest": {
        "category_suggest": [
            {
                "text": "musica",
                "options": [
                    {
                        "text": "musical a to z",
                        "_score": 1,
                        "_source": {
                            "category_suggest": "musical a to z"
                        }
                    },
                    {
                        "text": "musical adjectives",
                        "_score": 1,
                        "_source": {
                            "category_suggest": "musical adjectives"
                        }
                    },
                    {
                        "text": "musical architecture",
                        "_score": 1,
                        "_source": {
                            "category_suggest": "musical architecture"
                        }
                    }
                ]
            }
        ]
    }
}
{% endhighlight %}

На этот раз все результаты уникальны. Проверим их в действии. В моем небольшом примере также используются результаты прошлого поста и выводятся подсказки об опечатках. Для выбора подсказки от автодополнения в приложении используются клавиши с номером строки suggest'а.

![autocomplete-suggest]({{ site.baseurl }}/assets/img/2017-10-28/autocomplete_suggest1.gif "autocomplete-suggest")

У этого варианта есть достаточно серьезная проблема. Одним из преимуществ completion suggester'а озвучивалась подходящая релевантность, учитывающая число документов, содержащих строку запроса. Так как мы насильно создаем уникальные документы, все подсказки будут иметь одинаковую релевантность. Как итог, suggester предлагает результаты в алфавитном порядке (так как  в роли ключа мы указали эти же названия категорий).

## Раздадим веса
Настало время вспомнить, что у completion полей есть свойство *weight*. Так как мы хотим видеть среди подсказок наиболее часто встречающиеся в документах категории, будем обновлять вес необходимого поля, инкрементируя его, вместо перезаписи документа. Для этого при обработке каждого документа при индексировании будем использовать следующий запрос:

{% highlight json %}
{
    "script" : "ctx._source.category_suggest.weight += 1",
    "upsert" : {
        "category_suggest" : {
            "input": "concrete category name",
            "weight" : 1
        }
    }
}
{% endhighlight %}

Директива *upsert* выполняется в случае, если документа с таким *id* нет, создавая его и устанавливая вес подсказки в значение 1. Иначе выполняется операция *script*, которая увеличивает вес на единицу. Посмотрим на окончательный вариант примера:

![autocomplete-suggest]({{ site.baseurl }}/assets/img/2017-10-28/autocomplete_suggest2.gif "autocomplete-suggest")

## Заключение
Сегодня мы познакомились с еще одной классной фичой, предоставляемой Elasticsearch. Однако, несмотря на то, что этот движок имеет достаточно мощные возможности для управления автодополнением, в некоторых задачах это бывает не очень удобно. Тем не менее, создатели базы данных обещают решить эти проблемы в скором времени. А наше изучение Elasticsearch только начинается и впереди еще много чего интересного!