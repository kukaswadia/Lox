    private static void runFile(String path) throws IOException {
        byte[] bytes = Files.readAllBytes(Paths.get(path));
        run(new String(bytes, Charset.defaultCharset()));
    }

This line reads the entire contents of a file into a byte array.

- `Paths.get(path)`: Converts the string `path` into a `Path` object, representing the file location.
- `Files.readAllBytes(...)`: Reads all bytes from the file at the given `Path` and returns them as a `byte[]`.
- `byte[] bytes = ...`: Declares a byte array named `bytes` and assigns the result.

Syntax breakdown:
- `byte[]`: Array of bytes.
- `=`: Assignment operator.
- `Files.readAllBytes(...)`: Static method from `java.nio.file.Files`.
- `Paths.get(path)`: Static method from `java.nio.file.Paths`.

This is useful for loading the entire file into memory for further processing.


        for (;;) {
            System.out.print("> ");
            String line = reader.readLine();
            if (line == null) break;
            run(line);
        }

This is an infinite loop that repeatedly prompts the user for input, reads a line, and processes it.

- `for (;;) { ... }`: A `for` loop with no initialization, condition, or increment, so it runs forever.
- `System.out.print("> ");`: Prints a prompt to the console.
- `String line = reader.readLine();`: Reads a line of input from the user.
- `if (line == null) break;`: If the input is `null` (end of input), the loop exits.
- `run(line);`: Processes the input line.

This pattern is common for interactive command-line programs.

    private static void run(String source) {
        Scanner scanner = new Scanner(source);
        List<Token> tokens = scanner.scanTokens();

        // print tokens
        for (Token token : tokens) {
            System.out.println(token);
        }
    }

This function processes a string of source code and prints out its tokens.

**Syntax breakdown:**
- `private static void run(String source)`: Declares a private static method named `run` that takes a `String` parameter called `source` and returns nothing (`void`).
- `Scanner scanner = new Scanner(source);`: Creates a new `Scanner` object, passing the source code string to its constructor.
- `List<Token> tokens = scanner.scanTokens();`: Calls `scanTokens()` on the scanner, which returns a list of `Token` objects.
- `for (Token token : tokens) { ... }`: Enhanced for-loop that iterates over each token in the list.
- `System.out.println(token);`: Prints each token to the console.

**What it does:**
1. Takes source code as input.
2. Scans the source code to break it into tokens.
3. Prints each token to the console.

You know it's returning a list of `Token` objects because of this line:

```java
List<Token> tokens = scanner.scanTokens();
```

- `List<Token>` means a list (collection) of `Token` objects.
- The method `scanTokens()` is expected to return that type, as shown by the assignment.

The syntax `List<Token>` explicitly says "a list of objects of type `Token`".


```java
for (Token token : tokens);
```

This is called an "enhanced for loop" or "for-each loop" in Java.

**Syntax explanation:**
- `for (Token token : tokens)`
    - `Token` is the type of each element in the collection.
    - `token` is a variable that holds the current element.
    - `tokens` is the collection (like a `List<Token>`) you are looping through.
- The loop runs once for each element in `tokens`, assigning each element to `token` in turn.

**Purpose:**  
It lets you easily iterate over all elements in a collection without using an index.

    static void error(int line, String message) {
        report(line, "", message);
    }
    
    private static void report(int line, String where, String message) {
        System.out.println(
                "[line " + line + "] Error" + where + ": " + message);
        hadError = true;
    }

The `error()` function is used to notify the user that a syntax error occurred at a specific line. 
It calls the `report()` helper, which formats and prints the error message to the console 
and sets the `hadError` flag to `true` to indicate an error has occurred.

The `error` method is accessible within the package, while `report` is only accessible inside the same class.

The difference is in the access modifier:

- `static void error(...)`: No access modifier, so it is **package-private**. Only code in the same package can call it.
- `private static void report(...)`: Has the `private` modifier, so it is **private**. Only code inside the same class can call it.

Both are `static`, meaning they belong to the class, not an instance. The `void` means they do not return a value.

`private static void report(...)` is private because it is meant to be used only within the `Lox` class, hiding its implementation details from other classes.

`static void error(...)` is package-private (no modifier) so it can be called from other classes in the same package, allowing error reporting from outside the `Lox` class.

This is a way to control which methods are accessible from where, following encapsulation principles.

        hadError = true;

The first line checks if an error occurred (hadError is true). If so, it exits the program with status code 65.

            hadError = false;

The second line resets the hadError flag to false, clearing the error state for future runs.
Resetting `hadError` to `false` in the interactive loop ensures that a user mistake does not terminate the session. This allows the REPL to continue accepting input after an error, rather than exiting the program.

REPL stands for "Read-Eval-Print Loop." It is an interactive programming environment that:

- **Reads** user input (code or commands)
- **Evaluates** the input (executes the code)
- **Prints** the result or output
- **Loops** back to read the next input

REPLs allow users to quickly test code and see results immediately, 
commonly used in languages like Python, JavaScript, and interactive shells.