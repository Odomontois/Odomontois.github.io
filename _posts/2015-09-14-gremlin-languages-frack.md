---
layout: post
title: gremlin languages frack
categories: []
tags: [fracking_cylons]
published: True

---

Перед тем, как я начал писать какой-то код, мультилингвальная поддержка и Orient DB, и TinkerPop казалась всеобъемлющей. 

На странице [языков, на которых можно говорить с Orient][orient-langs] значатся Scala, Groovy и Clojure, однако при переходе на [страницу поддержки scala][scala-orient] мы видим, что нам предлагают пользоваться Java API и кидают ссылки на вопросы с http://stackoverflow.com/ , касающиеся интеграции Java в Scala

[Gremlin-scala][gremlin-scala] тоже существует, однако ведётся, по всей видимости, единственным разработчиком, и [ветка 2.х][orient-2.x], совместимая с доступным от основных разработчиков драйвером для Tinkerpop-2.4 обновлялась больше года назад. А [Orient driver для TinkerPop 3][orient-tp3], написанный всё тем же новозеландцем, обозначен как

> this is just a proof of concept, nowhere ready to use in production

Итак, касательно и core-driver, и gremlin в scala нам предлагают использовать Java API.

Если для Core я успел накатать уже какой-то свой фреймворк для загрузки case class ов посредством [shapeless][shapeless], который способен взять какой-то  такой класс
{% highlight scala %}
case class Airport(
        code: String
   ,    city_code: Option[String]
   ,    country_code: Option[String]
   ,    coordinates: Option[Coordinates @+ Embedded]
   ,    name: String
   ,    name_translations: Seq[AirportL10n] @+ LinkWith[L10nE]
) extends Encoded
{% endhighlight %}
и слепить для него функции выгрузки из JSON (спасибо play-json) и загрузки в orient.

То разница в gremlin запросах на java и groovy [приблизительно такая][groovy-vs-java-gremlin]:

####Groovy
{% highlight groovy %}
g.v(1).out.filter{it.name.startsWith('j')}
{% endhighlight %}

####Java
{% highlight java %}
new GremlinPipeline(g.getVertex(1)).out().filter(new PipeFunction<Vertex,Boolean>() { 
  public Boolean compute(Vertex v) {
    return v.getProperty("name").startsWith("j");
  }
}
{% endhighlight %}

Подумав слегка, я начал искать способы поюзать groovy в своём проекте.

[sbt-groovy][sbt-groovy] существует и даже компилит что-то, однако низкая активность и популярность проекта вызывает подозрения

Действительно, groovy классы, созданные в том же проекте фактически не могут быть использованы в scala коде. Компиляция Groovy просто запускается после scala. И единственная возможность, которую я обнаружил (не впервые) - выделить groovy в отдельный субпроект.

Зато прописанная вручную строчка
{% highlight scala %}
libraryDependencies += "org.codehaus.groovy" % "groovy-all" % "2.4.4"
{% endhighlight %}
Установила groovy на свежую версию.

##Результаты

Столь желанному `gremlin-scala` скорее всего, в проекте нет места
Оригинальный `gremlin-groovy` даёт не только красивые запросы, но и стабильность, возможность вставки загугленного кода и  повторного использования скриптов в функциях.

Выделение скриптов в отдельный подпроект, в общем-то обосновано и без проблем с порядком компиляции.


[orient-langs]: http://orientdb.com/docs/2.1/Programming-Language-Bindings.html
[scala-orient]: http://orientdb.com/docs/last/Scala-Language.html
[gremlin-scala]:https://github.com/mpollmeier/gremlin-scala/
[orient-2.x]:https://github.com/mpollmeier/gremlin-scala/tree/2.x
[orient-tp3]:https://github.com/mpollmeier/orientdb-gremlin
[shapeless]:https://gitter.im/milessabin/shapeless
[groovy-vs-java-gremlin]:https://github.com/tinkerpop/gremlin/wiki/JVM-Language-Implementations
[sbt-groovy]:https://github.com/fupelaqu/sbt-groovy

