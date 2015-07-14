---
layout: post
title: Пьеса и Лицедеи
tags: 
  - scala
  - akka
  - play
  - play-iteratee
---

В [Play](https://www.playframework.com), как известно, [всегда есть место для **Actor**а](https://www.playframework.com/documentation/2.4.x/ScalaAkka)

Это я и решил проверить.

Никакой особенной бизнес-логики, вот старенький сервис

{% highlight scala %}
object Service extends Controller {
//...
  def getSome(n: Int) = Action.async {
      Enumerator.enumerate(1 to n) run
        Iteratee.fold(Stream.empty[Int])(_.#::(_)) map
        (Json.toJson(_)) map
        (Ok(_))
    }
//...
}
{% endhighlight %}

Возвращает он вот такой простой список:

![список]({{ site.url }}/public/screens/07.2015/localhost_9000_some_100.png)

Попробуем попинать этот убывающий список через [**Actor**а](http://doc.akka.io/docs/akka/snapshot/scala/actors.html), слепим вот такого простого актёришку
{% highlight scala %}
class SomeGetter(limit: Int) extends Actor {
  def end: Receive = {
    case GetNext => sender() ! None
  }

  def get(i: Int): Receive = {
    case GetNext =>
      sender() ! Some(i)
      context become (if (i < limit) get(i + 1) else end)
  }
  val receive = get(0)
}

object SomeGetter {
  case object GetNext
  def props(i: Int) = Props(classOf[SomeGetter], i)
}
{% endhighlight %}
 
Как видим, Actor умеет принимать только сообщение `GetNext` и имеет две группы состояний:

-  `get` получающий очередное число и сдвигающее индекс на следующее
-   `end` устанавливающийся `get` в конце и возвращающий всегда `Ничто`

Таким образом свежий актор генерирует неотрицательные числа до определённого предела, затем впадает в отказ

Адаптируем наш контроллер для работы с `akka`:
{% highlight scala %}
@Singleton
class Service @Inject()(system: ActorSystem) extends Controller {
//...
}
{% endhighlight %}

В изменим также и определение `getSome`. Мы всё ещё пользуемся встроенной семантикой [play.iteratees](https://www.playframework.com/documentation/2.4.x/Iteratees):

{% highlight scala %}
def getSome(n: Int) = Action.async {
  val someActor = system actorOf SomeGetter.props(n)
  Enumerator.unfoldM(())(_ => for {
    i <- someActor ? GetNext map (_.asInstanceOf[Option[Int]])
  } yield for {
      x <- i
    } yield ((), x)) run collectStream map (s => Ok(upickle.write(s)))
  }
{% endhighlight %}
Наш Enumerator теперь не тупо копирует коллекцию, он всерьёз взаимодействует c Actor ом для получения каждого объекта.
[unfoldM](https://www.playframework.com/documentation/2.4.x/api/scala/index.html#play.api.libs.iteratee.Enumerator$@unfold[S,E](s:S)(f:S=>Option[(S,E)])(implicitec:scala.concurrent.ExecutionContext):play.api.libs.iteratee.Enumerator[E]) - логичная аллюзия на Haskellевский [unfoldr](http://hackage.haskell.org/package/base-4.8.0.0/docs/Data-List.html#v:unfoldr)

Суть `unfoldr` - в поэлементной генерации коллекции. Имеется функция определяющая очередной элемент в данном состоянии.
На каждый шаг для текующего состояния она возвращает одно из двух:

-  `Some((Следующее состояние, Следующий элемент))`(Just ...)
-  `None`(`Nothing`), если итерация закончена 

В нашем случае всё состояние сокрыто в самом акторе, поэтому мы передаём из состояния в состояние `() : Unit`
Буква `M` в конце метода означает, что функция универсализирована для работы в монаде. В случае Play, эта монада вполне конкретна - это [`Future`](http://www.scala-lang.org/api/current/index.html#scala.concurrent.Future), т.е. следующее состояние и элемент мы возвращаем асинхронно.

`Enumerator` сам по себе представляет такой асинхронный генератор, а [Iteratee](https://www.playframework.com/documentation/2.4.x/api/scala/index.html#play.api.libs.iteratee.Iteratee), объявленная выше как 
{% highlight scala %}
   def collectStream[T]: Iteratee[T, Stream[T]] = 
    Iteratee.fold(Stream.empty[T])(_.#::(_))
{% endhighlight %} 
и поедающая Enumerator в методе `run` будет терпеливо асинхронно собирать элемент за элементом в наш перевёрнутый список.

Учитывая, что наш `unfoldM` не имеет состояния, мы можем воспользоваться его специализированной версией `generateM`
{% highlight scala %}
Enumerator.generateM(
 for {
      i <- someActor ? GetNext map (_.asInstanceOf[Option[Int]])
    } yield for {
        x <- i
      } yield x)
{% endhighlight %}
Или, учитывая отстутсвие всяких действий внутри `Future` и `Option` просто
{% highlight scala %}
Enumerator.generateM(someActor ? GetNext map (_.asInstanceOf[Option[Int]]))
{% endhighlight %}

Локальный сервер демонстрирует легковесность и самой `akka` и `play.iteratee` собрать список из 100К чисел таким невнятным способом - дело долей секунды.

Пора наложить асинхронные лапы на кассандру.
