---
title: "1. Simple grammar to define loop semantics"
series: ["Writing a Parser for a Small Language"]
date: 2020-01-18T14:37:50+03:30
draft: false
tags: [antlr, kotlin, parser, grammar]
---

For a long time I've been facinated by parsers and in particularly [ANTLR] (the parser generator).

ANTLR is a parser generator which makes it simple to define a grammar made of tokens (lexer rules) and a parser rules, and then generate code to parse the defined grammar.

There are plenty of other articles explaining parsers, and in particular ANTLR, but I wanted to skip that and move straight into defining my own interpreted language leveraging the code generated by ANTLR.

> TODO: Listener vs Visitor pattern

## Series

1. Simple grammar to define loop semantics
2. Implementing the parser (in Kotlin)
2. Add variable assignment and interpolation
3. Add branching (if/else)

## StupidLang

... is the name of my stupid language. It's not made to be a useful language, but rather a means to learning about ANTLR and the Visitor pattern.

For starters, I wanted to be able to implement a loop.

```stupid
repeat 5 {
    print "hello world";
};
```

### Lexer Rules

Lexer rules make the parser grammar a lot easier to define, and improves on the utility of the generated code (you'll end up with objects representing the tokens rather than having to dig out and check strings). More on that later.

Looking at the example above, I see a number of tokens. In ANTLR grammar (g4) files, tokens must start with a capital letter (ideally complete caps) to differentiate from parser rules which begin with a lower case.

**Decisions**:
- Keywords will be tokenized.
- I want the script to be parsable without whitespace (this makes the parser rules easy to write because we are only concerned with a stream of meaningful tokens)

**StupidLangLexer.g4**

```antlr
// Specify that this are the lexer rules
lexer grammar StupidLangLexer;

// statement keywords
REPEAT          : 'repeat';
PRINT           : 'print';

// Types
NUMBER          : [0-9]+;

// Special characters
LBRACE          : '{';
RBRACE          : '}';
SEMICOLON       : ';';
DQUOTE          : '"';

// Send all whitespace to a hidden channel
WS              : [\r\n\t ]+ -> channel(HIDDEN);
```

The above seems pretty straight forward. Let's now write some parsing rules.

### Parser Rules

Parser rules define the actual structure of the langauge, and use tokens from the lexer rules to both the writing and reading of the grammar.

**Decisions**: 
- Each script file will be interpreted alone and in isolation for now.
- An empty file is valid.

Using the loop example above, it seems like we start of with a file, which contains zero or more statements. 

We support two statements so far:
- `repeat` with two parameters (a number, and a block or closure)
- `print` with a string parameter, and a semicolon to indicate the end of the statement.

**Note**: Typically code blocks (indicated by `{` and `}`) do not need a semicolon `;` as their end is unambiguous, but the parser rules become a bit more complex so I have made it mandatory. I could infer stataments end with a newline, but remeber, we are hiding whitespace. There are tradeoffs.

I like to put the entry point rule at the start to make it obvious. In this case `file`.

**StupidLangParser.g4**

```antlr
// Specify that this are the parser rules
parser grammar StupidLangParser;

// Refer to the lexer rules
options { tokenVocab = StupidLangLexer; }

// The file contains (or doesn't contain) statements
// EOF is a special token when then end of the token stream is reached.
file
    : statements? EOF
    ;

// Statements are one or more statement followed by a terminator
statements
    : (statement SEMICOLON)+
    ;

// We permit a number of statements
statement
    : repeat
    | print
    ;

// Repeat takes two parameters, a number and a code block
repeat
    : REPEAT times LBRACE statements? RBRACE
    ;

// Just a shortcut rule to make sense of the grammar.
times
    : NUMBER
    ;

// Print non greedily takes everything between two double quotes
// Do you spot the problem?
print
    : PRINT DQUOTE .*? DQUOTE
    ;

```

### Our first problem: Missing whitespaces

In trying to make the grammar simple to define, we told the lexer to ignore whitespaces... but what happens when we want to do `print "hello world"`? The parser would see the stream of charaters between the `DQUOTE` tokens as `helloworld`.

So we mostly want to hide whitespaces, but not always... Hmm, _I wish we could jump into a different mode for when we need to parse strings, and not hide whitespaces._

## Modes

Modes allow us to jump into a different parsing context, for when we need to sometimes apply different lexer rules. This is perfect. We want to remove whitespace in general, except for when we know we are parsing a string (indicated by the `DQUOTE` token).s

In ANTLR, modes are pushed onto, and popped off a stack as the parser takes in the stream of tokens.

So, let's revise our lexer grammar:

**StupidLangLexer.g4**

```antlr
// Special characters
// ...
DQUOTE          : '"' -> pushMode(STRING_MODE);

// Send all whitespace to a hidden channel
WS              : [\r\n\t ]+ -> channel(HIDDEN);

mode STRING_MODE;
S_DQUOTE        : '"' -> type(DQUOTE), popMode;
CHARACTER       : .;
```

**Important**: The rule for sending whitespace to a hidden channel must appear before any modes are defined, but after all rules that `pushMode` into a mode that should have spaces preserved.

**Note**: The rule names within each mode must be unique, hence the prefixing. Also notice that as we use `type(DQUOTE)` to equate the redefined double-quote with the already defined `DQUOTE`. _Todo: find out exactly what this does, I assume it is so you can simply use the main token inside the parser rules.

The stack nature of modes gives us a lot of power. For example, we could allow escaped charaters: `print "double quote: \""`. We could just define some extra lexer rules to take us into a literal parsing mode whenever we see a `\` from string processing mode.

**StupidLangLexer.g4**

```antlr

mode STRING_MODE;
S_ESCAPE        : '\' -> pushMode(LITERAL_MODE);
S_DQUOTE        : '"' -> type(DQUOTE), popMode;
CHARACTER       : .;

mode LITERAL_MODE;
LITERAL         : . -> popMode;
```

You could extend this to be able to do things like string interpolation of variables.

Now we can adjust the parser rule:

**StupidLangParser.g4**

```antlr
// ...

print
    : PRINT DQUOTE string DQUOTE
    ;

// Shortcut so we have a named property in the generated code
string
    : .*?
    ;

```

## Verify

Generate and compile the code

```sh
antlr4 StupidLangLexer.g4
antlr4 StupidLangParser.g4
javac StupidLang*.java
```

Make a file with the stupid script:

```sh
cat >> /tmp/loop.stupid <<EOF
repeat 5 {
    print "hello world";
};

repeat 3 {
    print "hi";
};

print "the end";
EOF
```

Run it against ANTLR's parse tree viewer:

```sh
grun StupidLang file -gui < /tmp/loop.stupid
```

![Parse Tree](/antlr/part1/antlr4_parse_tree.png)

---
[ANTLR]: https://www.antlr.org/