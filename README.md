== Wake Automatic Compiler Compiler

A tool like Yacc that generates parsing libraries for wake projects based on grammars.

Unlike yacc, grammars are specified with annotated classes, making this tool a bit unusual.

### Lexer

At the moment, you have to implement your own `Lexer` and `Token` class from scratch. Shoops! Need more features in wake to make this better!

    module wacc;

    import wacc.LexerShell;

    every Lexer (capable LexerShell):

		@Enum(1, "identifier", "Text")
		@Enum(2, "number", "Num")
		@Enum(3, "*")
        Token? -- lex(Text) {
			..
		}

	every Token:
		needs
			public Num type,
			public Text?,
			public Num?;

When the lexer returns `nothing`, EOF is assumed. The token class must have the `Num type` field, and may have other fields for storing values.

For each token, you must use the `@Enum` annotation to give it a number, name, and optionally a value storage type. Here, type 1 indicates an identifier, 2 a number, and 3
an asterisk. The asterisk does not store a value on the token, while identifier stores a Text and number stores a Num. This is required until Wake gets something like
algebraic data types.

### Super Basic Grammar

Once you have your lexer ready, we can give an example for a trivial grammar that simply accepts "id num". We make this rule our final `@Grammar`, and simply declare
the various tokens as its `needs`.

    @Grammar
    every Grammar:
        needs Text:identifier, Num:number;

This will be able to parse "hello 3", "world 99", etc.

### Generating And Using The Parser

You can compile this class, and then generate your parser with:

    wake Grammar.wk -o Grammar.o
    ./wacc-generator -o Parser.wk
    wake Parser.wk -o Parser.o

It can be used in your program like so

    import Parser;
    import Grammar;

    every Main:

        needs Parser;

        main() {
            var Grammar = Parser.fromText("hello 190");
			// or
            Grammar = Parser.fromFile(...);
		}

Your grammar class can be in a module and it will be automatically detected & used by the generated parser class.

### Multiple Rules and Tokens Without Dynamic Values

The easiest way to use a void token (a token which has no dynamic value beyond its type) is to ask for it as a dependency. Instead of using the type of the dynamic value
(i.e., Text for identifier), you can use the Token class directly. Once again, the specialty indicates the token you wish to match.

    @Grammar
    every Grammar:
        needs Text:identifier, Token:asterisk, Num:number;

Unfortunately this doesn't quite match our tokens previous name of `*`. We can match that as well, though, by using the `@Rule` annotation.

    @Grammar
    @Rule("identifier * number")
    every Grammar:
        needs Text, Num;

Here, the needs `Text` and `Num` match to `identifier` and `number` ordinally. However, we let the `@Rule` do the heavy lifting for us. Notice that the whitespace is ignored.

This also allows us to specify multiple `@Rule`s. Say we wish to match both "id * num" and "num * id". We can do this like so:

    @Grammar
    @Rule("identifier * number")
    @Rule("number * identifier")
    every Grammar:
        needs Text, Num;

Here the needs are still unambigous because the types can only match to one place.

### Disambiguating multiple Rules

When matching multiple rules, we often don't have such a simple situation as above. In this case, we need to match tokens to fields ourselves, which we can do by matching specializers.

    @Rule("number:gt > number:lt")
    @Rule("number:lt < number:gt")
    every Comparison:
        needs Num:gt, $Num:lt;

Here we can match both greater-than expressions and less-than expressions as a single object representation. In the case of "5 > 3", we can tell that `Num` should be set to 5, since 5
matches the left number which is specialized as `:gt`, matching `Num:gt`.

### Nesting Rules

None of our grammars so far would get anyone very far. How would we do something more interesting? Just like we can ask for Tokens and primitives to match tokens, we can ask for
complex classes to match complex rules.

	@Grammar
	every BinaryOperation:
		needs Num:number, Operation, $Num:number;

	// make an Operation interface
	every Operation:

	@Rule("+")
	every AddOperation (capable Operation):

	@Rule("-")
	every SubOperation (capable Operation):

	@Rule("/")
	every DivOperation (capable Operation):

	@Rule("*")
	every MulOperation (capable Operation):

Not only can we ask for the `Operation` classes directly, but as you can see here, we can implement interfaces and ask for the interface instead. In practice, we would want to
make this actually do something by putting some kind of methods on the `Operation` interface.

	@Grammar
	every BinaryOperation:
		needs Num:number, Operation, $Num:number;

		Num -- eval() {
			return Operator.operateOn(Num, $Num);
		}

	every Operation:

		Num -- operateOn(Num, Num);

Now we can match "1 + 2", "2 - 3", "4 * 6", and "8 / 4", etc!

### Recursion

If we want to parse anything interesting, we have to use recursion. This is a bit tricky in Wake since it catches recursive dependencies:

    every RecursiveClassExample:
        needs RecursiveClassExample; // Fails to compile! Cyclic dependencies!

Fortunately, we can use interfaces to solve this. Say we want to find matching parenthesis:

    every MatchingParenthesis:
		Num -- count();

	@Rule("( MatchingParenthesis )")
	every MatchingEnclosingParenthesis (capable MatchingParenthesis):
		needs inner MatchingParenthesis;
		Num -- count() {
			return inner.count() + 1;
		}

	@Rule("()")
	every MatchingEmptyParenthesis (capable MatchingParenthesis):
		Num -- count() {
			return 1;
		}

While `MatchingEmptyParenthesis` and `MatchingEnclosingParenthesis` are different classes, they both share the interface `MatchingParenthesis`. This allows `MatchingEnclosingParenthesis` to nest itself,
as it asks for itself by interface rather than implementation. Since there is another implementation of the interface which does not have a recursive dependency (namely, `MatchingEmptyParenthesis`), we
are satisfied.

### Classes that aren't rules

You may wish to create classes that implement the various rule interfaces which aren't rules. There are two ways to do this.

The first way is to simply mark your class as `@NotARule`.

	@NotARule
	every DynamicOperation (capable Operation):
        //...

This is the opt-out strategy, which is used by default. There is also an opt-in strategy, where your grammar can make everything `@NotARule` by default.

    @Grammar
    @NoImplicitRules
    every Grammar:
        //...

Under the opt-in approach, everything annotated with `@Rule` will be used as such, as will everything directly requested by the grammar, recursively. However,
for the remaining classes in your AST which are matched by interface only, and are not matched by a @Rule, you can opt in with the @ImplicitRule annotation.

	@ImplicitRule
	every InfixFuncOperation (capable Operation):
        needs Text:identifier;


### More

there is no more yet!
