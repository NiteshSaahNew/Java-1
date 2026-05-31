# Java Lambda Expressions: In-Depth Tutorial with JVM Internals

## 1. Introduction
Java **Lambda Expressions**, introduced in **Java 8**, enable
functional-style programming by allowing behavior to be treated as data.
They make code more concise, readable, and expressive.
<img width="1293" height="472" alt="image" src="https://github.com/user-attachments/assets/43107984-3154-45de-9d1e-927f6e9291d8" />

### Key Characteristics
-   Anonymous (no name)
-   Can accept parameters
-   Can return values
-   Can be passed as arguments or assigned to variables
-   Used with **functional interfaces**
------------------------------------------------------------------------

## 2. Lambda Expression Syntax

### Basic Syntax

``` java
(parameters) -> expression
```

or

``` java
(parameters) -> { statements; }
```

### Examples

``` java
// Without lambda (anonymous class)
Runnable r1 = new Runnable() {
    @Override
    public void run() {
        System.out.println("Hello World");
    }
};

// With lambda
Runnable r2 = () -> System.out.println("Hello World");
```

------------------------------------------------------------------------

## 3. Functional Interfaces

A **functional interface** is an interface that contains **exactly one
abstract method**. Lambda expressions provide the implementation for
this method.

``` java
@FunctionalInterface
interface Calculator {
    int operate(int a, int b);
}

Calculator add = (a, b) -> a + b;
System.out.println(add.operate(5, 3));  // Output: 8
```

### Common Built-in Functional Interfaces

  |Interface             |Method                |Example                       | Default Method                       |
  |--------------------- |--------------------- |------------------------------|--------------------------------------|
  |`Predicate<T>`        |`boolean test(T t)`   |`(x) -> x > 0`                |Has and, or, negate, but not compose.|
  |`Function<T,R>`       |`R apply(T t)`        |`(x) -> x * 2`                |**andThen**(Function) and **compose**(Function)	Core interface for transformations. Allows **chaining forward** (andThen) and **backward** (compose).|
  |`Consumer<T>`         |`void accept(T t)`    |`x -> System.out.println(x)`      | Has andThen, but **not** **compose**.|
  |`Supplier<T>`         |`T get()`             |`() -> new Random()`              |**No** andThen or compose     |
  |`BiFunction<T,U,R>`   |`R apply(T t, U u)`   |`(a,b) -> a+b`                    |Has **no** andThen or compose.|
  |`BinaryOperator<T>`   |`R apply(T t, U u)`   |`(a,b) -> a+b`                    | Inherits from BiFunction, so **no** andThen() or compose().|
  |`UnaryOperator <T>`   |`R apply(T t)`        |`(x) -> x > 0`                |Inherits **andThen**(Function) and **compose**(Function) from Function	.|

-----------------------------------------------------------------------------------------------------------------------
.

**🔑 Key Takeaways**

**Only** **Function** and **UnaryOperator** support **both** andThen and compose for **chaining.**.

     **andThen  →** executes current function first, then the next.
    
     **compose →** executes the argument function first, then the current one.

This distinction is critical when building pipelines:

 **Use andThen()** for **forward chaining** (left-to-right).
    
 **Use compose()** for **reverse chaining** (right-to-left).

-----------------------------------------------------------------------------------------------------------------------
## 4. Lambda Syntax Variations

``` java
// No parameters
() -> System.out.println("Hello");

// One parameter
x -> x * x;

// Multiple parameters
(a, b) -> a + b;

// With block body
(a, b) -> {
    int result = a + b;
    return result;
};
```

------------------------------------------------------------------------

## 5. Variable Capture and Scope

Lambdas can access: - Local variables (**must be effectively final**) -
Instance variables - Static variables

``` java
int factor = 10; // effectively final
Function<Integer, Integer> multiply = x -> x * factor;
```

------------------------------------------------------------------------

## 6. Method References

Method references provide a concise way to refer to existing methods.

``` java
// Lambda
list.forEach(x -> System.out.println(x));

// Method reference
list.forEach(System.out::println);
```

### Types of Method References

1.  Static method: `ClassName::staticMethod`
2.  Instance method of a particular object: `instance::method`
3.  Instance method of an arbitrary object: `ClassName::method`
4.  Constructor reference: `ClassName::new`

------------------------------------------------------------------------

## 7. How the JVM Treats Lambda Expressions

### 7.1 Not Implemented as Anonymous Inner Classes

Lambdas **do not generate separate `.class` files**. Instead, they rely
on the `invokedynamic` instruction introduced in Java 7.

### 7.2 Compilation Process

For the following lambda:

``` java
Runnable r = () -> System.out.println("Hello");
```

The Java compiler (`javac`) performs:

1.  **Creates a private static synthetic method**:

    ``` java
    private static void lambda$main$0() {
        System.out.println("Hello");
    }
    ```

2.  **Replaces the lambda with an `invokedynamic` instruction** in the
    bytecode.

3.  **Links to `LambdaMetafactory`** in `java.lang.invoke`.

### 7.3 Role of `invokedynamic`

`invokedynamic` defers lambda creation until runtime, enabling dynamic
linkage and JVM optimizations.

Example bytecode (`javap -c -p`):

``` text
0: invokedynamic #0,  0  // InvokeDynamic #0:run:()Ljava/lang/Runnable;
```

### 7.4 LambdaMetafactory

`java.lang.invoke.LambdaMetafactory` is responsible for: - Creating the
functional interface instance - Linking the lambda body to the interface
method - Enabling caching and JIT optimizations

### 7.5 Runtime Representation

-   **Stateless lambdas** may be cached and reused.
-   **Stateful lambdas** (capturing variables) typically create new
    instances.
-   Since Java 15, lambdas are implemented using **hidden classes**.

``` java
Runnable r1 = () -> System.out.println("Hello");
Runnable r2 = () -> System.out.println("Hello");
System.out.println(r1 == r2); // Often true for stateless lambdas
```

------------------------------------------------------------------------

## 8. Anonymous Class vs Lambda: JVM Perspective

  ----------------------------------------------------------------------------
  |Feature         |Anonymous Class              |Lambda Expression            |
  |--------------- |---------------------------- |-----------------------------|
  |Class File      |Separate `.class` generated  |No separate `.class`         |
  |`this` reference |Refers to anonymous class    |Refers to enclosing class    |                    
  |Performance     |Slightly heavier             |More optimized               |
  |Bytecode        | `new` instruction           | `invokedynamic`             |
  |Serialization   |Different semantics          |More flexible                |
  --------------------------------------------------------------------------

### Example Demonstrating `this`

``` java
class Test {
    void demonstrate() {
        Runnable anon = new Runnable() {
            @Override
            public void run() {
                System.out.println(this.getClass());
            }
        };

        Runnable lambda = () -> {
            System.out.println(this.getClass());
        };

        anon.run();   // Prints anonymous class
        lambda.run(); // Prints Test class
    }
}
```

------------------------------------------------------------------------

## 9. Performance Considerations

### Advantages

-   Lazy linkage via `invokedynamic`
-   Better JIT optimizations and inlining
-   Reduced memory footprint
-   Instance caching for stateless lambdas

------------------------------------------------------------------------

## 10. Serialization of Lambdas

Lambdas are serializable only if the target functional interface extends
`Serializable`.

``` java
@FunctionalInterface
interface SerializableRunnable extends Runnable, java.io.Serializable {}
```

Serialization relies on captured state rather than a generated class.

------------------------------------------------------------------------

## 11. Real-World Examples

### Sorting a List

``` java
List<String> names = Arrays.asList("Raj", "Amit", "Zara");
names.sort((a, b) -> a.compareTo(b));
```

### Using Streams

``` java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
numbers.stream()
       .filter(n -> n % 2 == 0)
       .map(n -> n * n)
       .forEach(System.out::println);
```

------------------------------------------------------------------------

## 12. Summary

  -----------------------------------------------------------------------
  Concept                       Key Point
  ----------------------------- -----------------------------------------
  Lambda Expression             Anonymous function implementing a
                                functional interface

  Functional Interface          Exactly one abstract method

  JVM Implementation            Uses `invokedynamic` and
                                `LambdaMetafactory`

  Class Generation              No separate `.class` file

  Variable Capture              Only effectively final variables

  Performance                   Efficient and JIT-optimizable
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 13. JVM Execution Flow

    Java Source Code
            ↓
          javac
            ↓
      Bytecode with invokedynamic
            ↓
    Bootstrap via LambdaMetafactory
            ↓
    Runtime creation of functional interface instance
            ↓
    JIT optimizations and execution

------------------------------------------------------------------------
<img width="1293" height="472" alt="image" src="https://github.com/user-attachments/assets/43107984-3154-45de-9d1e-927f6e9291d8" />

**Step 1: Java Source Code**g
The developer writes a lambda expression. Notice that no anonymous inner class is explicitly defined here; it is just a simple block of code.

**Step 2: Compilation**
The javac compiler avoids creating a physical .class file for the lambda. Instead, it compiles the lambda block into a private method and generates an invokedynamic instruction.

**Step 3: Invokedynamic Call Site**
At runtime, the JVM reaches the invokedynamic instruction. This delegates the creation logic to a Bootstrap Method (BSM) provided by the JDK.

**Step 4: LambdaMetaFactory**
The Bootstrap Method calls LambdaMetaFactory. This factory dynamically spins up a synthetic class in memory using the ASM bytecode framework, linking it to the private method created in **Step 2.**

**Step 5: Generated Lambda Instance**
The JVM instantiates this lightweight synthetic class (e.g., RunnableLambda.class). It is now a fully functional object.

**Summary of Benefits**
By delaying creation to runtime, Java keeps jar files small (No New .class file on disk), captures variables efficiently, and allows the JIT compiler to heavily optimize the dynamic invocation.

------------------------------------------------------------------------

## Conclusion

Java Lambda Expressions provide a modern, efficient, and expressive way
to write behavior-driven code. Understanding how the JVM treats
lambdas---through `invokedynamic` and `LambdaMetafactory`---helps
developers write more performant and maintainable applications.
