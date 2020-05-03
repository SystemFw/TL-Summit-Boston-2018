---
title: Declarative control flow with fs2 Streams
author: Fabio Labella (SystemFw)
theme: solarized
highlightTheme: solarized-light
revealOptions:
  transition: slide
---


# fs2
Declarative control flow with Streams<!-- .element: class="fragment" -->

Note:
- Remember to poll the audience for background
- This talk is about a Scala purely streaming IO library called fs2
- This talk is really about declarative control flow

---

## About me

![](/github.png)

Note:
- Senior software engineer in the financial industry 
- Open source author, pure FP Scala: 
- fs2 maintainer, maintainer or contrib to http4s, cats-effect, cats, shapeless


---

## cats-effect 

**`IO[A]`** <!-- .element: class="fragment" -->

- <!-- .element: class="fragment" --> Produces one value, fails or never terminates
- <!-- .element: class="fragment" --> *Referentially transparent* (pure)
- <!-- .element: class="fragment" --> Compositional
- <!-- .element: class="fragment" --> Many algebras (monad,...)

----

## Data processing

```scala
def process(in: List[String]): List[String] =
  in
   .filter(s => !s.trim.isEmpty && !s.startsWith("//"))
   .map(_.toUppercase)
   .take(100)
```
<!-- .element: class="fragment" -->

```scala
def p: IO[List[String]] = IO(readListFromFile(f)).map(process)
```
<!-- .element: class="fragment" -->

Note:
- process is compositional
- IO keeps compositionality with effects
- Memory usage

---

## fs2 Streams

**`Stream[F[_], A]`** <!-- .element: class="fragment" -->

-  <!-- .element: class="fragment" --> emits `0...n ` values of type `A`, where `n` can be ∞ 
-  <!-- .element: class="fragment" --> While requesting effects in `F`
-  <!-- .element: class="fragment" --> `F` is normally `IO`

Note:
- also pure, compositional, possesses algebras

----

## Streaming IO!

```scala
def fahrenheitToCelsius(f: Double): Double =
  (f - 32.0) * (5.0/9.0)

def converter: Stream[IO, Unit] =
  io.file.readAll[IO](Paths.get("testdata/fahrenheit.txt"), 4096)
    .through(text.utf8Decode)
    .through(text.lines)
    .filter(s => !s.trim.isEmpty && !s.startsWith("//"))
    .map(line => fahrenheitToCelsius(line.toDouble).toString)
    .intersperse("\n")
    .through(text.utf8Encode)
    .through(io.file.writeAll(Paths.get("testdata/celsius.txt")))
```

---

## Control Flow

Note:
- if you need streaming, fs2 is perfect
- if you don't, is it useful? YES

----

## fs2 Streams

```scala
Stream.emit(1, 2, 3)

Stream.eval { IO(println("hello") }

Stream.repeatEval { IO(println("hello") }
```
<!-- .element: class="fragment" -->

```scala
val action : IO[Unit] = yourStream.compile.drain
```
<!-- .element: class="fragment" -->
```scala
mainAction.unsafeRunSync
```
<!-- .element: class="fragment" -->

Note:
- Compose Streams together
- Run Streams by compiling to IO
- can compose the IOs
- run at the end of the world

----

### List with superpowers: `++`

```scala
List.range(1, 100).take(3) ++ List(21, 22)
// List(1, 2, 3, 21, 22)
```
<!-- .element: class="fragment"  -->


```scala
def put[A](a: A) = IO(println(a))

Stream.repeatEval(put("hello")).take(3) ++ Stream.eval(put("world"))
// hello
// hello
// hello
// world
```
<!-- .element: class="fragment"  -->

----

### List with superpowers: flatMap

```scala
List(1,2,3).flatMap(x => List(x,x))
// List(1, 1, 2, 2, 3, 3)
```
<!-- .element: class="fragment"  -->


```scala
def put[A](a: A) = IO(println(a))

Stream(1, 2, 3).flatMap(x => Stream.repeatEval(put(x)).take(2))
// 1
// 1
// 2
// 2
// 3
// 3
```
<!-- .element: class="fragment"  -->

----

### List with superpowers: zip

```scala
List(1,2,3).zip(List("a", "b", "c"))
// List((1, "a"), (2, "b"), (3, "c"))
```
<!-- .element: class="fragment"  -->


```scala
def put[A](a: A) = IO(println(a))

def printRange = Stream.range(1, 10).evalMap(put)

def seconds = Stream.awakeEvery_[IO](1.second)

seconds.zip(printRange)

// 1
// ...1 second
// 2
// ...1 second
// 3

```
<!-- .element: class="fragment"  -->

----

### Example

```scala
def healthCheck: IO[Boolean] = IO {...}
```
<!-- .element: class="fragment"  -->
```scala
def healthCheck: Stream[IO, Message] = {
  val retryCheck = 
    Stream.retry(healthCheck, 1.second, _ + 1, maxRetries = 5)
    
  val check = 
    retryCheck
      .map(HealthCheckMessage(_))
      .handleError(_ => ErrorMessage)
      
  val repeatedChecks =
    (check ++ Stream.sleep_(1.hour)).repeat
    
 repeatedChecks
}

```
<!-- .element: class="fragment"  -->

Note:
- Compare to just IO?

----

### Declarative control flow

- Create simple single actions in IO <!-- .element: class="fragment" -->
- Use Stream to assemble them <!-- .element: class="fragment" -->
- High level, declarative, composable <!-- .element: class="fragment" -->

----

### Declarative control flow

`IOs` are your _words_, `Streams` are your _sentences_

---

## Concurrency

----

### Concurrency features

- Stream concurrency <!-- .element: class="fragment" -->
- Concurrent coordination and data structures <!-- .element: class="fragment" -->
- Run on thread pools <!-- .element: class="fragment" -->
- Nonblocking <!-- .element: class="fragment" -->
- Resource safe <!-- .element: class="fragment" -->

----

### Why concurrency

- Streams are like logical threads of execution <!-- .element: class="fragment" -->
- Interleaving logical threads allows complex behaviour <!-- .element: class="fragment" -->
- Still declarative and composable <!-- .element: class="fragment" -->
- Pure FP for the real world <!-- .element: class="fragment" -->

Note:
- FP not good for effects, it's actually where it shines


----

### Concurrency

```scala
def healthCheck: Stream[IO, Message] = ???
def kafkaMessages: Stream[IO, Message] = ???
def celsiusConverter: Stream[IO, Unit] = ???
```
<!-- .element: class="fragment"  -->

```scala
def all: Stream[IO, Message] = Stream(
   healthCheck,
   celsiusConverter.drain,
   kafkaMessages
 ).covary[IO].joinUnbounded
 
 all.flatMap { ... }
```
<!-- .element: class="fragment"  -->

Note:
- interruptions, errors
- you can avoid the whole thing failing
- files are closed

----

### Concurrency

```scala
def stopAfter[A](in: Stream[IO, A], f: Duration): Stream[IO, A] = {
```
<!-- .element: class="fragment"  -->
```scala
  def out(s: Signal[IO, Boolean]): Stream[IO, A] = 
    in.interruptWhen(s)
```
<!-- .element: class="fragment"  -->
```scala
  def stop(s: Signal[IO, Boolean]): Stream[IO, Unit] = 
    Stream.sleep_[IO](f) ++ Stream.eval(s.set(true))
```
<!-- .element: class="fragment"  -->
```scala
  Stream.eval(async.signalOf[IO, Boolean](false)).flatMap {
    stopSignal =>
      val stopper = stop(stopSignal)
      val runner = out(stopSignal)
      // runner _what?_ stopper
   ```
<!-- .element: class="fragment"  -->
```scala
      runner.concurrently(stopper)
  }
}
```
<!-- .element: class="fragment"  -->

----

### Concurrency

```scala
def stopAfter[A](f: Duration): Stream[IO, A] => Stream[IO, A] = 
  in => {
   def close(s: Signal[IO, Boolean]): Stream[IO, Unit] = 
     Stream.sleep_[IO](f) ++ Stream.eval(s.set(true))
    
   Stream.eval(signalOf[IO, Boolean](false)).flatMap { end =>
    in.interruptWhen(end).concurrently(close(end)))
  }
}
```

```scala
Stream
 .repeatEval(IO(println("hello")))
 .through(stopAfter(2.seconds))
```
<!-- .element: class="fragment"  -->

----

## Go wild!

---

# FS2 Streams

- For data that's too big to fit in memory <!-- .element: class="fragment" -->
- For control flow that's too hard to fit in one's head <!-- .element: class="fragment" -->

---

# Questions?

- I'm not on Twitter, reach out on Gitter @SystemFw!
- Thanks to @mpilquist and @pchlupacek
