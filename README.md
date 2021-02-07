# Binding interface types on import 
This is a proposal for the language go, to have a simple mostly golang 1 compatible form of type binding (aka Generics).

## Motivation
There is the official golang draft in
https://go.googlesource.com/proposal/+/refs/heads/master/design/go2draft-type-parameters.md#Very-high-level-overview
After careful reading and reading the discussions about this proposal I am a bit fuzzed. This solution lacks the fascinating elegance, of the way the golang team managed to add inheritance to golang 1. Totally different from all other languages having inheritance, but a perfect illustration what inheritance really is, and how inheritance and delegation are interconnected. Simply drop the attribute name to move from delegation to inheritance.

Here is an attempt to solve the underlying issue in a completely different way, leading to a completely different result an in my opinion simpler code. Go already has a generic type constructor: the interface types. This is of course a form of dynamic polymorphism, i.e. the values contain the concrete types. What is missing in golang escpecially for container types is a way to constrain or parameterize such that the polymorphism is resolver at compile time, instead of runtime.

To go from polymorphic generic types, aka "interface" types in go, to monorphic types you need two things:
* A way to define a type parameter.
* A way to bind the type parameter with a concrete type.

The above mentioned proposal does this by adding positional parameters to struct types and functions, this proposal uses named interface types and type binding on import, which leads to a way simpler and less intrusive addition to the language.

How do we propose to solve the two tasks?
* We use the existing mechanism to name and export a type and reinterpret it as a named type parameter of the package.
* We add a type binding mechanism to the "import" statement.

So basically all you have to change if you want to switch from the runtime interface type polymorphism to compile time polymorphism is adding a type binding declaration in one place, on the "import" statement. We are able to:
* Reuse golang version 1 packages unchanged as they are.
* A simple addition to the import statement is all we syntactially need to change.

## A first example: container/list

One of the most simple examples found in the standard library is the package container/list https://pkg.go.dev/container/list.
In my personal experience I never used linked lists with a polymorphic type, so this is one of the classical examples.

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
		var int result = (int)e.Value
		fmt.Println(result)
	}

}
</pre>

I added an explicitly typed intermediate Variable of type "int" to the original example, which requires a type cast to access the List Value.

Let us dive into the details:
* What is the parameter type in "container/list"?
* What is the "binding" we want to have?
* What are the behavioural changes to the code?

### Named type parameter

Browsing through the source code of: https://go.googlesource.com/go/+/go1.15.8/src/container/list/list.go we find seven occurences of the type "interface{}", all in the same meaning:
<pre>
	type ValueType = interface{}
</pre>
Since "interface nothing says nothing" (https://go-proverbs.github.io/) let us first give nothing at least a name: "Design the architecture, name the components, document the details." We now have named an important design element the type of the component "Element", and at the same time documented an important detail.

Glancing over a second example namely the "Map" type in sync.go: The use of "interface{}" there is used for two different types: "KeyType" and "ValueType" which are two separate and distinct types.

### Type binding import

After we named the parameter type of container/list.go to "ValueType" we are able to use this name to bind the Type in our example. We simply replace the "import" statement with:
<pre>
	import "container/list" type list.ValueType int
</pre>
The rest of the example is left unchanged.

### What does this type binding import mean?

The type binding import has to effects:
* It checks that the binding is valid
* For the rest of the compilation unit the type "list.ValueType" is treated as if defined as a type alias to "int".

So we can now drop the type cast in:
<pre>
		var int result = (int)e.Value
</pre>
Adding a line such as:
<pre>
	l.PushBack("Hello world")
</pre>
fail to compile with: argument of type int expected, but is string.

## Named Interface Types
Golang allows you to give a name to an interface type, the "Stringer" beeing one well known example. In this respect nothing added to the language. But there is an additional constraint necessary for a Interface type to be useable as a type parameter of a package. When binding the type on import, we change from runtime bound types to compile time bound types. There we must be sure there are no runtime type errors in the implementation of the generic package. The "broken list" example illustrates this by modifying list.PushBack() to:

<pre>
func (l *List) Remove(e *Element) ValueType {
	if e.list == l {
		// if e.list == l, l must have been initialized when e was inserted
		// in l or l == nil (e is a zero Element) and l.remove will crash
		l.remove(e)
	}
	return "42"
}
</pre>
This modification breaks the semantics of remove by allways returning a fixed value. But when binding ValueType to "int" instead of "interface{}" this also needs to become a compile time type error in our example main import:
<pre>
	import "container/list" type list.ValueType int
</pre>

To fix this we require an exported named type to implement an additional property:
* Local named type equivalence for named interface types.

### Local named type equivalence for named interface types.

What is needed to enable the compiler to catch the quite obvious type error in the above example, without outruling existing valid code?


### Named versus Structural Type Equivalence
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
