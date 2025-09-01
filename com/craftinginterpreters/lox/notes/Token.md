`com/craftinginterpreters/lox/Token.java`

This class is a simple immutable data holder representing one lexical token produced by a scanner (lexer) when converting raw source text into a structured stream for a parser.

Details:
- `type`: Enum value (`TokenType`) identifying the token category (e.g. identifier, number, keyword, symbol). Drives parser decisions.
- `lexeme`: The exact substring of source text that matched (raw spelling). Preserved for error messages or re-display.
- `literal`: Parsed semantic value for tokens that carry one (e.g. a `Double` for numbers, a `String` for string literals, or `null` for tokens without an intrinsic value like `(` or keywords).
- `line`: Source line number where the token appears, used for precise error reporting.

Constructor:
- Initializes all fields. No setters are provided and fields are `final`, making each `Token` immutable after creation (thread\-safe and reliable to share).

`toString()`:
- Returns a concise debugging representation: `<type> <lexeme> <literal>`. Useful for logging the output of the scanner before parsing.

Lifecycle in a typical interpreter:
1. Scanner reads characters, groups them into lexemes, determines `TokenType`.
2. If the lexeme has a value (number, string), it is converted and stored in `literal`; otherwise `literal` is `null`.
3. A `Token` instance is created and appended to a token list.
4. The parser consumes that list, switching on `type`, optionally using `literal` when building the abstract syntax tree.
5. Errors reported later can reference `line` and `lexeme`.

Design rationale:
- Separates raw text (`lexeme`) from processed meaning (`literal`).
- Keeps metadata (`line`) alongside to avoid parallel structures.
- Uses `toString()` for quick diagnostics without custom formatting code elsewhere.

In summary, this class models the atomic unit of syntactic analysis, carrying both surface form and semantic payload needed downstream.


Think of reading code like reading a sentence.

1. Raw source text: This is just a long string of characters you typed (like a sentence typed with no structure).
2. Scanner / lexer: A tiny helper that walks through those characters and slices them into small meaningful pieces (like cutting the sentence into words and punctuation). Each slice is a token.
3. Token: One of those pieces. It has:
    - `type`: What kind of piece it is (word, number, string, parenthesis, etc.).
    - `lexeme`: The exact characters as they appeared.
    - `literal`: A processed value (for a number token, the actual numeric value; for others, often `null`).
    - `line`: Which line it was on for error messages.
4. Parser: Another helper that later looks at the list (stream) of tokens and figures out the structure (like grammar of the sentence).

"Immutable data holder" means:
- Once a `Token` object is created, its fields never change. That makes it safe and predictable.
- It just stores data; it does not do behavior or logic.

So: The `Token` class is a simple container for one slice of your code, created by the scanner so the parser can understand the program step by step.