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

# Named Interface Types
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

## Local named type equivalence for named interface types.

What is needed to enable the compiler to catch the quite obvious type error in the above example, without outruling existing valid code?
For this we require a named interface type to satisfy the property:
* local named type equivalence

It is defined by the following code transformation:
* Replace the "interface" on the right side by the minimal implementation of the interface type.
* The modified package must remain a valid go program

The minimal implementation of an interface type "T" is defined by:
* replace "interface" by "struct"
* replace a method definition of the form "m(p1, p2, ...) (r1, r2, ...)" in the interface by the corresponding method of the struct type: "func (T r) m(xp1 p1, xp2 p2, ...) (xr1 r1, xr2 r2, ...) { return }"

Why does this help? The language go uses "structural type equivalence" for interface types, whith the effect that the type "interface{}" in spite of "saying nothing" actually is "anything". A variable of this type may contain a value of any possible go type.
The type which actually represents "nothing" in go is the type "struct {}". It is compatible with no other type. You may add methods to this types as required, which does not change the behaviour at all, but helps to implement interfaces with methods.

By replacing "interface{}" by "struct{}" i.e. anything by nothing your generic package e.g. "container/list" obviously get useless. But we now know: The definition of the interface type catches all the methods required, and there is no assignment of a value of a different type to this type, as in our "broken Remove" example above.

Note: if the package contains tests, they typically do no longer compile after replacing the implementation of the interface type with the minimal implementation, as is the case in the "container/list" example. Tests are therefore exempted from the above definition. The file "example_test.go" is in a different package anyway so it does not have this problem.

Note: We think the property "local named type equivalence" is a property which is true for all of the uses of interface types within the go standard library. A go vet command for checking this property would help catch program errors.

# Type Binding imports

We now have the required parameter types of packages, all named interface types which satisfy the "local named type equivalence" are candidates for type binding. In case of simple "interface{}" the names are often missing, as is the case in "container/list" or the Map type in "sync", a problem which is fixed easily. Having the parameters it is time to define how to bind them.

A type binding import is defined by the following exmaples:
<pre>
	import "container/list" type list.ValueType int
</pre>
If you need more than one binding in a compilation unit you have to use the renaming import feature of go:
<pre>
	import intlist "container/list" type intlist.ValueType int
	import stringlist "container/list" type stringlist.ValueType string
</pre>

If the package defines more than one named interface type more than one type binding may be added to an import clause:
<pre>
	import "sync" type sync.Locker myType, sync.MapValue int, sync.MapKey string
</pre>
The example assumes the addition of a "type MapValue interface{}" and "type MapKey interface{}" to the file sync/map.go.

TODO: exact syntax definition
We call the named interface type in the first place of the binding the "bound parameter type" and the second the "binding type". In the "container/list" example list.ValueType is the "bound parameter type" and "int" is the "binding type" and we say "list.ValueType" is bound to "int".

The semantics of the above definition is defined by two program transformations:
* The polymorphic transformation
* The monomorphic transformation

Both transformation must yield a valid go program, for the programm to be a valid program.
Any of the transformations operational behaviour is a valid implementation of the program.
For the secand transformation to work the "binding type" is required to be exported. You may have to first do an additional program transformation to satisfy this reqirement. But only changing a private type to be exported never changes the behaviour of a go program.

## The polymorphic transformation

The polymorphic transformation has two steps:
1. it simply drops the binding statement from the source code.
2. insert a type cast to the binding type wherever a value of the bound type is used.

Open question: For step 2 fails when using the address of a value of the bound type. In the list example:
<pre>
	int * x = &l.Element
</pre>

## The monomorphic transformation

Again we have three steps:
1. copy and rename the imported package
2. replace the type definitions of the parameter types by the bindings from the import, potentially adding the package name
3. add imports as required to make the bound types visible
4. replace the import with the bindings by the imports of the renamed package

The renaming rule is:
* Add the bound types and the binding types (including package path) in the lexical order of the bound types to the pathname.
This rule assures that if two packages contain the same binding (potentially in different order) the resulting package will be the same.

The second step effectively replace the named interface definition by a type alias definition in the renamed package.

Note: After monomorphic transformation taking the address of a variable of the bound type is not a problem. i.e.
<pre>
	int * x = &l.Element
</pre>
will compile just fine, because the type of Element is now the following type alias:
<pre>
	type ElementType int
</pre>

The monomorphic transformation has to be transitiv: Whenever the binding type is a exported named interface type itself, the client of the package may add a binding, which then passes through, and requires more instances of the lower level package as well.

## Compile time errors

After binding a type on import we expect nice error messages in the following cases:
1. the binding type does not implement the bound type
2. there is a type mismatch relative to the binding type in the source.

The two transformation behave differently with respect to the two errors:
* The polymorphic transformation reports error 1 fine, but misses errors of type 2.
* The monomorphic transformation reports errors of type 2 fine, but reports type 1 errors when compiling the renamed packages which is bad.

## Differences in operational behaviour

At first glance we expect no difference in the operational behaviour of the two programms resulting from the two program transformations. By getting rid of the overhead of construction the value of the interface types we only expect a better performance of the second version.
But this is false in general as soon as you get more than one renamed instance of the same package. If the package contains a static variable, we now have two of them in the second transformation, but only one in the first.
Outruling static variables in the packages would solve the problem, but that also would outrule perfectly reasonable use cases such as defining an "error" Variable for the package.
The draft requires further discussion, ideally the two implementations should yield the same functional behaviour. For packages like "container/list" and other well defined types there is no difference in the functional behaviour. Ideally this could be verified at compile time, but i doubt this is possible in general.
The compiler should be free to choose the implementaion yielding the best compromise between compile time, code size and runtime of the program.

## Program transformation is a definition of semantics only

Directly implementing this proposal in the compiler instead of a separate tool chain will improve quality:
* error messages are directly given at the binding statement or the uses
* local named type equivalence is separately testable
* instantiation and renaming is transparently made on compile time or link time
* the compiler or linker may optimize for the best possible implementation compromise

# Discussion

What are the achievements of this proposal?
* Only one additional clause
* Seemless integration
* Reuse of golang 1 container packages without modification (except for additional naming)
* the type bindings are "out of the way"
* only one instantition per package and binding
* it is not "generics"

What are the disadvantages?
* You can not define monomorphic generic types locally within a package
* does not follow the trails of C++ or Java
* it is not called "generics"

## Only one place for the binding

If you carefully look at the "container/list" example you find two places where other approaches require a type parameter:
* When defining a variable of type "list.Element"
* On the function "list.New"

In the canonical example above there is only one place for passing the "int" Parameterbinding is required, namely the call of list.New(). But in a more elaborate example there might be a field in a local struct of type "list.Element" yielding to parameteriziation at several places.
In our approach it is easy to locate the binding: "list" is the name of the bound package, or whatever you used in the import clause.

## Seamless integration

Changing from a golang 1 program which allready uses a generic type monomorphically to a golang 2 programm using compile time monomorphic is a simple as changing exactly one line. Add the type binding to the import. If it compiles You are done.

You may also wish to drop the now redundant type casts (go vet may help locating them) but this is not necessary.
Type two implementation may change the program behaviour, but typically only broken implementations do so.

## Reuse and minimality

Any reasonable defined golang 1 package using a named interface type may be imported with type bindings completely untouched. The lack of naming, when no methods are required is a easy to solve problem, and remains a completely legal golang 1 package.

The "local named type equivalence" helps to improve the quality and uncover code glitches in generic packages.

## Type bindings on import

Adding the type binding to the import statement keeps the function bodies completely uncluttered by type parametrization. Type checking of the compiler yields perfect results, the fixed type bindings do not complicate the local type inference or type checking rules.

## Only one instantiation per package and type binding

If the same type binding is used in several compilation units, their is only one instantiation and one copy of the instatiated code necessary for the complete program.

## It is not generics

Leaving the usual trails of "generic" discussion is a feature. But it clearly follows the spirit of the go language design: Keep it simple and orthogonal and nicely fitted in the language: You want the "container/list" but only for elements of "YourPersonalData"? Add a binding on import. Done.

## You can not define monomorphic generic types locally within a package

I expect this to be a mayor draw back in the academic discussion of the features of this draft: You simply cannot put your favorite generic "max()" function example in a gist and discuss it. It allways requires two packages at least.

But this is a purely academic problem. In any real world scenario I can not see reasonable "generic type" or "generic function" with general usefulness but putting it in a separate package with its own unit test to be a cost penalty at all. It is either generic and of general use, thus in its own package or c&p is the better paradigma.

## does not follow the trails of C++ or Java

Big malus. But again, that part of go's genes anyway.

## it is not called "generics"

Because it is not. It is type binding on package import. It only binds two types, one of them must be an interface type at compile time. The types and functions using the interface type are generic, but this is allready true in golang 1.
