---
layout: post
title: Дай Этому Умереть
---
Без лишних вступлений, познакомимся с моим [DAO](https://ru.wikipedia.org/wiki/Data_Access_Object): 
{% highlight scala %}
object PurchaseOrder extends AsyncQuery {
...
}
{% endhighlight %}
Это действительно объект

*Problem?*

Проблема располагается где-то внутри многоточия.

{% highlight scala %}
def getItems(size: Int) = for {
  result <- runAsync(_.getAll, _.bind().setFetchSize(size))
  elements = result.iterator().toStream
  grouped = elements flatMap dbRead.fromRowL grouped size
} yield grouped toStream
{% endhighlight %}

Где `runAsync` определён в `AsyncQuery` так

{% highlight scala linenos %}
def runAsync(query: Statements ⇒ Future[PreparedStatement],
             finalize: PreparedStatement ⇒ Statement
              ): Future[ResultSet]
{% endhighlight %}

Итак, строка за строкой внутри монады `Future`:

{% highlight scala%}
result <- runAsync(_.getAll, _.bind().setFetchSize(size))
{% endhighlight %}

получает `ResultSet` 
{% highlight scala %}
elements = result.iterator().toStream
{% endhighlight %}
получает `Stream[Row]`

для следующей строки нам будет интересно определение
{% highlight scala %}
case class DBRead[R](fromRow: Row ⇒ Option[R]) extends AnyVal {
  def fromRowL = fromRow andThen (_.toList)
}
{% endhighlight %}
Это маленький объект, который определяет способ необязательно успешной конвертации из строки БД объект данных `R`
`fromRowL` - превращает `Option[R]` на выходе в `List[R]`, давая возможность использовать результат в `flatMap`, опуская безуспешные попытки получить данные

В нашём DAO такой объект явно используется так
{% highlight scala %}
val dbRead = implicitly[DBRead[ListedContract]]
{% endhighlight %}

Итак, третья строчка
{% highlight scala %}
grouped = elements flatMap dbRead.fromRowL grouped size
{% endhighlight %}
Берёт `Stream[Row]` получает для каждого опционально `ListedContract`, выкидывает пропущенные данные, и группирует их по `size` штук.
На выходе генерируем 
{% highlight scala %}
yield grouped toStream
{% endhighlight %}
Который учитывая монаду и тип `grouped` даст нам `Future[Stream[Stream[ListedContract]]]`, т.е. объект, который спустя определённое время вернёт неограниченный список страничек по `size` элементов `ListedContract`, используя для получения очередной страницы внутреннее разбитие в драйвере `cassandra`





