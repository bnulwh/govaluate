govaluate
====

This library contains quite a lot of functionality, this document is meant to be formal documentation on the operators and features of it.
Some of this documentation may duplicate what's in README.md, but should never conflict.

# Types

This library only officially deals with four types; `float64`, `bool`, `string`, and arrays.

All numeric literals, with or without a radix, will be converted to `float64` for evaluation. For instance; in practice, there is no difference between the literals "1.0" and "1", they both end up as `float64`. This matters to users because if you intend to return numeric values from your expressions, then the returned value will be `float64`, not any other numeric type.

Any string _literal_ (not parameter) which is interpretable as a date will be converted to a `float64` representation of that date's unix time. Any `time.Time` parameters will not be operable with these date literals; such parameters will need to use the `time.Time.Unix()` method to get a numeric representation.

Arrays are untyped, and can be mixed-type. Internally they're all just `interface{}`. Only two operators can interact with arrays, `IN` and `,`. All other operators will refuse to operate on arrays.

# Operators

## Modifiers

### Addition, concatenation `+`

If either left or right sides of the `+` operator are a `string`, then this operator will perform string concatenation and return that result. If neither are string, then both must be numeric, and this will return a numeric result.

Any other case is invalid.

### Arithmetic `-` `*` `/` `**` `%`

`**` refers to "take to the power of". For instance, `3 ** 4` == 81.

* _Left side_: numeric
* _Right side_: numeric
* _Returns_: numeric

### Bitwise shifts, masks `>>` `<<` `|` `&` `^`

All of these operators convert their `float64` left and right sides to `int64`, perform their operation, and then convert back.
Given how this library assumes numeric are represented (as `float64`), it is unlikely that this behavior will change, even though it may cause havoc with extremely large or small numbers.

* _Left side_: numeric
* _Right side_: numeric
* _Returns_: numeric

### Negation `-`

Prefix only. This can never have a left-hand value.

* _Right side_: numeric
* _Returns_: numeric

### Inversion `!`

Prefix only. This can never have a left-hand value.

* _Right side_: bool
* _Returns_: bool

### Bitwise NOT `~`

Prefix only. This can never have a left-hand value.

* _Right side_: numeric
* _Returns_: numeric

## Logical Operators

Unlike others, this library does not currently do short-circuit evaluation. The author isn't opposed to the idea, but hasn't implemented it. All logical operators evaluate both sides in their entirety before deciding what to do.

### Logical AND/OR `&&` `||`

* _Left side_: bool
* _Right side_: bool
* _Returns_: bool

### Ternary true `?`

Checks if the left side is `true`. If so, returns the right side. If the left side is `false`, returns `nil`.
In practice, this is commonly used with the other ternary operator.

* _Left side_: bool
* _Right side_: Any type.
* _Returns_: Right side or `nil`

### Ternary false `:`

Checks if the left side is `nil`. If so, returns the right side. If the left side is non-nil, returns the left side.
In practice, this is commonly used with the other ternary operator.

* _Left side_: Any type.
* _Right side_: Any type.
* _Returns_: Right side or `nil`

### Null coalescence `??`

Similar to the C# operator. If the left value is non-nil, it returns that. If not, then the right-value is returned.

* _Left side_: Any type.
* _Right side_: Any type.
* _Returns_: No specific type - whichever is passed to it.

## Comparators

### Numeric comparators `>` `<` `>=` `<=`

* _Left side_: numeric
* _Right side_: numeric
* _Returns_: bool

### Regex comparators `=~` `!~`

These use go's standard `regexp` flavor of regex. The left side is expected to be the candidate string, the right side is the pattern. `=~` returns whether or not the candidate string matches the regex pattern given on the right. `!~` is the inverted version of the same logic.

* _Left side_: string
* _Right side_: string
* _Returns_: bool

## Arrays

### Separator `,`

The separator, always paired with parenthesis, creates arrays. It must always have both a left and right-hand value, so for instance `(, 0)` and `(0,)` are invalid uses of it.

Again, this should always be used with parenthesis; like `(1, 2, 3, 4)`.

### Membership `IN`

The only operator with a text name, this operator checks the right-hand side array to see if it contains a value that is equal to the left-side value.
Equality is determined by the use of the `==` operator, and this library doesn't check types between the values. Any two values, when cast to `interface{}`, and can still be checked for equality with `==` will act as expected.

Note that you can use a parameter for the array, but it must be an `[]interface{}`.

* _Left side_: Any type.
* _Right side_: array
* _Returns_: bool

# Parameters

Parameters must be passed in every time the expression is evaluated. Parameters can be of any type, but will not cause errors unless actually used in an erroneous way. There is no difference in behavior for any of the above operators for parameters - they are type checked when used.

All `int` and `float` values of any width will be converted to `float64` before use.

At no point is the parameter structure, or any value thereof, modified by this library.

## Alternates to maps

The default form of parameters as a map may not serve your use case. You may have parameters in some other structure, you may want to change the no-parameter-found behavior, or maybe even just have some debugging print statements invoked when a parameter is accessed.

To do this, define a type that implements the `govaluate.Parameters` interface. When you want to evaluate, instead call `EvaluableExpression.Eval` and pass your parameter structure.

# Functions

During expression parsing (_not_ evaluation), a map of functions can be given to `govaluate.NewEvaluableExpressionWithFunctions` (the lengthiest and finest of function names). The resultant expression will be able to invoke those functions during evaluation. Once parsed, an expression cannot have functions added or removed - a new expression will need to be created if you want to change the functions, or behavior of said functions.

Functions always take the form `<name>(<parameters>)`, including parens. Functions can have an empty list of parameters, like `<name>()`, but still must have parens.

If the expression contains something that looks like it ought to be a function (such as `foo()`), but no such function was given to it, it will error on parsing.

Functions must be of type `map[string]govaluate.ExpressionFunction`. `ExpressionFunction`, for brevity, has the following signature:

`func(args ...interface{}) (interface{}, error)`

Where `args` is whatever is passed to the function when called. If a non-nil error is returned from a function during evaluation, the evaluation stops and ultimately returns that error to the caller of `Evaluate()` or `Eval()`.

## Built-in functions

There aren't any builtin functions. The author is opposed to maintaining a standard library of functions to be used.

Every use case of this library is different, and even in simple use cases (such as parameters, see above) different users need different behavior, naming, or even functionality. The author prefers that users make their own decisions about what functions they need, and how they operate.