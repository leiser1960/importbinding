# importbinding
## Named Interface Types used as Type Parameters
This is a proposal for the language go, to have a simple mostly golang 1 compatible form of type binding (aka Generics).

## Motivation
There is the official golang draft in https://go.googlesource.com/proposal/+/refs/heads/master/design/go2draft-type-parameters.md.
After careful reading and reading the discussions about this proposal I am somewhat disappointed by it. I am a bit fuzzed about the many brackets and type parameters, which have to be added at several places.

This solution lacks the facinating elegance, of the way the golang team managed to add inheritance to golang 1. Totally different from all other languages having inheritance, but a perfect illustration what inheritance really is, and how inheritance and delegation are interconnected. Simply drop the attribute name to move from delegation to inheritance.

This proposal starts from golang 1 form of inheritance by simply adding a mechanism for binding types at the import of a package. The "generic" package already in the golang 1 standard library my be used nearly unchanged, and can still be used in a polymorphic manor. By adding the type binding on the import the user of a standard package switches from dynamic type checking of his uses to static type checking. In exchange the otherwise necessary type casts my be dropped. Dropping the type bindings and reintroducing the type casts easily allows you to port the package back to golang 1.

## Named Interface Types
golang allows you to give a name to an interface type, the "Stringer" beeing one well known example. But if you look at container/list.go you find seven textual occurances of "interface{}". But as a go proverb say: "Interface nothing says nothing." Seven times nothing still does not say anything. Why not give it a name?
* type Element = interface{}
And replace the seven occurances of "interface{}" by this name. Naming nothing says something. This gets more obvious if we look at golang sync/map.go. Here we have 24 times "interface{}" but we are talking about two different types:
* type Element = interface{}
* type Key = interface{}
After carefully replacing all occurences by either "Key" oder "Elemen" we read for example
* func (m *Map) Store(key Key, value Element)
Only the name of the parameter or return value gives a hint about which of the two types we are talking about. So misunderstanding the api is easy, no compile time typechecking helps keeping things separate.

## Named versus Structural Type Equivalence
Go uses mostly named type equivalence for struct types and structural type for interface types. It is brilliant how much of a difference this makes in comparison for example with the use of interfaces in Java, which are apart from the named type equivalence semantically close.

But for the above example this is not so good. The two types:
* type Element = interface{}
* type Key = interface{}
introduced in sync/map.go still say "nothing" allthough the implementation keeps them completely separat.

To change this I would like to change the type system as follows:
* Treat Named interface types with Named Type Equivalence locally.

More formally this may be implemented by a simple program transformation:
* Replace each named Interface type by a new struct{} type, having exactly the methods required in the interface type.
* compile the modified package file using golang 1 compiler.
* if compilation succeeds we know, that there is no mixing of distinct interface types in the implementation.

Changing to Local Named Type Equivalence clearly breaks the golang 1 compatibilty contract. But following this stricter rule results in a package which remains still valid golang 1 code.

## Type Binding imports

Using the slightly modified example from container/list.go:
<pre>
package main

import "container/list"
import "fmt"

func main() {
	// Create a new list and put some numbers in it.
	l := list.New()
	e4 := l.PushBack(4)
	e1 := l.PushFront(1)
	l.InsertBefore(3, e4)
	l.InsertAfter(2, e1)

	// Iterate through list and print its contents.
	for e := l.Front(); e != nil; e = e.Next() {
    int result = (int)e.Value
		fmt.Println(result)
	}

}
</pre>

What we really want to have hier is a "list of int". And no type cast (which is my addition). How could we express our desire? My suggestion is:
The original line:
* import "container/list"
is augmented to:
* import "container/list" (type int list.Element)
where the type "list.Element" is the type introduced in the above modification of "containter/list.go"
* type Element interface{}

### Implementation by Programm transformation

There are two possible Implementations of the above type binding on import:
* Simply drop it and use golang 1 semantics.
* Apply a programm transformation to the imported golang package replace the declaration of type by a type alias declaration using the type of the import phrase. And using this transformed code under a new and unique package path.

Note the Program is only valid if *both* implementations compile.

In the example the program transformation would as simply as:
* type Element int

The transformed source code would be renamed to:
* container/4711/list.go
(with a new unique string replacing the 4711)
The import statment would be replaced to:
* import "container/4711/list.go"
If the resulting program is a correct golang 1 programm both implementations are considered valid implementations of the type binding.

The first is a polymorphic, the second an monomorphic implementation, the first results in smaller, the second in faster programms, but your mileage may vary.

### What about two lists with different Bindings in a program?

Simply use the "renaming import" Form of the import statement:
* import intlist "container/list.go" (type int intlist.element)
* import doublelist "ontainer/list.go" (type double doublelist.element)

This leads to two transformed modules:
* import intlist "container/4711/list.go"       // from the first example with type Element int
* import doubllist "container/4712/list.go"     // with type Element double

### What about binding more than one type?

The example in "sync/map.go" has two Bindable Types:
* type Key interface{}
* type Element interface{}
So an import clause may bind either or both of these types:
* import "sync" (type string sync.Key, int sync.Element)
The "unbound" Types keep their golang 1 semantics unchanged, but the package as a whole gets instantiated.

### Caveats of the program transformation semantics of import bindings

* The definition requires the types used in the binding import to be exported types.
* The implementation one still requires type casts.
* Both implementations may have subtle differences in behaviour.
* The sync package has too many bindable types.

The Program transformation defined above fails to work if I try to bind "list.Element" with a local not exported type.
This should be easy to overcome using a more general definition of the semantics behind a the type binding import clause.
And of course this should be completely legal.

The same holds for the second problem: Because we know the type binding dropped by the transformation, it should be easy for the compiler to insert the necessary type cast at compile time, according to the dropped bindings.

The behavioural difference is more subtle harder to understand and harder to solve.
Depending on the actual code in the imported package the behaviour of the two implementations may be different. A simple example exemplifies this:
* Let us add at "var int Counter" static Variable to "containter/list.go" and increment it on every call to "list.New()"
* The first implementation counts all "New()" calls of the whole program.
* The second implementation counts the "New()" calls of every import with type binding separately.
Why would you do this? Think of generic container types with persistence. Having a connection pool to the database stored in a variable is a typical necessity here. Having one such variable for every instantiate type may break resource limits and change the transaction behaviour more than you may imagine.

Simply forbidding to have static variables in a package solves the problem. But this would rule out e.g. initialized error Variables which are completely reasonable and since they are effectively constants are completely harmless to replicate. 
What is the "correct" behaviour? Hard to say.

The problem of having more Parameter types than you like is easy to solve:
* "sync/map.go" is one of several files in the package sync, more or less unrelated e.g. with "mutex.go" which contains the interface type Locker. And the above mentioned "sync/map.go" with the two Parameter types "Key" and "Element" waiting to be inserted. Since we always instantiate on import, we get more instantiation than what we wanted. But there is no need to bind types, which you do not use, or explizitly prefer to use with golang 1 semantics. And the compiler is free to chance between the two above defined implementations, even within the same package. And the compiler knows that Locker is only used in mutex.go, but Key and Element are only used in map.go.
