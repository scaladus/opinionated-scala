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

Examples: ...

## Constructing classes with constraints
Sometimes a class must adhere to certain constraints. These constraints should be represented as types as much as possible and reasonable.
When enforcing the constraints at compiletime is not possible or reasonable, don't (only) make the constructor throw exceptions.
Instead, provide a factory-function in the companion-object that tries to construct the class and does not simply return the class itself but a type that indicates if the construction was successfull or not.

Example: ...


# More...
- TABU: f: Option[A] => ???
- TABU: Cakepattern
- Dependency Injection nur als simple Constructor Injection. Keine DI Frameworks, insb. kein runtime DI
- Partial functions vermeiden (z.B. bei Map)
- Usage of Future bzw. future.failed anstatt Future[Either[…]]
- Grenzen des System und deren Schnittstellenformate als ADTs modellieren und eigenes Domainmodell als ADT + Konvertierung der Schnittstellen ADTs in Domainmodell