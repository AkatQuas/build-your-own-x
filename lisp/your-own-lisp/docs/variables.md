# Variables

[docs](http://www.buildyourownlisp.com/chapter11_variables)

codes:
   
- [variables](../codes/variables.c)

## Immutability

We are adding some new features to the language. In this chapter, we add **variables**.

The variables are **immutable** meaning they cannot change. Actually the variables are simply a way of **naming values**. They let us assign a **name** to a **value**, and then let us get a copy of that **value** later on when we need it.

To allow for **naming values** we need to create a structure which stores the name and value of everything named in our program. We call this the **environment**. When we start a new interactive prompt we want to create a new environment to go along with it, in which each new bit of input is evaluated. Then we can store and recall variables as we program.

>What happens when we re-assign a name to something new? Isn't this like mutability?
>
> In our Lisp, when we re-assign a name we're going to delete the old association and create a new one. This gives the illusion that the thing assigned to that name has changed, and is mutable, but in fact we have deleted the old thing and assigned it a new thing. This is different to C where we really can change the data pointed to by a pointer, or stored in a struct, without deleting it and creating a new one.

## Symbol Syntax

Now that we're going to allow for user defined variables we need to update the grammar for symbols to be more flexible. Rather than just our builtin functions it should match any possible valid symbol. Unlike in C, where the name a variable can be given is fairly restrictive, we're going to allow for all sorts of characters in the name of a variable.

We can create a regular expression that expresses the range of characters available as follows.

    /[a-zA-Z0-9_+\\-*\\/\\\\=<>!&]+/

On first glance this looks like we've just bashed our hands into keyboard. Actually it is a regular expression using a big range specifier `[]`. Inside range specifiers special characters lose their meaning, but some of these characters still need to be escaped with backslashes. Because this is part of a C string we need to put two backslashes to represent a single backslash character in the input.

This rule lets symbols be any of the normal C identifier characters `a-zA-Z0-9_` the arithmetic operator characters `+\\-*\\/` the backslash character `\\\\` the comparison operator characters `=<>!` or an ampersands `&`. This will give us all the flexibility we need for defining new and existing symbols.

```c
mpca_lang(MPCA_LANG_DEFAULT,
    "\
        number: /-?[0-9]+/ ;
        symbol: /[a-zA-Z0-9_+\\-*\\/\\\\=<>!&]+/ ;\
        sexpr: '(' <expr>* ')'; \
        qexpr: '{' <expr>* '}'; \
        expr: <number> | <symbol> | <sexpr> | <qexpr>; \
        lispy: /^/ <expr>* /$/; \
    ",
    Number, Symbol, Sexpr, Qexpr, Expr, Lispy);
```

## Function Pointers 

Once we introduce variables, symbols will no longer represent functions in our language, but rather they will represent a name for us to look up into our environment and get some new value back from.

Therefore we need a new value to represent functions in our language, which we can return once one of the builtin symbols is encountered. To create this new type of `lval` we are going to use something called a **function pointer**.

Function pointers are a great feature of C that lets you store and pass around pointers to functions. It doesn't make sense to edit the data pointed to by these pointers. Instead we use them to call the function they point to, as if it were a normal function.

Like normal pointers, function pointers have some type associated with them. This type specifies the type of the function pointed to, not the type of the data pointed to. This lets the compiler work out if it has been called correctly.

In the previous chapter our builtin functions took a `lval*` as input and returned a `lval*` as output. In this chapter our builtin functions will take an extra pointer to the environment `lenv*` as input. We can declare a new function pointer type called `lbuiltin`, for this type of function, like this.

    typedef lval*(*lbuiltin)(lenv*, lval*);
    
> Why is that syntax so odd?
>
> In some places the syntax of C can look particularly weird. It can help if we understand exactly why the syntax is like this. Let us de-construct the syntax in the example above part by part.
>
> First the `typedef`. This can be put before any standard variable declaration. It results in the name of the variable, being declared a new type, matching what would be the inferred type of that variable. This is why in the above declaration what looks like the function name becomes the new type name.
>
> Next all those `*`. Pointer types in C are actually meant to be written with the star `*` on the left hand side of the variable name, not the right hand side of the type `int *x`;. This is because C type syntax works by a kind of inference. Instead of reading "Create a new `int` pointer `x`". It is meant to read "Create a new variable `x` where to dereference `x` results in an `int`." Therefore `x` is inferred to be a pointer to an `int`.
>
>This idea is extended to function pointers. We can read the above declaration as follows. "To get an `lval*` we dereference `lbuiltin` and call it with a `lenv*` and a `lval*`." Therefore `lbuiltin` must be a function pointer that takes an `lenv*` and a `lval*` and returns a `lval*`.

## Cyclic Types

The `lbuiltin` type references the `lval` type and the `lenv` type. This means that they should be declared first in the source file.

But we want to make a `lbuiltin` field in our `lval` struct so we can create function values. So therefore our `lbuiltin` declaration must go before our `lval` declaration. This leads to what is called a cyclic type dependency, where two types depend on each other. 

We've come across this problem before with functions which depend on each other. The solution was to create a **forward declaration** which declared a function but left the body of it empty.

In C we can do exactly the same with types. First we declare two `struct` types without a body. Secondly we typedef these to the names `lval` and `lenv`. Then we can define our `lbuiltin` function pointer type. And finally we can define the body of our `lval` struct. Now all our type issues are resolved and the compiler won't complain any more.

```c
struct lval;
struct lenv;
typedef struct lval lval;
typedef struct lenv lenv;

enum { LVAL_ERR, LVAL_NUM, LVAL_SYM, LVAL_FUN, LVAL_SEXPR, LVAL_QEXPR };

typedef lval*(*lbuiltin)(lenv*, lval*);

struct lval {
    int type;
    long num;
    char* err;
    char* sym;
    lbuiltin fun;
    
    int count;
    lval** cell;
};
```

## Function Type

As we've added a new possible `lval` type with the enumeration `LVAL_FUN`. We should update all our relevant functions that work on `lvals` to deal correctly with this update. In most cases this just means inserting new cases into switch statements.

We can start by making a new constructor function for this type.

```c
lval* lval_fun(lbuiltin func) {
    lval* v = malloc(sizeof(lval));
    v->type = LVAL_FUN;
    v->fun = func;
    return v;
}
```

On `lval_del`, we don't need to do anything special for function pointers.

```c
case LVAL_FUN: break;
```

On `lval_print`, we can just print out a nominal string.

```c
case LVAL_FUN: printf("<function>"); break;
```

We're also going to add a new function for **copying** an `lval`. This is going to come in useful when we put things into, and take things out of, the environment. 

For numbers and functions we can just copy the relevant fields directly. For strings we need to copy using `malloc` and `strcpy`. To copy lists we need to allocate the correct amount of space and then copy each element individually.

```c
lval* lval_copy(lval* v) {
    lval* x = malloc(sizeof(lval));
    x->type = v->type;
    
    switch (v->type) {
        /* Copy Functions and Numbers directly */
        case LVAL_FUN: x->fun = v->fun; break;
        case LVAL_NUM: x->num = v->num; break;
        
        /* Copy Strings using malloc and strcpy */
        case LVAL_ERR:
            x->err = malloc(strlen(v->err)+1);
            strcpy(x->err, v->err); break;
            
        case LVAL_SYM:
            x->sym = malloc(strlen(v->sym) + 1);
            strcpy(x->sym, v->sym); break;
            
        /* Copy Lists by coping each sub-expression */
        case LVAL_SEXPR:
        case LVAL_QEXPR:
            x->count = v->count;
            x->cell = malloc(sizeof(lval*) * x->count);
            for(int i = 0; i < x->count; i++) {
                x->cell[i] = lval_copy(v->cell[i]0;
            }
            break;
    }
    
    return x;
}   
```

## Environment

The environment structure must encode a list of relationships between **names** and **values**. There are many ways to build a structure that can do this sort of thing. We are going to go for the simplest possible method that works well. This is to use two lists of equal length. One is a list of `lval*`, and the other is a list of `char*`. Each entry in one list has a corresponding entry in the other list at the same position.

We've already forward declared the `lenv` struct, so it's time to define it as follows:

```c
strcut lenv {
    int count;
    char** syms;
    lval** vals;
};
```

We need some functions to create and delete this structure. Creation initialises the struct fields, while deletion iterates over the items in both lists and deletes or frees them.

```c
lenv* lenv_new(void) {
    lenv* e = malloc(sizeof(lenv));
    e->count = 0;
    e->syms = NULL;
    e->vals = NULL;
    return e;
}

void lenv_del(lenv* e) {
    for (int i = 0; i< e->count; i++) {
        free(e->syms[i]);
        lval_del(e->vals[i]);
    }
    free(e->syms);
    free(e->vals);
    free(e);
}
```

Next we can create two functions that either get values from the environment or put values into it.

To get a value from the environment we loop over all the items in the environment and check if the given symbol matches any of the stored strings. If we find a match we can return a copy of the stored value. If no match is found we should return an error.

The function for putting new variables into the environment is a little bit more complex. First we want to check if a variable with the same name already exists. If this is the case we should replace its value with the new one. To do this we loop over all the existing variables in the environment and check their name. If a match is found we delete the value stored at that location, and store there a copy of the input value.

If no existing value is found with that name, we need to allocate some more space to put it in. For this we can use `realloc`, and store a copy of the `lval` and its name at the newly allocated locations.

```c
lval* lenv_get(lenv* e, lval* k) {
    for (int i = 0; i < e->count; i++ ) {
        if (strcmp(e->syms[i], k->sym) == 0) {
            return lval_copy(e->vals[i]);
        }
    }
    
    return lval_err("unbound symbol!");
}

void lenv_put(lenv* e, lval* k, lval* v) {
    for (int i = 0; i < e->count; i++ ) {
        if (strcmp(e->sym[i], k->sym) == 0) {
            lval_del(e->vals[i]);
            e->vals[i] = lval_copy(v);
            return;
        }
    }
    
    e->count++;
    e->vals = realloc(e->vals, sizeof(lval*) * e->count);
    e->syms = realloc(e->syms, sizeof(char*) * e->count);
    
    e->vals[e->count-1] = lval_copy(v);
    e->syms[e->count-1] = malloc(strlen(k->sym)+1);
    strcpy(e->syms[e->count-1], k->sym);
}
```

## Variable Evaluation

The evaluation function now depends on some environment. We should pass this in as an argument and use it to get a value if we encounter a symbol type. Because our environment returns a copy of the value we need to remember to delete the input symbol `lval`.

```c
lval* lval_eval(lenv* e, lval* v) {
    if (v->type == LVAL_SYM) {
        lval* x = lenv_get(e, v);
        lval_del(v);
        return x;
    }
    
    if (v->type == LVAL_SEXPR) { return lval_eval_sexpr(e, v); }
    
    return v;
}
```

Because we've added a function type, our evaluation of S-Expressions also needs to change. Instead of checking for a symbol type we want to ensure it is a function type. If this condition holds we can call the `fun` field of the `lval` using the same notation as standard function calls.

```c
lval* lval_eval_sexpr(lenv* e, lval* v) {
    for (int i = 0; i < v->count; i++) {
        v->cell[i] = lval_eval(e, v->cell[i]);
    }
    
    for (int i = 0; i < v->count; i++) {
        if (v->cell[i]->type == LVAL_ERR) { return lval_take(v, i);
    }
    
    if (v->count == 0) { return v; }
    if (v->count == 1) { return lval_take(v, 0); }
    
    lval* f = lval_pop(v, 0);
    
    if (f->type != LVAL_FUN) {
        lval_del(v);
        lval_del(f);
        return lval_err("first element is not a function");
    }
    
    lval* result = f->fun(e,v);
    lval_del(f);
    return result;
}
```

## Builtins

Now that our evaluation relies on the new function type we need to make sure we can register all of our builtin functions with the environment before we start the interactive prompt. At the moment our builtin functions are not the correct type. We need to change their type signature such that they take in some environment, and where appropriate change them to pass this environment into other calls that require it.

Once we've changed the builtins to the correct type we can create a function that registers all of our builtins into an environment.

For each builtin we want to create a function `lval` and symbol `lval` with the given name. We then register these with the environment using `lenv_put`. The environment always takes or returns copies of a values, so we need to remember to delete these two `lval` after registration as we don't need them any more.

If we split this task into two functions we can neatly register all of our builtins with some enviroment.

```c
void lenv_add_builtin(lenv* e, char* name, lbuiltin func) {
    lval* k = lval_sym(name);
    lval* v = lval_fun(func);
    lenv_put(e, k, v);
    lval_del(k); lval_del(v);
}

vodi lenv_add_builtins(lenv* e) {
    /* List Functions */
    lenv_add_builtin(e, "list", builtin_list);
    lenv_add_builtin(e, "head", builtin_head);
    lenv_add_builtin(e, "tail", builtin_tail);
    lenv_add_builtin(e, "eval", builtin_eval);
    lenv_add_builtin(e, "join", builtin_join);

    /* Mathematical Functions */
    lenv_add_builtin(e, "+", builtin_add);
    lenv_add_builtin(e, "-", builtin_sub);
    lenv_add_builtin(e, "*", builtin_mul);
    lenv_add_builtin(e, "/", builtin_div);
}
```

The final step is to call this function before we create the interactive prompt. We also need to remember to delete the environment once we are finished.

```c
lenv* e = lenv_new();
lenv_add_builtins(e);

while(1) {
    char* input = readline("lispy> ");
    add_history(input);
    
    mpc_result_t r;
    
    if (mpc_parse("<stdin>", input, Lispy, &r)) {
        lval* x = lval_eval(e, lval_read(r.output));
        lval_println(x);
        lval_del(x);
        
        mpc_ast_delete(r.output);
    } else {
        mpc_err_print(r.error);
        mpc_err_delete(r.error);
    }
    
    free(input);
}

lenv_del(e);
```

## Define Function

We've managed to register our builtins as variables but we still don't have a way for users to define their own variables.

This is actually a bit awkward. We need to get the user to pass in a symbol to name, as well as the value to assign to it. But symbols can't appear on their own. Otherwise the evaluation function will attempt to retrieve a value for them from the environment.
                                
The only way we can pass around symbols without them being evaluated is to put them between `{}` in a quoted expression. So we're going to use this technique for our define function. It will take as input a list of symbols, and a number of other values. It will then assign each of the values to each of the symbols.

This function should act like any other builtin. It first checks for error conditions and then performs some command and returns a value. In this case first checks that the input arguments are the correct types. It then iterates over each symbol and value and puts them into the environment. If there is an error we can return it, but on success we will return the empty expression `()`.

```c
lval* builtin_def(lenv* e, lval* a) {
    LASSERT(a, a->cell[0]->type == LVAL_QEXPR, "Function 'def' passed incorrect type!");
    
    lval* syms = a->cell[0];
    
    for (int i = 0; i < syms->count; i++) {
        LASSERT(a, syms->cell[i]->type == LVAL_SYM, "Function 'def' cannot define non-symbols");
    }
    
    LASSERT(a, syms->count == a->count -1, "Function 'def' cannot define incorrect number of values of symbols");
    
    for (int i= 0; i < syms->count; i++) {
        lenv_put(e, syms->cell[i], a->cell[i+1]);
    }   
    
    lval_del(a);
    
    return lval_sexpr();
}
```

We need to register this new builtin using our builtin function `lenv_add_builtins`.

```c
lenv_add_builtin(e, "def", builtin_def);
```

Now we should be able to support user defined variables. Because our `def` function takes in a lsti of symbols we can do some cool things storing and manipulating symbols in lists before passing them to be defined. Have a play around in the prompt and ensure everything is working correctly.

## Error Reporting

So far the error reporting doesn't work so well. We can report when an error occurs, and give a vague notion of what the problem was, but we don't give the user much information about what exactly has gone wrong. For example if there is an unbound symbol we should be able to report exactly which symbol was unbound. This can help the user track down errors, typos, and other trivial problems.

Wouldn't it be great if we could write a function that can report errors in a similar way to how `printf` works. It would be ideal if we could pass in strings, integers, and other data to make our error messages richer.

The `printf` function is a special function in C because it takes a variable number of arguments. We can create our own **variable arguments** functions, which is what we're going to da to make our error reporting better.

We'll modify `lval_err` to act in the same way as `printf`, taking in a format string, and after that a variable number of arguments to match into this string.

To declare that a function takes variables arguments in the type signature you use the special syntax of ellipsis `...`, which represent the rest of the arguments.

```c
lval* lavl_err(char* fmt, ...);
```

Then, inside the function there are standard library functions we can use to examine what the caller has passed in.

The first step is to create a `va_list` struct and initialise it with `va_start`, passing in the last named argument. For other purposes it is possible to examine each argument passed in using `va_arg`, but we are going to pass our whole variable argument list directly to the `vsnprintf` function. This function acts like `printf` but instead writes to a string and takes in a `va_list`. Once we are done with our variable arguments, we should call `va_end` to cleanup any resources used.

The `vsnprintf` function outputs to a string, which we need to allocate some first. Because we don't know the size of this string until we've run the function we first allocate a buffer `512` characters big and then reallocate to a smaller buffer once we've output to it. If an error message is going to be longer than 512 characters it will just get cut off, but hopefully this won't happen.

Putting it all together the new error function looks like this:

```c
lval* lval_err(char* fmt, ...) {
    lval* v = malloc(sizeof(lval));
    v->type =LVAL_ERR;
    
    va_list va;
    va_start(va, fmt);
    
    v->err = malloc(512);
    
    vsnprintf(v->err, 511, fmt, va);
    
    v->err = realloc(v->err, strlen(v->err) + 1);
    
    va_end(va);
    
    return v;
}
```

Using this we can then start adding in some better error messages to our functions. As an example we can look at `lenv_get`. When a symbol can't be found, rather than reporting a generic error, we can actually report the name that was not found.

```c
return lval_err("Unbound Symbol '%s'", k->sym);
```

We can also adapter the `LASSERT` macro such that it can take variable arguments too. Because this is a macro and not a standard function the syntax is slightly different. On the left hand side of the definition we use the ellipses notation again, but on the right hand side we use a special variable `__VA_ARGS__` to paste in the contents of all the other arguments.

We need to prefix this special variable with two hash signs `##`. This ensure that it is pasted correctly when the marco is passed no extra arguments. In essence what this does is make sure to remove the leading comma `,` to appear as if no extra arguments were passed in.

```c
#define LASSERT(args, cond, fmt, ...) \
    if (!(cond)) { \
        lval* err = lval_err(fmt, ##__VA_ARGS__); \
        lval_del(args);\
        return err;\
    }
```

Now we can update some of our error messages to make them more informative. For example if the incorrect number of arguments were passed we can specify how many were required and how many were given.

```c
LASSERT(a, a->count == 1, "Function 'head' passed too many arguments. Expected %i, got %i", 1, a->count);
```

We can also improve the error reporting for type errors. We should attempt to report what type was expected by a function and what type it actually got. Before we can do this it would be useful to have function that took as input some type enumeration and returned a string representation of that type.

```c
char* ltype_name(int t) {
    switch(t) {
        case LVAL_FUN: return "Function";
        case LVAL_NUM: return "Number";
        case LVAL_ERR: return "Error";
        case LVAL_SYM: return "Symbol";
        case LVAL_SEXPR: return "S-Expression";
        case LVAL_QEXPR: return "Q-Expression";
        default: return "Unknown";
    }
}

LASSERT(a, a->cell[0]->type == LVAL_QEXPR, "Function 'head' passed incorrect type for argument 0. Expected %s, got %s", ltype_name(a->cell[0]->type), ltype_name(LVAL_QEXPR));
```

[Next on Functions](functions.md)
