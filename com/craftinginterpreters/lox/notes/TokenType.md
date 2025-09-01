package com.craftinginterpreters.lox;

enum TokenType {

    // Single-character tokens
    LEFT_PAREN, RIGHT_PAREN, LEFT_BRACE, RIGHT_BRACE,
    COMMA, DOT, MINUS, PLUS, SEMICOLON, SLASH, START, 
    
    // One or two character tokens.
    BANG, BANG_EQUAL, EQUAL_EQUAL, GREATER, GREATER_EQUAL, 
    LESS, LESS_EQUAL,
    
    // Literals. 
    IDENTIFIER, STRING, NUMBER, 
    
    // Keywords. 
    AND, CLASS, ELSE, FALSE, FUN, FOR, IF, NIL, OR, 
    PRINT, RETURN, SUPER, THIS, TRUE, VAR, WHILE , EOF

}


The above code defines an `enum` named `TokenType` in the `com.craftinginterpreters.lox` package.  
This `enum` lists all possible token types that a lexer or parser might recognize in a programming language, such as 
operators, literals, and keywords.  
It is used to categorize and identify different parts of source code during interpretation or compilation.

An `enum` (short for "enumeration") is a special data type in Java that defines a set of named constants. It is used to represent a fixed set of related values, such as days of the week, 
directions, or token types in a parser. Enums improve code readability and type safety by restricting variables to predefined values.