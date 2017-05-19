# Opinionated Scala

## Implicit defs
Implicit methods should generally be avoided. Exceptions are:

- Creating a new instance of a custom defined class. In this case the implicit class pattern can be used instead.
- Typelevel programming. That is, the method must have only implicit parameters.
 
Examples:
- BAD: implicit def(s: String): List[Char] = ???
- GOOD: implicit def (s: String): StringOps = new StringOps(s)
- BETTER: implicit case class StringOps(s: String) { … }
- GOOD: implicit def orderingByNumeric[A](implicit ord: Numeric[A]): Ordering[B] = ???

## Implicit method parameters
Implicit method parameters should only be used for passing type class instances and not for passing contexts/configurations.
Making (parts of) arguments implicit because it is tedious to pass them down the call chain is a code smell and there usually exists a way around, e.g. by using the Reader Monad or creating a DSL with the Free Monad.
Providing an implicit argument by programmers mistake must be extremely rare/hard and thus: the more general the implicit parameter the worse. 

Examples:
- BAD: def greetUser(name: UserName)(implicit lang: Language) = ???
- WORSE: def allUsers(repository: UserRepository)(implicit ec: ExecutionContext) = ???
- DOOMED: def getAllUsers(repository: UserRepository)(implicit ec: ExecutionContext, timeoutLimit: Long) = ???
- GOOD: serialize[A](obj: A)(implicit si: SerializeInstructions[A]) = ???
- BETTER: serialize[A: SerializeInstructions](obj: A) = ???

## Usage of traits
### General use cases
- (G)ADTs...
- Inheritance
- Typeclasses
- ...?

### Abstract Classes
???

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

## Case classes
case classes are „java-DTOs“. They must never contain state or interact with other components in any way.
The only thing that is allowed are „convenience“ methods that use only data from inside of the case class (and no additional data).
The reason here is, that Plain-Data classes should only ever be coupled with classes that they existentially depend on.
In the example: Urls can exists in a world without anything like users, so don't couple Urls to their existence.

Examples:
case class Url(host: String, port: Int) {
    def urlString = s"$host:$port" //Okay!
    def isHttps = host.take(5) == "https" //"Okay"!
    
    //Returns http(s)://${user.name}@${user.password}:$password@$host:$uri
    def withUserCredentials(user: User) = ??? //Bad! Avoid this. Use implicit class instead
}

## Constructing classes with constraints
Sometimes a class must adhere to certain constraints. These constraints should be represented as types as much as possible and reasonable.
When enforcing the constraints at compiletime is not possible or reasonable, don't (only) make the constructor throw exceptions.
Instead, provide a factory-function in the companion-object that tries to construct the class and does not simply return the class itself but a type that indicates if the construction was successfull or not.

Examples:
- BAD:
case class PositiveInt(value: Int) {
    throw new IllegalArgumentException("PositiveInt must have positive value")
}
- Good:
case class PositiveInt private(value: Int) {
    throw new IllegalArgumentException("PositiveInt must have positive value")
}
object PositiveInt {
    type Error = String
    def fromDouble(value: Int): Either[Error, PositiveInt] =
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
  sealed abstract case class PositiveInt private[Numbers](toInt: Int) {
    require(toInt >= 0)
  }
  object PositiveInt {
    def apply(n: Int): Either[Error, PositiveInt] =
      if (n >= 0) Right(new PositiveInt(n){})
      else Left("PositiveInt must have positive value")
  }
}

import Numbers.PositiveInt

PositiveInt(5) //Some(PositiveInt(5))
PositiveInt(-13) //None

- Summarized: choose you the least bad of them...

# More...
- TABU: f: Option[A] => ???
- TABU: Cakepattern
- Dependency Injection nur als simple Constructor Injection. Keine DI Frameworks, insb. kein runtime DI
- Partial functions vermeiden (z.B. bei Map)
- Usage of Future bzw. future.failed anstatt Future[Either[…]]
- Grenzen des System und deren Schnittstellenformate als ADTs modellieren und eigenes Domainmodell als ADT + Konvertierung der Schnittstellen ADTs in Domainmodell