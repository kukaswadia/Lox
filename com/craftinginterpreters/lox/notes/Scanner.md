### What a scanner (lexer) does

A scanner (also called a lexer or tokenizer) converts a flat string of source characters into a structured sequence of tokens. Each token carries:
- Type (e.g. IDENTIFIER, NUMBER, LEFT\_PAREN, DOT)
- The raw lexeme text (exact substring from the source)
- An optional literal value (e.g. numeric value 42.0)
- Line number (for error messages)

### Core state inside the class

- `source`: the entire input program text kept unchanged.
- `start`: index where the current lexeme (token text) began.
- `current`: index of the next character to read (one past the last consumed).
- `line`: current line number (increments on newline).
- `tokens`: list that accumulates produced tokens.

### Key helper methods

1. Constructor: stores the incoming source string so other methods can read characters by index.
2. `isAtEnd()`: returns true when `current` has reached the string length (no more characters).
3. `advance()`: returns the current character then moves `current` forward (post-increment).
4. `addToken(type, literal)`: slices the substring from `start` to `current`, wraps it (plus literal and line) into a Token object, and appends it to `tokens`.
5. (Typically present) `scanToken()`: looks at one character, decides what kind of token starts here (punctuation, string, number, identifier, etc.), consumes the needed characters, then calls `addToken`.

### Scanning loop (conceptual)

Repeat while not `isAtEnd()`:
1. Set `start = current` (mark beginning of next lexeme).
2. Call `scanToken()`:
    - Consume one (or more) characters with `advance()`.
    - Classify them (e.g. a digit starts a number; a letter starts an identifier; `.` is a DOT).
    - Possibly read additional chars (e.g. all digits of a number).
    - Call `addToken(...)`.

After loop, append an EOF token (marks end of stream).

### Why indices instead of building strings char by char

Using `start` and `current` lets the scanner avoid incremental string concatenation. It just remembers where a token began and, once finished, slices once with `substring(start, current)`.

### Example (simplified)

Source: `var x = 42;`

Passes:
- Reads `v a r` → IDENTIFIER token "var"
- Whitespace skipped
- `x` → IDENTIFIER "x"
- `=` → EQUAL token
- `42` → NUMBER token with literal 42.0
- `;` → SEMICOLON token
- EOF added

### Summary flow

Raw characters → controlled cursor movement (`advance()`) → classify patterns → build tokens with text + optional parsed literal → list of tokens consumed later by the parser.

That is the scanner’s job: linear pass, no syntax decisions yet, just turning characters into meaningful symbolic units.


Detailed explanation of `Scanner` (file `com/craftinginterpreters/lox/Scanner.java`)

Purpose: A scanner (lexer) walks through the source text one character at a time and converts raw characters into a sequence of token objects that later phases (parser etc.) can consume.

Current fields:
1. source: The entire input program as a single string.
2. tokens: A growing list that will hold all produced tokens in order.
3. start: Index where the current lexeme (the text of the token now being built) began.
4. current: Index of the next character to read. Everything from start up to but not including current belongs to the current lexeme.
5. line: Current line number (increment whenever a newline is encountered) for error reporting.

Lifecycle idea that the class is aiming for:
Begin at the start of the source. Repeatedly
- Set start to current
- Consume characters belonging to exactly one token
- Classify them and add a Token object to tokens
  When out of characters add a special EOF token so later code can safely look ahead.

What the helper methods are meant to do:
- isAtEnd: Returns true when current has reached or passed source length meaning no more characters.
- advance: Returns the current character and moves current forward by one.
- addToken (overloaded): Captures the substring for the lexeme (source substring from start to current) and appends a new Token with a type and optional literal value.


The scanToken method:
It consumes one character using advance and uses a switch to recognize single character tokens like parentheses braces comma dot minus plus semicolon star. 
For each recognized symbol it calls addToken with the appropriate token type. It currently does not handle other categories (whitespace skipping newlines comments multi character operators strings numbers identifiers keywords error handling etc). 
Also it does not update line because it does not look for newline characters. Characters outside the handled set are silently ignored right now (they just fall out of the switch doing nothing) which is incomplete.

The EOF token addition:
After the loop (if the recursion bug were fixed) it appends a final token whose type is EOF with empty lexeme and null literal carrying the finishing line number. This sentinel lets the parser know it has reached the logical end without needing special bounds checks.

Data flow when fixed conceptually:
1. current and start both zero.
2. Loop begins. start set to current.
3. scanToken consumes one or more characters (currently only one) and calls addToken which slices the lexeme using start and current and appends a Token.
4. Repeat until current equals source length.
5. Append EOF and return the list.

Summary of current shortcomings:
- Critical recursion bug in scanTokens calling itself.
- scanToken only recognizes a small subset of tokens.
- No handling for whitespace newline comments literals identifiers or errors.
- line never increments so all tokens report line 1.
- Unrecognized characters are ignored instead of reported.

In essence this class is a partial skeleton: it sets up the bookkeeping (start current line token collection) and a place for single character token recognition. The main driver needs to call scanToken each loop iteration instead of itself and more logic must be added to recognize all token types and maintain line numbers.

```java
    List<Token> scanTokens() {
        while (!isAtEnd()) {
            // Beginning of the next lexeme
            start = current;
            scanToken();
        }

        tokens.add(new Token(EOF, "", null, line));
        return tokens;
    }
```

Detailed breakdown:

1. `List<Token>`: Generic return type. Method returns a `java.util.List` whose elements are `Token` objects (strongly typed via generics, erases to `List` at runtime).
2. `scanTokens()`: Method name plus empty parameter list (no args). Default (package‑private) access because no modifier (`public`, `private`, etc.) is present.
3. Method body `{ ... }`: Contains the scanning loop, EOF sentinel append, and final return.

Loop section:
- `while (!isAtEnd())`: A pre‑test loop that keeps iterating while the helper `isAtEnd()` returns `false`. The `!` is logical NOT.
- Comment marks intention: start of a new lexeme.
- `start = current;`: Records the index where the next token’s text begins. `start` and `current` are integer indices into the source string.
- `scanToken();`: Consumes characters for exactly one token (advances `current`) and appends a token to `tokens` (via `addToken(...)` inside that method).

After loop:
- `tokens.add(new Token(EOF, "", null, line));`
    - Creates a sentinel end‑of‑file token so downstream phases can safely look ahead.
      - It means:
        - After scanning all real characters, the scanner appends one extra dummy (sentinel) token whose type is `EOF`.
        - This token is not from the source; it just marks the logical end.
        - Parser code can "look ahead" (peek at the next token) without first checking "am I past the list?"—the worst it will see is the `EOF` token.
        - Simplifies logic: loops can read tokens until they encounter `EOF` instead of constantly doing index vs. length checks.
        - Prevents `IndexOutOfBounds` and eliminates special end‑of-input branches; the sentinel is a safe, always‑present stopper.

  - `EOF`: An enum constant from `TokenType`.
  - `""`: Empty lexeme (no source characters correspond to EOF).
  - `null`: No literal value payload.
  - `line`: The current line number for error reporting.

Return:
- `return tokens;`: Hands back the accumulated ordered list of all tokens including the final EOF.

Flow summary:
Initialize indices. Repeatedly set the start of the next lexeme, scan one token, stop when the entire source string is consumed, append EOF, return the list.

```java
    private void addToken(TokenType type, Object literal) {
        String text = source.substring(start, current);
        tokens.add(new Token(type, text, literal, line));
    }
```

**Explanation**

- `private`: Method visible only inside the `Scanner` class.
- `void`: Returns no value.
- `addToken`: Method name (helper used internally when a token has been recognized).
- Parameters:
    - `TokenType type`: Enum (or similar) indicating the kind of token (e.g. IDENTIFIER, NUMBER, PLUS).
    - `Object literal`: Optional parsed literal value (e.g. a `Double` for numbers, `String` for strings); may be `null`.
- `String text = source.substring(start, current);`:
    - `source` is the entire input source code string.
    - `start` marks the index where the current lexeme began.
    - `current` is one past the last consumed character.
    - `substring(start, current)` extracts the raw lexeme text.
- `tokens.add(new Token(type, text, literal, line));`:
    - Constructs a `Token` with:
        - token type
        - lexeme text (what appeared in source)
        - literal value (semantic value or `null`)
        - `line` number for error reporting
    - Appends it to the `tokens` list (`List<Token>`).
- Purpose: Centralizes creation & storage of a token after scanning its characters, keeping lexing logic clean.

```java
    private boolean isAtEnd() {
        return current >= source.length();
    }

    private char advance() {
        return source.charAt(current++);
    }
```

\- `private boolean isAtEnd()`  
\- `private`: accessible only inside the `Scanner` class.  
\- `boolean`: returns a truth value.  
\- Purpose: answers "have we consumed all source characters?"  
\- Expression `current >= source.length()`:  
\- `current` is the index of the next char to read.  
\- `source.length()` is one past the last valid index.  
\- When `current` equals (or surpasses, defensively) the length, the scan is finished (end of file/input).

\- `private char advance()`  
\- Returns the current character and advances the cursor.  
\- `source.charAt(current++)` uses the post\-increment operator:  
\- Retrieves the character at the current index.  
\- Then increments `current` by 1 (side effect) so the next call moves forward.  
\- Centralizes consumption of a character, ensuring consistent index progression.

Summary: `isAtEnd()` is the termination test; `advance()` consumes one character (returning it) and moves the scanning position forward.

```java
    Scanner(String source) {
        this.source = source;
    }
```

It is the constructor. Purpose:

- Receives the program text to scan as a `String` parameter.
- Assigns that parameter to the instance field so later methods (`advance()`, `isAtEnd()`, `substring(...)`) can read from it.
- Uses `this.source` to distinguish the field from the parameter with the same name.
- Initialization happens once; afterwards the scanner walks through that stored string using the `start` and `current` indices.
- No access modifier means package\-private: only code in the same package can create a scanner (fine for an internal tool).
- Could optionally validate (e.g. check for `null`) but minimal constructor keeps it simple.

