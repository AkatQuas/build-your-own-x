In the previous article you have learned how to parse (recognize) and interpret arithmetic expressions with any number of plus or minus oprators in them. You also learned about syntax diagrams and how they can be used to specify the syntax of a programming language.

Today you're going to learn how to parse and interpret arithmetic expressions with any number of multiplication and division operators in them, for example *7 * 4 / 2 * 3*. The division in this article will be an integer division, so if the expression is *9 / 4*, then the answer will be an integer: 2.

We will also learn quite a bit today about another widely used notation for specifying the syntax of a programming language. It's called *context-free grammars* (grammars, for short) or *BNF (Backus-Naur Form)*. For the purpose of this article I will not use pure BNF notation but more like a modified EBNF notation.

Here are a couple of reasons to use grammars:

1. A grammar specifies the syntax of a programming language in a concise manner. Unlike syntax diagrams, grammars are very compact. You will see me using grammars more and more in future articles.
1. A grammar can serve as great documentation.
1. A grammar is a good starting point even if you manually write your parser from scratch. Quite often you can just convert the grammar to code by following a set of simple rules.
1. There is a set of tools, called *parser generators*, which accept a grammar as an input and automatically generate a parser for you based on that grammar. I will talk about those tools later on in the series.

Here is a grammar that describes arithmetic expressions like *7 * 4 / 2 * 3* (it's just one of the many expressions that can be generated by the grammar):

![](./imgs/lsbasi_part4_bnf1.png)

A grammer consists fo a sequence of *rules*, also known as *productions*, There are two rules in our grammar:

![](./imgs/lsbasi_part4_bnf2.png)

A rule consists fo a *non-terminal*, called the *head* or *left-hand side* of the production, a colon, and a sequence of terminals and/or non-terminars, called the *body* or *right-hand side* of the productions:

![](./imgs/lsbasi_part4_bnf3.png)

In the grammar I showed above, tokens like **MUL, DIV, and INTEGER** are called *terminals* and variables like *expr* and *factor* are called *non-terminals*. Non-terminals usually consist of a sequence of terminals and/or non-terminals:

![](./imgs/lsbasi_part4_bnf4.png)

The non-terminal symbol on the left side of the first rule is called the *start symbol*. In the case of our grammar, the start symbol is *expr*:

![](./imgs/lsbasi_part4_bnf5.png)

What is a *factor*? For the purpose of this article a *factor* is just an integer.

Let's quickly go over the symbols used in the grammar and their meaning.

- | - Alternatives. A bar means “or”. So (MUL | DIV) means either MUL or DIV.
- ( … ) - An open and closing parentheses mean grouping of terminals and/or non-terminals as in (MUL | DIV).
- ( … )* - Match contents within the group zero or more times.

A grammar defines a language by explaining what sentences it can form. This is how you can *derive* an arithmetic expression using the grammar: first you begin with the start symbol *expr* and then repeatedly replace a non-terminal by the body of a rule for that non-terminal until you have generated a sentence consisting solely of terminals. Those sentences form a language defined by the grammar.

If the grammar cannot derive a certain arithmetic expression, then it doesn't support that expression and the parser will generate a syntax error when it tries to recognize the expression.

Now, let's map the above grammar to code, okay?

Here are the guidelines that we will use to convert the grammar to source code. By following them, you can literally translate the grammar to a working parser:

1. Each rule, R, defined in the grammar, becomes a method with the same name, and references to that rule become a method call: R(). The body of the method follows the flow of the body of the rule using the very same guidelines.
1. Alternatives (a1 | a2 | aN) become an if-elif-else statement
1. An optional grouping (…)* becomes a while statement that can loop over zero or more times
1. Each token reference T becomes a call to the method eat: eat(T). The way the eat method works is that it consumes the token T if it matches the current lookahead token, then it gets a new token from the lexer and assigns that token to the current_token internal variable.

Visually the guidelines look like this:

![](./imgs/lsbasi_part4_rules.png)

Let's get moving and convert our grammar to code following the above guidelines.

There are two rules in our grammar: one *expr* rule and one *factor* rule. Let's start with the *factor* rule (production). According to the guidelines, you need to create a method called *factor* that has a single call to the *eat* method to consume the INTEGER token (guideline 4):

```python
def factor(self):
    self.eat(INTEGER)
```

The rule *expr* becomes the *expr* method. The body of the rule starts with a reference to *factor* that becomes a *factor()* method call. The optional grouping *(...)\** becomes a *while loop* and *(MUL|DIV)* alternatives become an *if-elif-else* statement. By combing those pieces together we get the following *expr* method:

```python
def expr(self):
    self.factor()

    while self.current_token.type in (MUL, DIV):
        token = self.current_token
        if token.type == MUL:
            self.eat(MUL)
            self.factor()
        elif token.type == DIV:
            self.eat(DIV)
            self.factor()
```

This is how a syntax diagram for the same *expr* rule look like:

![](./imgs/lsbasi_part4_sd.png)

It's about time to dig into the source code of the new arithemtic expression interpreter. Below is the code of a calculator that can handle valid arithmetic expressions containing integers and any number of multiplication and division (integer division) operators.

```python
# Token types
#
# EOF (end-of-file) token is used to indicate that
# there is no more input left for lexical analysis
INTEGER, MUL, DIV, EOF = 'INTEGER', 'MUL', 'DIV', 'EOF'


class Token(object):
    def __init__(self, type, value):
        # token type: INTEGER, MUL, DIV, or EOF
        self.type = type
        # token value: non-negative integer value, '*', '/', or None
        self.value = value

    def __str__(self):
        """String representation of the class instance.
        Examples:
            Token(INTEGER, 3)
            Token(MUL, '*')
        """
        return 'Token({type}, {value})'.format(
            type=self.type,
            value=repr(self.value)
        )

    def __repr__(self):
        return self.__str__()


class Lexer(object):
    def __init__(self, text):
        # client string input, e.g. "3 * 5", "12 / 3 * 4", etc
        self.text = text
        # self.pos is an index into self.text
        self.pos = 0
        self.current_char = self.text[self.pos]

    def error(self):
        raise Exception('Invalid character')

    def advance(self):
        """Advance the `pos` pointer and set the `current_char` variable."""
        self.pos += 1
        if self.pos > len(self.text) - 1:
            self.current_char = None  # Indicates end of input
        else:
            self.current_char = self.text[self.pos]

    def skip_whitespace(self):
        while self.current_char is not None and self.current_char.isspace():
            self.advance()

    def integer(self):
        """Return a (multidigit) integer consumed from the input."""
        result = ''
        while self.current_char is not None and self.current_char.isdigit():
            result += self.current_char
            self.advance()
        return int(result)

    def get_next_token(self):
        """Lexical analyzer (also known as scanner or tokenizer)
        This method is responsible for breaking a sentence
        apart into tokens. One token at a time.
        """
        while self.current_char is not None:

            if self.current_char.isspace():
                self.skip_whitespace()
                continue

            if self.current_char.isdigit():
                return Token(INTEGER, self.integer())

            if self.current_char == '*':
                self.advance()
                return Token(MUL, '*')

            if self.current_char == '/':
                self.advance()
                return Token(DIV, '/')

            self.error()

        return Token(EOF, None)


class Parser(object):
    def __init__(self, lexer):
        self.lexer = lexer
        # set current token to the first token taken from the input
        self.current_token = self.lexer.get_next_token()

    def error(self):
        raise Exception('Invalid syntax')

    def eat(self, token_type):
        # compare the current token type with the passed token
        # type and if they match then "eat" the current token
        # and assign the next token to the self.current_token,
        # otherwise raise an exception.
        if self.current_token.type == token_type:
            self.current_token = self.lexer.get_next_token()
        else:
            self.error()

    def factor(self):
        """Parse integer.
        factor : INTEGER
        """
        token = self.current_token
        self.eat(INTEGER)
        return token.value

    def parse(self):
        """Arithmetic expression parser.
        Grammar:
        expr   : factor ((MUL | DIV) factor)*
        factor : INTEGER
        """
        result = self.factor()

        while self.current_token.type in (MUL, DIV):
            token = self.current_token
            if token.type == MUL:
                self.eat(MUL)
                result = result * self.factor()
            elif token.type == DIV:
                self.eat(DIV)
                result = result / self.factor()

        return result


def main():
    while True:
        try:
            text = input('calc>')
        except EOFError:
            break
        if not text:
            continue

        lexer = Lexer(text)
        parser = Parser(lexer)
        result = parser.parse()
        print(result)


if __name__ == '__main__':
    main()
```

[Next](./part-5.md)