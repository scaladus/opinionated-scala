# Opinionated Scala

## Basics

### Referential Transparency

- Never use null!
- Never use vars in case classes.
- If you use vars anywhere, never leak them outside of a function scope (keep them referentially transparent)

### Other recommendations
- Avoid using a types partial functions `Option.get`, `List.head`, `Map.apply`
- Use partial functions where applicable 
    - ```map { case SomeType() => doSomething() }```  - BAD, map expects a function
    - ```collect { case SomeType() => doSomething() }```  GOOD, collect expects a partial function
- Don't await Futures!
- Be aware that standard `Seq` is mutable, consider using `immutable.List` for small structures or `immutable.Vector` for large sequences
- Try to avoid writing functions on functors, but map the function instead:
    - ```def doSomethingWithOption(input:Option[Int]):Option[Int] // BAD```
    - ```def doSomethingWithOption(input:Int):Int ; myOption.map(doSomethingWithOption)  // GOOD```
    - ```def doSomethingWithList(input:List[Int]):List[Int] // BAD```
    - ```def doSomethingWithList(input:Int):Int ; myList.map(doSomethingWithList)  // GOOD```
- Use a single file with multiple classes only for your data model, i.e. ADTs
- Public methods and vals should have a type annotation

==> [Scapegoat](https://github.com/sksamuel/sbt-scapegoat) and [WartRemover](http://www.wartremover.org/) may help

## Implicits
### Implicit defs
Implicit methods should generally be avoided. Exceptions are:

- Creating a new instance of a custom defined class. In this case the implicit class pattern can be used instead.
- Typelevel programming. That is, the method must have only implicit parameters.
 
Examples:
- BAD: implicit def(s: String): List[Char] = ???
- GOOD: implicit def (s: String): StringOps = new StringOps(s)
- BETTER: implicit case class StringOps(s: String) { … }
- GOOD: implicit def orderingByNumeric[A](implicit ord: Numeric[A]): Ordering[B] = ???

### Implicit method parameters
Implicit method parameters should only be used for passing type class instances and not for passing contexts/configurations.
Making (parts of) arguments implicit because it is tedious to pass them down the call chain is a code smell and there usually exists a way around, e.g. by using the Reader Monad or creating a DSL with the Free Monad.
Providing an implicit argument by programmers mistake must be extremely rare/hard and thus: the more general the implicit parameter the worse. 

Examples:
- BAD: def greetUser(name: UserName)(implicit lang: Language) = ???
- WORSE: def allUsers(repository: UserRepository)(implicit ec: ExecutionContext) = ???
- DOOMED: def getAllUsers(repository: UserRepository)(implicit ec: ExecutionContext, timeoutLimit: Long) = ???
- GOOD: def serialize[A](obj: A)(implicit si: SerializeInstructions[A]) = ???
- BETTER: def serialize[A: SerializeInstructions](obj: A) = ???

## Traits
### Sumtypes / Uniontypes
One recommended use case for traits is the encoding of sumtypes. A sumtype can be represented like that:
sealed trait Shape
	final case class  Circle(radius: Double) extends Shape
	final case class  Rectangle(width: Double, height: Double) extends Shape
	final case class  Cone(radius: Double, height: Double) extends Shape
	final case object Rheinturm extends Shape
	final case object KoelnerDom extends Shape
	
We now have safe pattern matching (compiler warns us if we forget a case) and an easy to understand representation of the different kind of things that are handled by our application.
Don't forget to add the sealed keyword! This is mandatory. Never leave that out. Also note that the trait should not have methods defined (also see [Inheritance](#Inheritance of traits)).

Advanced: What if you don't have control over some or all of the classes (Circle, Rectangle, Cone, ...) but still want to have type-safe transform functionality similiar to pattern matching?
Answer: Use shapeless Coproduct: type Shape = Circle :+: Rectangle :+: Some3rdPartyShape :+: Another3rdPartyShape :+: CNil

### Inheritance of traits
Other than in Java, interface/trait inheritance to provide a commond set of functionality is usually discouraged in Scala.
It is, however, okay to use inheritance under the condition that you are in complete control of the code. That means, you must be able, at any time, to rewrite every single part of any code that could possibly use your trait (including 3rd party code) and you have weighted the benefits vs. costs in comparison with typeclasses.
That means, using traits when writing a library for others is an absolute NOGO. Don't do this.

- Inheritance is okay for ADTs and also in cases like this:
sealed trait LogMessage {
	def msg: String
}
case class InfoMessage(msg: String) extends LogMessage
case class WarnMessage(msg: String) extends LogMessage
case class ErrorMessage(msg: String) extends LogMessage

trait Drawable {
	def asImage: JPG
}
case class Circle(...) extends Drawable {
	override def asImage: JPG = ???
}

Note, however, that typeclasses should be used as soon as the Drawable trait gets more complex or used rather often.

### Typeclasses

### Cakepattern





### Usage of traits for library writers
When writing a library that provides a method that accepts objects by the client (thus not under your control), never force inheritance usage.
That is, don't make the client inherit from a trait, abstract class or anything else. Instead, always use typeclasses when the library needs certain functionality that should be provided by its client.

Examples:

- LoggingLibrary BAD:
trait StringSerializeable {
    def serialized: String
}
def log[A <: StringSerializeable](obj: A) = writeToLogFile( obj.serialized )
 
- LoggingLibrary GOOD:
trait StringSerializer {
    def serialize[A](obj: A): String
}
def log[A: StringSerializer](obj: A) = writeToLogFile( implicitly[StringSerializer].serialize(obj) )

## Classes
### Case classes
case classes are „java-DTOs“. They must never contain state or interact with other components in any way.
The only thing that is allowed are „convenience“ methods that use only data from inside of the case class (and no additional data).
The reason here is, that Plain-Data classes should only ever be coupled with classes that they existentially depend on.
In the example: Urls can exists in a world without anything like users, so don't couple Urls to the fact that there exists a thing like a user.

Examples:
final case class Url(host: String, port: Int) {
    def urlString = s"$host:$port" //Okay!
    def isHttps = host.take(5) == "https" //"Okay"!
    
    //Returns http(s)://${user.name}@${user.password}:$password@$host:$uri
    def withUserCredentials(user: User) = ??? //Bad! Avoid this. Use implicit class instead
}

### Abstract Classes
Abstract classes are similiar to traits. Rule of thumb: always use trait unless you have specific reasons for an abstract class.
In some situations it is convenient to have constructor parameters in ADTs for types with fixed values and in such cases traits add boilerplate.
This is an example of an okayish usage of abstract classes:

sealed abstract class ServiceError(msg: String)
case object Timeout extends ServiceError("The Request to the Service took too long")
case object InvalidResponse extends ServiceError("Could not parse the Service response")

Using a sealed trait it would look like that:
sealed trait ServiceError {
  def msg: String
}
case object Timeout extends ServiceError {
  override val msg: String = "The Request to the Service took too long"
}
case object InvalidResponse extends ServiceError {
  override val msg: String = "Could not parse the Service response"
}

An important difference between abstract classes and traits is multiple inheritance which is possible with traits but not with abstract classes.
This does not matter though, because we discourage multiple inheritance (and often even inheritance itself) anyways most of the time.

### Constructing classes with constraints
Sometimes a class must adhere to certain constraints. These constraints should be represented as types as much as possible and reasonable.
When enforcing the constraints at compiletime is not possible or reasonable, don't (only) make the constructor throw exceptions.
Instead, provide a factory-function in the companion-object that tries to construct the class and does not simply return the class itself but a type that indicates if the construction was successfull or not.

Examples:
- BAD:
final case class PositiveInt(value: Int) {
    throw new IllegalArgumentException("PositiveInt must have positive value")
}
- Good:
final case class PositiveInt private(value: Int) {
    throw new IllegalArgumentException("PositiveInt must have positive value")
}
object PositiveInt {
    type Error = String
    def fromInt(value: Int): Either[Error, PositiveInt] =
        if(value >= 0)  Right(PositiveInt(value))
        else            Left("PositiveInt must have positive value")
}
- But:
    - PositiveInt(-50) still compiles
    - .copy(value = -50) still compiles
    - Making PositiveInt a normal class throws away all convenience (pattern matching, hashcode/equals, copy, ...)

- Crazy workaround:
object Numbers {
  type Error = String
  sealed abstract case class PositiveInt private[Numbers](value: Int) {
    require(value >= 0)
  }
  object PositiveInt {
    def fromInt(value: Int): Either[Error, PositiveInt] =
      if (n >= 0) Right(new PositiveInt(value){})
      else Left("PositiveInt must have positive value")
  }
}

PositiveInt.fromInt(42) //Some(PositiveInt(42))
PositiveInt.fromInt(-42) //None
new PositiveInt(-42) //Compile error
PositiveInt.fromInt(42).copy(value = -42) //Compile error
PositiveInt.fromInt(42) match { //Compiles and works!
	case PositiveInt(value) => ??? 
}

Summarized: choose you the least bad of them...

## Pattern matching
Pattern matching is a powerful language feature and makes a lot of things easier. Because of its power it also comes with some dangers.
For that reason, pattern matching should only used when needed and there are some basic rules to be followed to make its usage safe.

### Simple comparisons
Pattern matching can be used to check conditions and to extract values. If only simple conditions are checked, use if/else instead of pattern matching.
Example:
birthyear match {
	case 1989 => println("this is the best year")
	case 2017 => println("You are to young to read this anways")
}
Instead use
if(birthyear == 1989) println("this is the best year")
else if(birthyear == 2017) println("You are to young to read this anways")

The reason is that pattern matching should ideally indicate that all cases are handled which is not the case here. Thus, if you don't need the power of pattern matching, don't use it.
It's not per se better than using if/else. One advantage of if/else is, that one can immediatly see that there is no else clause, thus some cases will most likely not be caught.
 
If you need the power of pattern matching e.g. because you wan't to extract values from case classes and match on structure, take care to make pattern matching safe.

### Case classes of sealed traits
Pattern matching is safe if you match on case classes of sealed traits without using guards. This rule recursively applies for matching things within case classes.
For the built in primives like String, Boolean, Int, ... it is also safe to match on their values even though they are not case classes because the compiler will warn you regardless.
If you match on non-case classes or use guards, you must always provide a default case that applies when all other matches failed.
- Examples

This one is okay because the compiler will warn (or fail to compile with compiler flag)
"hello" match { case "really hello" => ??? }

sealed trait Fruit
case class Apple(weight: Int) extends Fruit
case class Banana(length: Float) extends Fruit

This one is also good. Missing classes will create compiler warnings/errors
someFruit match {
	case Apple(weight) => ???
	case Banana(length) => ???
}

Even the next one is fine, because messing up with their inner values will throw warnings/errors because they are primitives
someFruit match {
	case Apple(20) => ???
	case Banana(length) => ???
}
Will result in compiler saying: Warning:(39, 13) match may not be exhaustive. It would fail on the following input: Apple((x: Int forSome x not in 20))

Now a guard is used. The guard checks if the weight is over 20. This will NOT trigger compiler warnings/errors and thus can lead to unsafe pattern matchings, resulting in MatchErrors
someFruit match {
	case Apple(weight) if weight > 20 => ???
	case Banana(length) => ???
}

TODO: Add examples for matching in non-case class, non sealed traits and using the default case




## Exceptions
Exceptions have multiple purposes.
One of them is to indicate technical errors (NullPointer) or even fatal technical errors (OutOfMemory).
Another one is to indicate domain errors to which we count both custom defined exceptions but also built in ones like NumberFormatException.
In Scala you should not trigger or manually throw exceptions for domain errors. There are alternatives for different cases.
If you use a function that parses a string to a number which can throw an exception, use Try(iCanThrowExceptions()) to work with exceptions. Don't mix Try up with try/catch syntax!
Try does not catch fatal exceptions like StackOverflow or OutOfMemory which is almost always a good thing because you can't handle them anyways except loging the error and gracefully shutdown at the toplevel of your application.
If you a are implementing a function where you are tempted to throw an exception, don't throw an exception but instead return a (sum)type like Either which can contain the normal return value or the domain error(s).

The same is valid for built-in Scala types like Future that internally use exceptions.
Futures can be successful or failed, but they should only be failed due to technical errors (e.g. see above).
If there are other than technical reasons that a value cannot be retrieved, use a sum(type) like Either.
Example:
- BAD
def asyncGetUser(userId): Future[User] = ???
val userResult = Await(asyncGetUser(someId), 20.seconds)
userResult match {
    case Success(user) => println("wuh, user found!")
    case Failure(ResponseTimeoutException()) => println("Timed out!")
    case Failure(UserNotFound(userId)) => println("Could not find user")
    case Failure(UserParseError(jsonError)) => println("Could find user but not parse it")
    case Failure(throwable) => println("Uh oh, something went wrong. Maybe the Thread was interrupted OR maybe we did forget to handle a domain error? Who knows...")
}
- GOOD
def asyncGetUser(userId): Future[Either[DomainError, User]] = ???
val userResult = Await(asyncGetUser(someId), 20.seconds)
userResult match {
    //Technically everything went well. We can now check for domain errors.
    case Success(domainResult) => domainResult match {
        case Right(user) => println("wuh, user found!")
        case Left(ResponseTimeoutException) => println("Timed out!")
        case Left(UserNotFound(userId)) => println("Could not find user")
        case Left(UserParseError(jsonError)) => println("Could find user but not parse it")
    }
    //Only technical error
    case Failure(throwable) => println("Uh oh, something went wrong. Maybe the Thread was interrupted but it can't be a domain error that we forgot to handle!")
}

## General opinions
### General architecture
Grenzen des System und deren Schnittstellenformate als ADTs modellieren und eigenes Domainmodell als ADT + Konvertierung der Schnittstellen ADTs in Domainmodell
### Dependency injection
Dependency Injection nur als simple Constructor Injection. Keine DI Frameworks, insb. kein runtime DI

### Library usage
* Use Scala library over Java 
* Use strongly typed, functional libraries ( [Typelevel](http://www.typelevel.org) is a good starting point)
* Do you really need to write your own library? Guideline
  - Business-specific? OK 
  - Not specific to your business: Google it!
  
### Keep it simple (stupid)! 
- Do you really need Akka or Spark?
- Do you really need a macro for that? 

### Separation of concerns 
- Do one thing and do it well!
  
