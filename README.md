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

Advanced: What if you don't have control over some or all of the classes (Circle, Rectangle, Cone, ...) but still want to have type-safe pattern matching?
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
In the example: Urls can exists in a world without anything like users, so don't couple Urls to their existence.

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
This does not matter though, because we consider multiple inheritance (and even inheritance itself) as useless to extremely dangerous anyways most of the time.

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

## Exceptions
Usage of Future bzw. future.failed anstatt Future[Either[…]]

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
  
