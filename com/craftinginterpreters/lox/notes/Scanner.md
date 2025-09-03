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

- The start and current fields are offsets that index into the string. The start field points to the first character in the lexeme being scanned, and current points at the character currently being considered. 
- The line field tracks what source line current is on so we can produce tokens that know their location. 
- The isAtEnd() helper function tells us if we have consumed all the characters. 
- The advance() method consumes the next character in the source file and returns it. 
- Where advance() is for input, addToken() is for output. It grabs the text of the current lexeme and creates a new token for it. 

```java
    private boolean match (char expected) {
        if (isAtEnd()) return false;
        if (source.charAt(current) != expected) return false;

        current++;
        return true;
    }
```

It conditionally consumes the next character only if it matches an expected one (used for two‑character operators like \!=, ==, <=, >=).

Step by step:
1. `isAtEnd()` guard: If already at end of input, there is nothing to match; return false (do not advance).
2. Character check: Compare the current (unconsumed) character at index `current` with `expected`. If different, return false (cursor stays put).
3. On success: Increment `current` (consume that character).
4. Return true indicating the expected character was present and consumed.

Why not just call `advance()` and compare?
- Because if it does *not* match you must not consume it; later logic still needs to see that character.
- This pattern is a safe "peek and maybe advance."

Effect:
- Pointer (`current`) advances exactly one position only on a successful match.
- Leaves state unchanged on failure.
- Simplifies higher‑level code: `if (match('=')) addToken(EQUAL_EQUAL); else addToken(EQUAL);`

To Summarise:
- It's like conditional advance(). We only consume the current character if it's what we're looking for.
- Using match(), we recognize these lexemes in two stages. When we reach, for example, !, we jump to its switch case.
- That means we know the lexeme starts with !. Then we look at the next character to determine if we're on a != or merely a !.

```java
    private char peek() {
        if (isAtEnd()) return '\0';
        return source.charAt(current);
    }
```

This defines a helper method that looks at the current character without consuming it.

Syntax breakdown:
- `private` \=> visible only inside the `Scanner` class.
- `char` \=> return type is a single 16‑bit UTF‑16 code unit.
- Method name `peek` \=> conventional for non\-consuming lookahead.
- Empty parameter list `()` \=> no arguments.
- Body `{ ... }` contains two `return` paths.

Logic:
1. `if (isAtEnd()) return '\0';`  
   Calls another method to see if the cursor index `current` is at or beyond `source.length()`. If so, it returns the NUL character literal `'\0'` (used here as a sentinel indicating “no real character”).
2. Otherwise returns `source.charAt(current);`  
   Fetches but does not advance (so contrast with an `advance()` method that would increment `current`).

Why `'\0'`:
- Distinct from any valid source character (assuming typical source text) so downstream code can safely test for end.
- Avoids throwing an exception by indexing past the end.

Side effects:
- None; it does not modify `current`.

Usage:
- Supports lookahead in scanning decisions (e.g., deciding if a `.` starts a number, or whether to enter a comment) without consuming input.


It returns the single `char` located at position `current` in the `source` string.  
Details:
- `source` is the full input text.
- `current` is an integer index pointing at the next character to look at.
- The method looks (peeks) at that character without advancing the index.
- The returned value is a 16‑bit UTF‑16 code unit (Java `char`), e.g. `'a'`, `'{'`, etc.

Contrast: `advance()` both fetches the character and increments `current`; `peek()` (this method) just fetches it so calling code can decide what to do next.
- It's sort of like advance(), but doesn't consume the character. This is called lookahead. 
- Since it only looks at the current unconsumed character, we have one character of lookahead. 

```java
            case '/':
                if (match('/')) {
                    // A comment goes until the end of the line.
                    while (peek() != '\n' && !isAtEnd()) advance();
                } else {
                    addToken(SLASH);
                }
                break;
```

It distinguishes between a division operator and a single‑line comment.

- When a `/` is read, `match('/')` does a lookahead: if the next character is also `/`, both are consumed.
- In that case it is a comment start `//`. The loop advances characters until a newline `\n` or end of file, effectively skipping the comment text (no token emitted).
- If the next character is not `/`, it is just the division operator, so a `SLASH` token is added.
- `peek()` lets it look at the current (next) character without consuming; `isAtEnd()` prevents overruns.
- The `break` exits the switch after handling either path.

Line by line:

1. `case '/'`: We are handling the situation where the current character just consumed is a slash.
2. `if (match('/')) {`: Look ahead for a second slash. `match` returns true and advances if the next character is `'/'`. If true, we have started a single‑line comment (`//`).
3. Comment inside: Just a human note explaining the loop’s purpose.
4. `while (peek() != '\n' && !isAtEnd()) advance();`: Skip every character of the comment until a newline or end of file is reached.
    - `peek()` looks at the current unconsumed character without advancing.
    - `advance()` consumes characters one by one.
    - Loop stops on newline or when there is nothing left.
5. `} else {`: If there was no second slash, it was only a single `/`.
6. `addToken(SLASH);`: Emit a token representing the division operator.
7. `}`: End of the if/else block.
8. `break;`: Exit the switch so no further cases run for this character.

If there is no second /, the scanner treats the single / as the division operator and calls addToken(SLASH) to emit a SLASH token.

- This is similar to the other two-character operators, except that when we find a second /, we don't end the token yet. 
- Instead, we keep consuming characters until we reach the end of the line. 
- This is our general strategy for handling longer lexemes. After we detect the beginning of one, we shunt over to some lexeme-specific code that keeps eating characters until it sees the end. 

```java
            case ' ':
            case '\r':
            case '\t':
                break;
            case '\n':
                line++;
                break;
```

- When encountering whitespace, we simply go back to the beginning of the scan loop. That starts a new lexeme after the whitespace character. 
- For new lines we do the same thing, but we also increment the line counter. 

```java
    private void string() {
        while (peek() != '"' && !isAtEnd()) {
            if (peek() == '\n') line++;
            advance();
        }
        
        if (isAtEnd()) {
            Lox.error(line, "Unterminated string.");
            return;
        }
        
        advance();  // The closing ".
        
        // Trim the surrounding quotes. 
        String value = source.substring(start + 1, current - 1);
        addToken(STRING, value);
    }
```

Line by line:

1. `private void string() {`  
   Starts the helper that scans a string literal (already consumed opening `"`).

2. `while (peek() != '"' && !isAtEnd()) {`  
   Loop until a closing quote is found or end of source reached.

3. `if (peek() == '\n') line++;`  
   Count newline characters inside the string to keep accurate line numbers.

4. `advance();`  
   Consume the current character and move forward.

5. `}`  
   Ends the while loop.

6. `if (isAtEnd()) {`  
   If end of source reached without closing quote.

7. `Lox.error(line, "Unterminated string.");`  
   Report an unterminated string error at current line.

8. `return;`  
   Abort string scanning (do not add a token).

9. `}`  
   End of the unterminated string handling block.

10. `advance();  // The closing ".`  
    Consume the closing quote character.

11. `String value = source.substring(start + 1, current - 1);`  
    Extract the string contents excluding the opening and closing quotes.

12. `addToken(STRING, value);`  
    Emit a STRING token with the extracted literal value.



It scans a string literal after the opening quote has already been consumed.

Step by step:
1. Loop: while current char is not a closing `"` and not end of file, advance; if a newline is seen, increment line counter.
2. If end of file is reached before a closing `"`, report an unterminated string error and abort.
3. Consume the closing `"` (advance once more).
4. Extract the raw string contents (exclude the surrounding quotes) using `substring(start + 1, current - 1)`.
5. Emit a `STRING` token with that extracted value.

Purpose: turn source text between quotes into a STRING token, tracking line numbers and detecting missing closing quotes.

Why are we calling advance()?
- `advance()` consumes the current character: it returns `source.charAt(current)` and then increments `current`.  
- Inside the loop, `peek()` only looks ahead without moving. Without calling `advance()`, `current` would never progress, causing an infinite loop. So each iteration:
1. Check if current char is a closing quote or end of source.
2. If it is a newline, increment the line counter.
3. Call `advance()` to consume that character and move forward to the next one.

Purpose: sequentially skip over all characters inside the string literal until the terminating `"` (or report unterminated if EOF).


```java
            default:
                if (isDigit(c)) {
                    number();
                } else {
                    Lox.error(line, "Unexpected character.");
                }
                break;
```

What does the above code do?
- In the scanner's switch over the current character, the default branch handles any character not matched earlier. 
- It checks:
  - if the character is a digit -> it then delegates to number() to consume a full numberic literal.
  - otherwise it reports a lexical error via Lox.error starting the character is unexpected.
- Then it breaks out of the switch. 

```java
    private boolean isDigit(char c) {
        return c >= '0' && c <= '9';
    }
```
- The above code will return True if the given character lies in the ASCII rangge for digits '0' through '9'; otherwise false. 

```java
    private void number() {
        while (isDigit(peek())) advance();

        // Look for a functional part.
        if (peek() == '.' && isDigit(peekNext())) {
            // Consume the "."
            advance();

            while (isDigit(peek())) advance();
        }

        addToken(NUMBER, Double.parseDouble(source.substring(start, current)));
    }
```

Explain the above code? 
```java
while (isDigit(peek())) advance();
```
- This consumes all consecutive digit characters for the integer part.
- peek() looks ahead without consuming.
- advance() moves current forward.

```java
        if (peek() == '.' && isDigit(peekNext())) {
            advance();
```
- This means only treat a '.' as decimal point if followed by a digit
- advance() -> consumes the '.'

```java
        addToken(NUMBER, Double.parseDouble(source.substring(start, current)));
```
- This slices the source from start up to (but not including) current 
- parse it as a double 
- and emit a NUMBER token that will carry both lexeme text and numeric value. 
- In the above, substring(start, current) -> creates the exact lexeme slice (start inclusive, current exclusive)
- That slice is first parsed(DOuble.parseDouble(...)) and later (in addToken) the same indices are used again to slice for the token text.

## What does parse above mean?
- Parse means take the raw substring of characters that make up the numeric lexeme and interpret(decode) it according to the numeric grammar to produce an actual numeric value ( a double) the program can use. 
- So essentially, the scanner first slices the lexeme text (the characters)
- Double.parseDouble(...) validates that text matches the format for a floating-point literal and converts it into an internal binary IEEE-754 double. 
- Now you have both:
  - the lexeme string (for re-printing / error msgs)
  - the parsed numeric literal value (for computation)
- So to conclude, parse means -> interpret the character sequence as a number and create its structured runtime value. 

## Where does the code say consume all consecutive digit characters? 
- peek() -> looks at the current (next unconsumed) character without moving the cursor. 
- isDigit(peek()) -> tests whethere that character is a digit
- advance() -> consumes exactly one character (increments current)
- while -> repeats this as long as the next character is still a digit
- The first non-digit causes the condition to become false, exiting the loop

## Overall flow: 

1. The scanner detects a starting digit
2. Delegates to number()
3. Which consumes integer digits, optionally a fractional portion
4. Then emits a typed token
5. Unexpected non-digit characters in the default branch causes an error. 

- We consume as many digits as we find for the integer part of the literal. 
- Then we look for a fractional part, which is a decimal point (.) followed by at least one digit. 
- If we do have a fractional part, again, we consume as many digits as we can find. 
- Looking past the decimal point requires a second character of lookahead since we don't want to consume the '.' until we are sure there is a digit after it.
- So we add:

```java
    private char peekNext() {
        if (current + 1 >= source.length()) return '\0';
        return source.charAt(current + 1);
    }
```
- the above cpde performs a one-character lookahead beyond the current position without consuming input. 
- So it checks if current + 1 is past the end of the source. 
- If so, returns the sentinel '\0' (means no next character)
- Otherwise returns the character at index current + 1. 

What is the purpose of the above? 
- Supports decisions needing two characters of lookahead (e.g. detecting a fractional part after a .)
- It differs from peek() as that looks at the current unconsumed char whereas this looks at the next one. 
- Returns '\0' instead of throwinf an exception when there no next character 

What do we mean by 'if current + 1 is past the end of the source'?
- It means verify there really is another character after the one at index current
- current is the index of the next unconsumed character. 
- current + 1 would be the index of the character one position ahead
- if current + 1 >= source.length() -> that index would be outside the valid range(past the last character), so there is no next character. 
- In that case the method returns the sentinel '\0' instead of trying to access memory out of bounds. 


## What is maximal munch?
- When two lexical grammar rules can both match a chunk of code that the scanner is looking at, whichever one matches the most character wins. 
- That rule states that if we match orchid as an identifier and or as a keyword, then the former wins. 
- This is also why we assumed that <= should be scanned as a single <= token and not < followed by =

```java
    private void identifier() {
        while (isAlphaNumeric(peek())) advance();
        addToken(IDENTIFIER);
    }
```

- The above code scans an identifier (a name) starting at the current lexeme start.
- Before this method is called, start was set to the current index. 
- The while loop keeps looking at the next unconsumed char via peek().
- isAlphaNumeric(...) return true for letters, digits, or _
- As long as the above is true, advance() consumes the character and moves current forward.
- The loop stops at the first non-alphanumeric character, leaving current just past the last valid identifier char. 
- addToken(IDENTIFIER) slices the source substring from start (inclusive) to current (exclusive) and creates a token of type identifier with that text. 

### Where does the code set start to the current index? 
- It is done in the main scanning loop in scanTokens() before every token is scanned.

```java
private List<Token> scanTokens() {
    while (!isAtEnd()) {
        start = current;   // <-- start set to current here
        scanToken();       // may call identifier(), number(), string(), etc.
    }
    tokens.add(new Token(EOF, "", null, line));
    return tokens;
}
```
- So when scanToken() later detects a letter and calls identifier(), start already marks the beginning of that lexeme. 

What is the code below doing and how is it different?

```java
    private boolean isAlpha(char c) {
        return (c >= 'a' && c <= 'z') ||
                (c >= 'A' && c <= 'Z') ||
                c == '_';
    }
    
    private boolean isAlphaNumeric(char c) {
        return isAlpha(c) || isDigit(c);
    }
```

- isAlpha returns true if the character is:
  - a lower case letter ('a' - 'z')
  - an uppercase letter ('A' - 'Z')
  - an underscore ('_')
- isAlphaNumeric broadens that by also allowing digits: it calls isAlpha and (else) isDigit (which checks '0' - '9').
- Together they classify characters valid in identifiers: first char handled by isAlpha, subsequent chars by isAlphaNumeric. 

- To handle keywords, we see if the identifier's lexeme is one of the reserved words. 
- If so, we use a token type specific to that keyword. 
- We define the set of reserved words in a map. 
- Then after we can an identifier, we check to see if it matches anything in the map. 

```java
    private void identifier() {
        while (isAlphaNumeric(peek())) advance();

        String text = source.substring(start, current);
        TokenType type = keywords.get(text);
        if (type == null) type = IDENTIFIER;
        addToken(type);
    }
```

## Breakdown of what happens inside the identifier scanning logic:

Preconditions:
1. In the main loop scanTokens(), before any specific token scan starts, 
2. start is set to the current index. 
3. A leading character (already consumed by scanTokens()) was recognized as a letter or underscore
4. so the identifier routine is invoked with start pointing at the first character of the lexeme and current already one past that first character. 

Looping over the rest:
1. while (isAlphaNumeric(peek())) advance(); -> repeatedly looks ahead at the next unconsumed character (without moving) using peek(). 
2. If it is a letter, digit, or underscore -> advance() consumes it (increments current). 
3. This continues until the next character is not valid in an identifier or end of source is reached. 
4. After the loop, current is positioned just after the final identifier character. 

Extracting the raw text:
1. The substring from start (inclusive) to current (exclusive) is taken - this is the exact identifier lexeme as it appeared in source. 

Keyword check:
1. A lookup is performed in the keywords map. 
2. If the text matches a reserved word (e.g. class, for, return). the corresponding reserved TokenType is used. 
3. If the map returns null, it is treated as a user-defined name and the type defaults to IDENTIFIER. 

Token Creation:
1. addToken(type) is called
2. That method re-slices the same substring ( a tiny redundant substring operation ) and constricts a Token with:
   - the resolved TokenType
   - the lexeme text 
   - a null literal value (identifiers/keywords have no literal)
   - the current line number.

Result:
1. The new token is appended to the token list
2. Scanning then resumes from the character that terminated the loop.

### Key concepts:
1. start marks where the current token began
2. current is one past the last consumed character
3. peek() is non-consuming lookahead; advance() consumes

















