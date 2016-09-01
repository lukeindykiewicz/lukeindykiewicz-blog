+++
date = "2016-09-01T12:22:31+02:00"
draft = false
title = "free monads in business application"

+++

# What

This post describes [Letter Shop API](https://github.com/lukeindykiewicz/letter-shop-api) implementation which uses Free Monads, exactly Free from [Cats](https://github.com/typelevel/cats)

# tl;dr

[show me the code](https://github.com/lukeindykiewicz/letter-shop-free-scala)

# Why

There are some nice materials about free monads on the net, but I always wanted to see how such concepts, applied to regular business applications, look like. How difficult or cumbersome it is to implement some business use case using a particular concept, e.g. free monads. 

Letter Shop is my way to make business application implementation comparable. I have explained what it is in this [post](http://lukeindykiewicz.com/blog/letter-shop/)

# How

Free monads separate algorithm definition from its implementation. We need a program - the code that describes steps of an algorithm, the compiler - the code that interprets/runs some other code for algorithm steps and data on which our algorithm works.

## ADT as your internal API

We have to define all operations that are possible from business logic point of view as ADT. 

ADT in Scala is described as a trait and case classes/objects implementing it. Each class/object describes one operation. Let's look at `GetCart` in [StorageADT](https://github.com/lukeindykiewicz/letter-shop-free-scala/blob/master/src/main/scala/domain.scala#L27):

{{< highlight scala >}}
case class GetCart(cartId: String) extends StorageADT[Cart]
{{</ highlight >}}

`GetCart` case class represents one operation (getting cart of course). It takes one parameter (`cartId`) and returns cart, which is represented by `StotrageADT` parameterized with `Cart` type.

The whole ADT has a generic parameter which is specified in each class/object implementing it - this parameter is for return type of each operation.

Defining ADT in the first place looks like some huge amount of code that has to be written just to make free monads work, but in the end it occurs that it makes you think which operations are really needed in your code and it also defines your api. All this operations are your internal api. 

You can of course have different ADTs for different parts of application, like: storage, users, validation, etc.

## Helpers aka boilerplate

After defining our plain old ADT we need to lift each class/object from ADT to `Free`. In the easiest example it could be done simply by `Free.liftF`, but this works when we have only one ADT in our application. I doubt it's true for any business application, so if we don't write small and simple dsl we need to complicate it a little bit.

There is a special method in `Free` called `inject`, which in fact calls `Free.liftF`. It doesn't lift our ADT, but it lifts value of type, which is injected as defined in a call to `Free.inject`. Let's see how does it look in [code](https://github.com/lukeindykiewicz/letter-shop-free-scala/blob/master/src/main/scala/free.scala):

{{< highlight scala >}}
 class StorageConstructors[F[_]](implicit I: Inject[StorageADT, F]) {
    def getCart(cartId: String): Free[F, Cart] =
      Free.inject[StorageADT, F](GetCart(cartId))
 }
{{</ highlight >}}

`getCart` is called a smart constructor. This is something that makes our case class `GetCart` usable in context of Free when we have more than one ADT. Return type is simply `Free` with `F` and return type of given operation - `Cart` in the above example.

So coming back to lifting, it "repacks" (`injects`) from `StorageADT` type into `F` type and the result is lifted into `Free`. This `F` type in our case is `LetterShop`, which is `Coproduct` of our ADTs. I will come to `Coproduct` in a while.

`StorageConstructors` companion object defines implicit conversion from `Inject` to our `StorageConstructors`, but at the same time `Inject` parameter is automatically created on demand (look into `Programs` [trait](https://github.com/lukeindykiewicz/letter-shop-free-scala/blob/master/src/main/scala/Programs.scala), each program needs implicit `StorageConstructors`). 
	
We have also defined type alias 

{{< highlight scala >}}
type LetterShop[A] = Coproduct[StorageADT, PriceADT, A]
{{</ highlight >}}

`LetterShop` is a `Coproduct` of ADTs that we have used in application, that is `StorageADT` and `PriceADT`. `Coproduct` is another type from `Cats` which allows us to combine two ADTs and then use it with `Free`. If we have more than two ADTs we need to combine one coproduct of two ADTs into another coproduct with third ADT and so on.

There is a lot of code that has to be written to satisfy the compiler here and it's only preparation for writing programs and compilers - the core part of our application.

## UC's as programs

`Free` is using terms like programs and compilers to talk about code that implements in abstract way the algorithm and code that really executes it. Use case (user story, name it as you wish) is a part of business logic in our code that gathers all the operations needed to achieve some goal. Typically it's equivalent of one request from frontend to our backend code. Let's see how does it look in [code](https://github.com/lukeindykiewicz/letter-shop-free-scala/blob/master/src/main/scala/Programs.scala)

{{< highlight scala >}}
def getCartProgram(cartId: String)(
    implicit S: StorageConstructors[LetterShop]): Free[LetterShop, Cart] = {
  import S._
  for {
    cart <- getCart(cartId)
  } yield cart
}
{{</ highlight >}}

`getCartProgram` takes all needed params in the first parameter list (`cartId` in our case) and implicit `StorageConstructors[LetterShop]`. Compiler sees the need for `StorageConstructors` and finds implicit conversion in `StorageConstructors` companion object, which in turn needs `Inject` type. We don't have such value in our code, but `cats.free.Inject.leftInjectInstance` can be found. That's why everything compiles and all the types are in place.

All steps of the program are coded using for comprehension. Return type for each program is also `Free`. That makes it perfect for composition, because all our business logic returns `Free`. Let's look at `checkoutCartProgram`:

{{< highlight scala >}}
def checkoutCartProgram(cartId: String, promoCode: Option[String])(
    implicit S: StorageConstructors[LetterShop],
    P: PromoConstructors[LetterShop]): Free[LetterShop, Checkout] = {
  import S._, P._
  val uuid = UUID.randomUUID.toString
  for {
    p    <- checkCartProgram(cartId, promoCode)
    cart <- getCart(cartId)
    _    <- addToReceipts(cartId, ReceiptHistory(p.price, uuid, cart.letters))
    _    <- removeCart(cartId)
  } yield Checkout(p.price, uuid)
}
{{</ highlight >}}

This UC/program has more steps and as we can see first step just calls other program. That is followed by some other steps. Of course such composition is also possible without using `Free`, but having one type container for every UC makes it a lot easier.

Program only generates steps of the algorithm. When it comes to the execution ... we need another code - compiler.

## Compilers

We always wanted to write applications in which parts of them can be changed to some other parts without fixing the whole application. There are many ways to do this (interfaces, ports and adapters architecture, etc.). Free monads give us one more option. It's nice because it's really interchangeable without bothering that our business logic will fail (at least from business logic "algorithm" point of view).

So let's write a compiler. In case of free monads from `Cats` compiler is a `cats.arrow.NaturalTransformation` (aliased as `~>`) from ADT, that we have created, to some other type. It could be `Future`, but the simplest one is `Id`. Compiler has to be written for defined ADTs.

{{< highlight scala >}}
def storageCompiler: StorageADT ~> Id =
  new (StorageADT ~> Id) {
  ... 
  def apply[A](fa: StorageADT[A]): Id[A] = fa match {
    case GetCart(cartId) => Cart(carts.getOrElse(cartId, ""))
    case AddToCart(cartId, letters) =>
      carts += (cartId -> letters)
      ()
  ...
  }
}
{{</ highlight >}}

The whole code is [here](https://github.com/lukeindykiewicz/letter-shop-free-scala/blob/master/src/main/scala/Compilers.scala). This example is using `TrieMap` and stores everything in memory just for simplicity. It could also go to database or connect to some service - that depends on compiler implementation.

`storageCompiler` is a compiler for `StorageADT` and it defines natural transformation from `StorageADT` to `Id`. To make it work we have to implement apply method which should know what to do with every element of our ADT. That's why we use simple pattern matching. Of course compiler will tell us if we forget to implement one of case classes/objects as they are under sealed trait.

The other compiler from our example, `priceCompiler`, delegates execution to `PromotionService`.

The last step is to combine two compilers to be able to run all our programs - it's simple:

{{< highlight scala >}}
def compiler: LetterShop ~> Id = storageCompiler or priceCompiler
{{</ highlight >}}

I use this compiler in [Routes](https://github.com/lukeindykiewicz/letter-shop-free-scala/blob/master/src/main/scala/main.scala#L36) to be able to run programs.

{{< highlight scala >}}
lazy val getCart = get {
  path(Segment) { cartId =>
    complete(getCartProgram(cartId).foldMap(cmp))
  }
}
{{</ highlight >}}

To run the program we need to call the program with needed params and `foldMap` it with chosen compiler.

# Materials

The best material on practical usage of Free Monads in my opinion is [Free](http://typelevel.org/cats/tut/freemonad.html) description from Typelevel

# Summary

This blog post describes how to use `Free` from `Cats` to implement exemplary business application. Of course it's not a silver bullet to take and use in every application, but it looks interesting. Especially when flexibility in choosing the way our code is executed is needed. It also gives our code some structure with one main type to which business logic is implemented. It's useful in many ways not to have a code in which every UC returns different type (i.e. Option, Future, Either, etc.). For sure a lot of additional code has to be written to achieve this. The decision is yours.

# Comments

If you have some opinions about `Free` and/or this text feel free to make PR, create issues and comment on twitter.
