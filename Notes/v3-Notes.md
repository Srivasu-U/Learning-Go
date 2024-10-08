## Notes for Monkey v3

### Evaluation, the E in REPL
- Giving code meaning
- Set of rules that defines how a language is interpreter
- If we consider the following code, whether this returns `a` or `b` is dependent of if the interpreter being used considering 5 truthy or not. 
It is truthy in some languages but in some languages, we would have to use something like `5 != 0`
```
let num = 5;
if (num) {
    return a;
} else {
    return b;
}
```
- There are no clear distinctions between what an interpreter and a compiler is
    - The most commonly accepted idea is that compilers provide an executable artifacts while interpreters don't but this doesn't always hold true.
- Interpreters that traverse the AST and do what the node represents are called *tree-walking interpreters*
- There are also interpreters that traverse the AST but don't directly interpret it. Instead, the AST is converted in bytecode. 
    - Bytecode is an intermediate representation (IR) that is neither assembly language, nor machine code.
    - It is not executed by the OS or the CPU, but it is instead interpreted by a VM.
    - If this bytecode is not executed directly in the VM, but the VM instead converted this bytecode to native machine code right before execution, this is called a JIT interpreter/compiler.
        - JIT can also be the name for something that skips the bytecode step entirely. Just compile to native machine code and execute.


### Tree-Walking Interpreter        
- As usual, the choice of style is based on performance and usability needs. Tree-walking is the slowest but the most easy to build and extend. ***This is what is built as part of this project.***
- Based on the Lisp interpreter, inspiration from *The Structure and Interpretation of Computer Programs* - [click here](https://web.mit.edu/6.001/6.037/sicp.pdf)
- Pseudocode for our evaluator
```
function eval(astNode) {
    if (astNode is integerLiteral) {
        return astNode.integerValue
    } else if (astNode is booleanLiteral) {
        return astNode.booleanValue
    } else if (astNode is infixExpression) {
        leftEvaluated = eval(astNode.Left)
        rightEvaluated = eval(astNode.Right)

        if astNode.Operator == '+'{
            return leftEvaluated + rightEvaluate
        } else if astNode.Operator == '-' {
            return leftEvaluated - rightEvaluated
        }
    }
}
```
- What is the return type for eval? - Dependent on the internal object system of the interpreter (?). 
    - We need a system to represent the values generated when evaluating the AST. Done by using the `object.go` class. Essentially creating object wrappers around the base datatypes with helper methods.
    - Consider this code 
    ```
    let a = 5
    [...]
    a + a
    ```
    - We bind the value `5` to `a` and the code continues to execute. But when encountering `a + a`, we need to fetch the `5` again. 
    - In the AST, 5 is a `*ast.IntegerLiteral` but we need to keep track of this and continue to evaluate rest of the AST.
        - Some languages use native types (booleans, integer etc) of the host language to represent values from the interpreted language, without a wrapper
        - Some languages only use pointers
        - Some use a mixture of the two
        - These differences are because the host language can represent its values natively in one way and/or the interpreted language can have its own limitations. For example, the interpreted language may only have booleans and ints, but another interpreted language may need dicts, lists and arrays.
        - Speed of execution and memory usage is another consideration as usual
        - Garbage collection as well
        - TODO: Read source code of Wren [click here](https://github.com/wren-lang/wren)


### Evaluator
- The file `object/object.go` is essentially creating wrapper objects around the datatypes supported by Monkey, aka, int, bool and null.
    - So when an integer literal is encountered by the interpreter, it is converted into an `ast.IntegerLiteral` then during the evaluation phase, it is turned into `object.Integer`. This value is saved in the struct and a reference is passed around to this struct.
    - To put it simply, the `eval` function takes in an `ast.Node` (basically all nodes are `ast.Node`) as an input and returns an `object.Object` as an output
    ```
    func Eval(node ast.Node) object.Object
    ```

#### Self-Evaluating Expressions    
- Self-evaluating expressions = literals, because they evaluate to their own values, such as integers or booleans
    - In the language of Monkey, given as `*ast.IntegerLiteral`, `Eval` needs to return an `*object.Integer` with the same `Value` as `*ast.IntegerLiteral.Value`
        - Looking at `evaluator.go`, if we take an integer literal as an example, such as `5;`, then the `Eval()` method receives the AST, starting with `*ast.Program`. This is evaluated into `[]ast.Statement`
        - For each `ast.Statement`, `Eval()` is called recursively, encountering `*ast.ExpressionStatement` and then `*ast.IntegerLiteral` in case of `5;`
- This is very similar to boolean literals as well, since they are also self-evaluating

#### Prefix Expressions
- As mentioned in `v2-Notes.md`, but this is not the case with the evaluator.
> A lot of Monkey language constructs are treated as a prefix because it the easiest way to parse them and register methods specific to a type of construct.
- Only one operator and one operand is evaluated as a prefix expression. 
    - Prefix operations are ! and -

#### Conditionals
- In Monkey, the consequence block, or the if block, will be executed if the condition resolves to anything *"truthy"*, it doesn't actually need to be *true*. *Truthy* just means not false and not null. This is the design decision for this language. For example
```
let x = 10;
if (x) {
puts("everything okay!");  //this is executed because if (10) is "truthy", if not exactly true
} else {
puts("x is too high!");
shutdownSystem();
}
```

#### Return statements
- The main thing for return statements is that whatever statements are after `return`, are never executed.
- Return values are wrapped in their own objects to be kept track of.

#### Error handling
- Mainly about internal error handling such as wrong operators, unsupported operations and other exceptions
- Errors and return statements are extremely similarly handled because both need to stop the execution of the program as soon as they are encountered
- Errors are also wrapped, like return values, in their own objects.
- The error object used in this interpreter only gives an error message
    - Ideally, in production, the error should be a message and a stack trace with line and column numbers
    - This can be done if the lexer also provides line and column numbers but ours does not do that

#### Bindings
- We have an identifier and a value is must be bound to but how exactly do you create this binding?
- This is done by using something called environments
    - This is essentially a hashmap that associates strings with objects
- Functions have their own environments and function call logic must deal with this separate env
- The function block must execute with params in its own separate env.
    - The current env, aka the "class env" cannot be overridden either
    - The way to solve this is to create a new `object.Environment` with a pointer to the environment it is extending
    - This way, we have two separate `Get()` and `Set()` and we can add to what is necessary/relevant
    - If a value to be `Get()` is not in the function env, then it can check in the class env, until there is no more extending envs and then we can return an error
- ***Note to self: Bindings, function call and function body/literal are interesting parts that can help with more understanding.***
- Functions extend the environment they are currently in, instead of the current environment extending to match a function to deal with 
    - Closures (aka functions within functions). Nested functions have access to their own env and the enclosing env. Something like this is possible in Monkey already
    ```
    let newAdder = fn(x) {
    fn(y) { x + y };
    };
    let addTwo = newAdder(2);
    addTwo(2);
    ```

#### Garbage collection
- Uses Go's own GC
- Writing our own GC would require us to disable Go's GC and then take over the responsibilities of both Monkey memory evaluation and also Go's
- The environments are also destroyed by the GC after they are done