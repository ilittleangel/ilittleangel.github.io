---
layout: post
title:  "Introducción a Akka"
date:   2020-11-05 10:00:00 +0200
categories: scala akka
---

# Introducción a Akka

* [¿Que es Akka?](#que-es-akka)
* [¿Que es un actor?](#que-es-un-actor)
* [Implementación de un actor](#implementación-de-un-actor)
* [Creación de actores](#creación-de-actores)
* [Comunicación entre actores](#comunicación-entre-actores)
* [Unit testing de actores Akka](#unit-testing-de-actores-akka)
* [Ejercicio: Crear un actor que modifique su estado interno](#ejercicio-crear-un-actor-que-modifique-su-estado-interno)


## ¿Que es Akka?

La definición oficial es:

>Akka is a toolkit for building highly concurrent, distributed
>and fault tolerant message-driven applications on the JVM.

Es decir, Akka es un conjunto de herramientas que permiten el desarrollo de aplicaciones
que hacen uso de la concurrencia y que además proporciona alta disponibilidad y resiliencia.

Se encarga de gestionar las tareas más complejas en un sistema concurrente, es decir, el manejo,
creación y sincronización de multiples Threads. Simplifica la gestión de la concurrencia.

Akka está basado en el [**_Actor Model_**](https://en.wikipedia.org/wiki/Actor_model). Esto es:
- Todo son actores.
- Los actores se comunican entre ellos mediante el envío de mensajes.
- En un actor se asegura el principio llamado **_Single-Thread Illusion_**, es decir, en cada instancia 
  de un actor solo corre un thread y cada mensaje que llega se procesa de uno en uno.


## ¿Que es un actor?

* Es un **_objeto_** que solo recibe mensajes y hace algo dependiendo del tipo de mensaje. Nada más!
  No depende de nada externo al propio actor. Desacoplamiento.
* Para poder recibir mensajes el actor tiene una **_dirección única_**.
* Los mensajes recibidos se encolan en un buzón llamado **_mailbox_**.
* Los mensajes se van procesando uno a uno y van saliendo del mailbox.
* A La interpretación del mensaje que le llega se le llama **_behaviour_**.
* Los actores suelen tener un **_estado_**. Esto puede ser una variable que cambia según el mensaje recibido.
* Dentro de un actor se cumple el principio **_Single-Thread Illusion_**, gracias a lo anterior.
* Los actores reciben mensajes y cuando los manejan pueden:
    - almacenar información (estado)
    - crear nuevos actores (hijos)
    - mandar mensajes a si mismo o a otros actores
    - cambiar el behaviour para manejar los mensajes siguientes


## Implementación de un actor

* Para implementar un actor hay que crear una clase que extienda de `Actor`.
* Esto nos obliga a sobreescribir el método `receive()` encargado de manejar los mensajes,
  es decir, nuestro **_protocolo de mensajes_**.
* Una buena practica es usar un companion object para declarar los tipos de mensajes
* También usaremos el companion para crear el método de factoría `Props` para crear instancias de actores.

La definición más sencilla de un actor seria algo así:

```scala
import akka.actor.Actor

class MyActor extends Actor {

  override def receive: Receive = {
    case "hola" => println("adios")
    case "adios" => println("hasta pronto")
  }
}
```

Pero lo normal es tipificar los mensajes con case classes o case objects y definir nuestro método `props()`:

```scala
import akka.actor.{Actor, Props}

class MyActor extends Actor {
  import MyActor._

  override def receive: Receive = {
    case HazAlgo(accion) => println("Trabajando en.. " + accion)
    case QueHaces => println("Estoy trabajando..")
  }
}

object MyActor {
  case class HazAlgo(accion: String)
  case object QueHaces

  def props: Props = Props(new MyActor)
}
```


## Creación de actores

* Para crear actores necesitamos un `ActorSystem`
    - es el sistema que permita la organización y gestión de actores
    - tiene todos los métodos necesarios para la gestión de actores
    - usamos el método factoría `system.actorOf`

> Muy importante a tener en cuenta es que no creamos una instancia de nuestro actor
mediante `new MyActor` sino con el metodo `system.actorOf`. Esto es porque nunca 
interactuamos directamente con el actor, es decir, con el objeto creado a partir
de la clase `MyActor`, sino con una **referencia** a la instancia de nuestro actor.
Esta referencia al actor se llama `ActorRef`. Se hace para proteger al propio actor
detrás de esta referencia, de forma que no podamos desde fuera modificar el estado
interno del actor.

* Solo podremos comunicarnos con el actor usando un `ActorRef`.
* También podemos crear actores hijos, estos serán creados desde un actor padre.

```scala
val system: ActorSystem = ActorSystem("MySystem")

val myActor: ActorRef = system.actorOf(MyActor.props)
```


## Comunicación entre actores

### tell (`!`)

* Para mandarle un mensaje a un actor se usa el patrón `tell`.
* Se necesita una referencia a un actor `ActorRef`.
* El envío de mensajes es asíncrono (fire & forget). El hilo de ejecución no espera ninguna 
  respuesta y continua con la siguiente instrucción.
 
Como usarlo:

```scala
actor ! Mensaje
actor.tell(Mensaje)
```

Ejemplo:

Dentro de tu aplicación quieres que el actor haga algo, pues:
- creas una referencia a tu actor usando el método de factoría
- usando el patron `tell` mandas un mensaje al actor
- el actor recibe el mensaje e invoca al método `receive()` donde se evalúa que tipo de mensaje es
  ejecutando su bloque de codigo asociado

```scala
val myActor = system.actorOf(MyActor.props)

myActor ! HazAlgo("cocinar")
```

En la salida estándar tendremos:

```
$> "Trabajando en.. cocinar"
```

Tambien podemos enviarle:

```scala
myActor ! QueHaces
```

En la salida estándar tendremos:

```
$> "Estoy trabajando.."
```

### ask (`?`)

* ¿Que pasa cuando quieres interactuar con los actores desde fuera?
* Si envías un mensaje a un actor y esperas una respuesta no puedes enviar el mensaje 
  con `!` pues la respuesta del actor iria a un buzón llamado `dead-letters`.
* Para poder recibir una respuesta tienes que preguntar al actor mediante el patron `ask`.
* Esto devuelve un `Future` con la respuesta esperada. Por tanto podemos desde fuera pausar 
  la ejecución del hilo hasta obtener la respuesta.
* También podemos definir un `timeout`. Si no obtenemos la respuesta pasado un tiempo se lanza
  un `AskTimeoutException`.

Como usarlo:

```scala
actor ? Mensaje
actor.ask(Mensaje)
```


## Unit testing de actores Akka

Otra cosa de la que disponemos en Akka es una completa librería de testing llamada _**akka-testkit**_.

Referencia doc oficial: [akka-testkit](https://doc.akka.io/docs/akka/current/testing.html)

Esto para una parte II.


## Ejercicio: Crear un actor que modifique su estado interno

`MyActor.scala`
```scala
import akka.actor.{Actor, Props}

class MyActor extends Actor {
  import MyActor._

  private var count: Int = 0

  override def receive: Receive = {
    case AddNumber(number) =>
      count += number

    case Subtract(number) =>
      count -= number

    case GetCurrentCount =>
      sender() ! count
  }
}

object MyActor {
  case class AddNumber(n: Int)
  case class Subtract(n: Int)
  case object GetCurrentCount

  def props: Props = Props(new MyActor)
}
```

`MyApp.scala`
```scala
import akka.actor.{ActorRef, ActorSystem}
import akka.pattern.ask
import akka.util.Timeout
import MyActor._

import scala.concurrent.Await
import scala.concurrent.duration._

object MyApp extends App {

  // timeout necesario para el patron ask
  implicit val timeout: Timeout = Timeout(1.second)

  val system: ActorSystem = ActorSystem("MySystem")

  val myActor: ActorRef = system.actorOf(MyActor.props)

  myActor ! AddNumber(17)
  val response = (myActor ? GetCurrentCount).mapTo[Int]
  val currentCount = Await.result(response, timeout.duration)
  println(currentCount)

  myActor ! Subtract(22)
  val newResponse = (myActor ? GetCurrentCount).mapTo[Int]
  val newCurrentCount = Await.result(newResponse, timeout.duration)
  println(currentCount)
}
```
