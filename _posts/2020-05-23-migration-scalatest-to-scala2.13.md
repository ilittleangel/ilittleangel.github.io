---
layout: post
title:  "Como migrar Scalatest JUnitRunner a Scala 2.13"
date:   2020-05-23 10:00:00 +0200
categories: scala scalatest junit
---

* [Antecedentes](#antecedentes)
* [Buscando soluciones](#buscando-soluciones)
* [Solución](#solucion)



## Antecedentes

Tengo un proyecto **Scala 2.12** con sus test unitarios en [ScalaTest](#https://www.scalatest.org/) y 
haciendo uso del **JUnit runner** de ScalaTest para poder ejecutarlos con Gradle.

Al verme en la necesidad de migrar el codigo a **Scala 2.13** lo primero que haces despues de adaptar ciertos 
fragmentos deprecados dentro de tu codigo fuente es ejecutar los test unitarios. Pero ... sorpresa! el runner 
no está incluido en la version de ScalaTest para Scala 2.13.


## Buscando soluciones

Para solucionarlo y poder ejecutar los test con Gradle desde la linea de comandos o desde un pipeline de IC
sin quitar la anotación `@RunWith(classOf[JUnitRunner])` de la cabecera de las clases de test, me pongo a buscar 
en internet posibles soluciones aportadas por la comunidad.

Estas van desde:

* crear tu propia tarea Gradle test
* usar **JUnit Jupiter** y **JUnit Platform**
* usar runners de JUnit para Scalatest creados en pequeños proyectos opensource

Pero ninguno me daba lo que quería que básicamente es:

* cuando ejecuto mis test con `gradlew test` obtener el mismo formato y orden de ejecución. Ya que son actores akka 
y en algunos casos necesito orden de ejecución entre los diferentes test.
* Seguir usando **JUnit 4** puesto que desde [ScalaTest](#https://www.scalatest.org/user_guide/using_junit_runner) 
ya nos indican que la anotación `RunWith` tiene que ejecutarse con JUnit 4.
* cambiar lo mínimo el `build.gradle`

Después de buscar mucho por internet y no encontrar ninguna alternativa acorde con mis necesidades, encuentro 
la solución en la documentacion oficial de Scala Test:

[http://doc.scalatest.org/3.0.8/org/scalatest/junit/JUnitRunner.html](#http://doc.scalatest.org/3.0.8/org/scalatest/junit/JUnitRunner.html)


## Solucion

En el enlace proporcionado anteriormente se indica que `JUnitRunner` de ScalaTest está deprecado 
y que uses `org.scalatestplus.junit`. Asi que allá vamos.

### Añadir al `build.gradle`

```groovy
testCompile "org.scalatestplus:scalatestplus-junit_2.13:1.0.0-M2"
```

### Cambiar `ExampleSpec.scala`

#### Antes:

```scala
import org.scalatest.junit.JUnitRunner
import org.scalatest.{BeforeAndAfterAll, Matchers, WordSpec}

@RunWith(classOf[JUnitRunner])
class ExampleSpec extends WordSpec with Matchers with BeforeAndAfterAll
```

#### Después:

```scala
import org.scalatest.BeforeAndAfterAll
import org.scalatest.matchers.should.Matchers
import org.scalatest.wordspec.AnyWordSpec
import org.scalatestplus.junit.JUnitRunner

@RunWith(classOf[JUnitRunner])
class ExampleSpec extends AnyWordSpec with Matchers with BeforeAndAfterAll 
```

> `WordSpec` está deprecado por `AnyWordSpec`  
> El trait `Matchers` ahora se encuentra en `org.scalatest.matchers.should.Matchers`


