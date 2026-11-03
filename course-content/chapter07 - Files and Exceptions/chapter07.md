# Chapter Seven: Files and Exceptions

So far, everything your programs knew disappeared the moment they ended. In this chapter you will learn to **read and write text files**, so your data can outlive your program. Working with files also brings new ways for things to go wrong: the file may be missing, or its contents may not be what you expected. You will learn to **handle those errors** with exceptions, so your program can respond instead of crashing.

[← Back to Course Index](../table-of-contents.md)

---

## Chapter Goals

In this chapter you will **learn**:

- To read and write text files
- To break the lines of a file into words and fields
- To raise and handle exceptions
- To use the `with` statement so that files are always closed

---

## Chapter Contents

- **7.1 Reading and Writing Text Files** — `open`, `close`, `readline`, and `write`
- **7.2 Processing Text** — Looping over lines, `rstrip`, `split`, and records
- **7.3 Exceptions** — `try`/`except`, catching `Exception`, and `raise`
- **7.4 The `with` Statement** — Closing files automatically

---

## 7.1 Reading and Writing Text Files

### Opening and Closing a File

To use a file, you first **open** it, then work with it, then **close** it:

```python
input_file = open("input.txt", "r")
# ... read from input_file ...
input_file.close()
```

`open` takes the file name and a **mode**, and returns a **file object**. All access to the file goes through that object. A file name with no folder refers to the folder your program runs in, which in PyCharm is normally the folder containing your script.

| Mode  | Meaning | If the file exists                 | If it does not exist           |
| ----- | ------- | ---------------------------------- | ------------------------------ |
| `"r"` | Read    | Reading starts at the beginning    | Python stops with an error     |
| `"w"` | Write   | **Its contents are erased**        | A new empty file is created    |
| `"a"` | Append  | New data is added at the end       | A new empty file is created    |

Always close a file when you are done. If a program ends without closing a file it was writing to, some of the output may never reach the disk.

### Reading from a File

**`readline()`** reads one line and moves on to the next. Suppose `input.txt` contains:

```text
Monday
Tuesday
```

| Call                | Returns        |
| ------------------- | -------------- |
| First `readline()`  | `"Monday\n"`   |
| Second `readline()` | `"Tuesday\n"`  |
| Third `readline()`  | `""`           |

Notice that each line keeps its newline character `\n`. An empty string means the **end of the file**. A blank line in the file is not empty: it comes back as `"\n"`. That makes `""` a natural sentinel value for a loop:

```python
input_file = open("input.txt", "r")
line = input_file.readline()
while line != "":
    print(line)
    line = input_file.readline()
input_file.close()
```

Lines from a file are always **strings**. If a line holds a number, convert it with `int()` or `float()`. Both ignore the newline at the end:

```python
value = float(line)
```

### Writing to a File

Open the file in mode `"w"` or `"a"` and call **`write()`**. Unlike `print`, `write` does **not** start a new line for you, so you add the `\n` yourself:

```python
output_file = open("output.txt", "w")
output_file.write("Hello, World!\n")
output_file.write(f"Total: {total:.2f}\n")
output_file.close()
```

### Example: Reading and Writing

This program reads a file with one number per line. It writes the numbers to a new file, lined up in a column, followed by their total and average.

**`numbers.txt`:**

```text
32.0
54.0
67.5
80.25
115.0
```

**The program:**

```python
input_file = open("numbers.txt", "r")
output_file = open("report.txt", "w")

total = 0.0
count = 0

line = input_file.readline()
while line != "":
    value = float(line)
    output_file.write(f"{value:8.2f}\n")
    total = total + value
    count = count + 1
    line = input_file.readline()

if count > 0:
    average = total / count
    output_file.write("--------\n")
    output_file.write(f"{total:8.2f} total\n")
    output_file.write(f"{average:8.2f} average\n")

input_file.close()
output_file.close()
```

**`report.txt` after the program runs:**

```text
   32.00
   54.00
   67.50
   80.25
  115.00
--------
  348.75 total
   69.75 average
```

### Common Error: Backslashes in File Paths

In a string, a backslash starts an escape sequence such as `\n`. To write a Windows path in your code, double every backslash:

```python
input_file = open("C:\\homework\\input.txt", "r")
```

A user who types a path at an `input()` prompt types single backslashes as usual.

---

## 7.2 Processing Text

### Looping Over the Lines

A `for` loop can go through a file line by line, which is shorter than calling `readline()` yourself:

```python
input_file = open("input.txt", "r")
for line in input_file:
    print(line)
input_file.close()
```

Run this and you will see a blank line after each line of text: one newline comes from the file and another from `print`.

### Removing the Newline: `rstrip`

**`rstrip()`** returns a copy of a string with the whitespace at its right end removed, including the newline:

```python
line = line.rstrip()
```

There are three of these methods. Each can also be given the characters to remove:

| Method      | Removes from       | Example                       | Result       |
| ----------- | ------------------ | ----------------------------- | ------------ |
| `strip()`   | Both ends          | `"  Yes! \n".strip()`         | `"Yes!"`     |
| `lstrip()`  | The left end only  | `"  Yes!".lstrip()`           | `"Yes!"`     |
| `rstrip()`  | The right end only | `"Yes!\n".rstrip()`           | `"Yes!"`     |
| `rstrip(chars)` | Any of `chars`, from the right end | `"lamb,".rstrip(".,?!")` | `"lamb"` |

### Breaking a Line into Words: `split`

**`split()`** breaks a string into a **list** of words. With no argument it splits on whitespace. Given a separator, it splits on that instead:

| Example                       | Result                  |
| ----------------------------- | ----------------------- |
| `"Mary had a lamb".split()`   | `['Mary', 'had', 'a', 'lamb']` |
| `"China:1330044605".split(":")` | `['China', '1330044605']` |

### Example: Reading Words

Suppose `lyrics.txt` contains:

```text
Mary had a little lamb,
whose fleece was white as snow.
```

This program prints each word on its own line, without punctuation:

```python
input_file = open("lyrics.txt", "r")
for line in input_file:
    word_list = line.rstrip().split()
    for word in word_list:
        print(word.rstrip(".,?!"))
input_file.close()
```

```text
Mary
had
a
little
lamb
whose
fleece
was
white
as
snow
```

### Example: Reading Records

A file often stores **records**, each made up of several **fields**. A common layout is one record per line, with a **delimiter** between the fields. Suppose `countries.txt` contains:

```text
China:1330044605
India:1147995898
United States:303824646
```

Read each line, then split it on the delimiter:

```python
total_population = 0

input_file = open("countries.txt", "r")
for line in input_file:
    fields = line.rstrip().split(":")
    country_name = fields[0]
    population = int(fields[1])
    print(f"{country_name:15}{population:12}")
    total_population = total_population + population
input_file.close()

print(f"{'Total':15}{total_population:12}")
```

```text
China            1330044605
India            1147995898
United States     303824646
Total            2781865149
```

---

## 7.3 Exceptions

### What Is an Exception?

You have already seen programs stop with an error. Try to open a file that does not exist, or convert text that is not a number:

```text
FileNotFoundError: [Errno 2] No such file or directory: 'missing.txt'
```

```text
ValueError: invalid literal for int() with base 10: '35x2'
```

These are **exceptions**. An exception signals that something went wrong and normal execution cannot continue. Unless your program handles it, the program stops. A few you will meet often:

| Exception           | Typical cause                                   |
| ------------------- | ----------------------------------------------- |
| `FileNotFoundError` | Opening a file for reading that does not exist  |
| `ValueError`        | A value has the wrong form, as in `int("35x2")` |
| `IndexError`        | A list index is out of range                    |
| `ZeroDivisionError` | Dividing by zero                                |

### Handling Exceptions: `try` and `except`

Put the code that might fail in a **`try`** block, and what to do about the failure in an **`except`** block:

```python
try:
    age = int(input("Enter your age: "))
    print(f"Next year you will be {age + 1}.")
except ValueError:
    print("That is not a whole number.")
```

```text
Enter your age: twenty
That is not a whole number.
```

As soon as a line in the `try` block raises an exception, the **rest of the block is skipped** and Python jumps to the matching `except`. Here, the second `print` never runs. If nothing goes wrong, the `except` block is skipped instead. Either way, the program carries on with whatever comes next.

### Which Exception Should You Catch?

You do not need to memorize exception names, and you do not need to predict every possible error. There are two practical ways to find the name.

**Read the error message.** When your program crashes while you are testing it, the last line names the exception. Here is the program above without the `try`, after a user enters bad input:

```text
Enter your age: twenty
Traceback (most recent call last):
  File "C:\homework\age.py", line 1, in <module>
    age = int(input("Enter your age: "))
ValueError: invalid literal for int() with base 10: 'twenty'
```

The word before the colon, `ValueError`, is the name to put after `except`. For errors that are easy to cause, such as bad input or a missing file, you can try them on purpose to see the name.

**Check the documentation.** Some errors are hard to cause on purpose, such as a lost network connection. For those, look up the function you are calling, because its documentation usually says what it can raise. The Python documentation also [lists all the built-in exceptions](https://docs.python.org/3/library/exceptions.html).

For the errors you expect, catch the specific exception, so that you can give the user a helpful response. For everything else, there is a catch-all, covered below.

### Handling Several Kinds of Exception

A `try` can have one `except` for each kind of problem. To see the message that came with the exception, name it with **`as`**:

```python
file_name = input("Enter the file name: ")

try:
    input_file = open(file_name, "r")
    value = int(input_file.readline())
    input_file.close()
    print(f"The value is {value}.")
except FileNotFoundError:
    print("Error: file not found.")
except ValueError as e:
    print(f"Error: {e}")
```

If the first line of the file is `35x2`, this prints:

```text
Error: invalid literal for int() with base 10: '35x2\n'
```

### Catching Everything Else: `Exception`

**`Exception`** matches almost any error. Put `except Exception as e` **last**, after the specific `except` blocks, to catch whatever you did not expect. Include `e` in the message, so the user (and you) can still see what went wrong:

```python
try:
    total = float(input("Total bill: "))
    people = int(input("Number of people: "))
    print(f"Each person pays {total / people:.2f}.")
except ValueError:
    print("Please enter numbers only.")
except Exception as e:
    print(f"Unexpected error: {e}")
```

```text
Total bill: 90
Number of people: 0
Unexpected error: float division by zero
```

A zero is not really unexpected. The programmer simply forgot about it, and the catch-all kept the program from crashing. Once you notice a case like this, give it its own `except` (here, `except ZeroDivisionError:`) with a clearer message.

Python checks the `except` blocks from top to bottom and runs the **first** one that matches. That is why `Exception` must come last: if it came first, it would catch the `ValueError` too, and the helpful message would never be shown.

**Watch out:** the catch-all also catches mistakes in **your own code**. If you misspell `total` as `totl`, the program above prints `Unexpected error: name 'totl' is not defined` instead of an error that points to the line. If you see an unexpected error while testing, temporarily comment out the `except Exception` block to get the full error message with its line number.

### Raising Your Own Exceptions

Your own functions can signal a problem with **`raise`**. Give the exception a message that says what went wrong:

```python
def withdraw(balance, amount):
    if amount > balance:
        raise ValueError("Amount exceeds balance")
    return balance - amount


def main():
    try:
        new_balance = withdraw(100, 500)
        print(f"New balance: {new_balance}")
    except ValueError as e:
        print(f"Error: {e}")


main()
```

```text
Error: Amount exceeds balance
```

`withdraw` detects the problem but does not know what the program should do about it, so it raises an exception. Execution leaves the function immediately and continues in the `except` block of `main`, which decides how to respond. That is the usual pattern: **raise where you detect a problem, and catch where you can do something about it.**

---

## 7.4 The `with` Statement

Look again at the `try` block from §7.3:

```python
try:
    input_file = open(file_name, "r")
    value = int(input_file.readline())
    input_file.close()
```

There is a problem here. If `int` raises an exception, the rest of the block is skipped, and that includes `close()`. The file is left open.

The **`with`** statement solves this. It opens the file, and closes it automatically when the indented block ends, even if an exception occurred:

```python
try:
    with open(file_name, "r") as input_file:
        value = int(input_file.readline())
    print(f"The value is {value}.")
except FileNotFoundError:
    print("Error: file not found.")
except ValueError as e:
    print(f"Error: {e}")
```

Notice that there is no call to `close()`. Now that you know why it exists, use `with` whenever you work with a file.

### Example: Summing a File, with Retry

This program puts the chapter together. It reads numbers from a file and prints their total. If anything is wrong with the file, the user gets another chance:

```python
def read_numbers(file_name):
    """Returns the list of numbers in a file with one number per line."""
    numbers = []
    with open(file_name, "r") as input_file:
        for line in input_file:
            numbers.append(float(line))

    if len(numbers) == 0:
        raise ValueError("The file is empty.")
    return numbers


def main():
    done = False
    while not done:
        file_name = input("Enter the file name: ")
        try:
            numbers = read_numbers(file_name)
            print(f"The total is {sum(numbers):.2f}.")
            done = True
        except FileNotFoundError:
            print("Error: file not found. Please try again.")
        except ValueError as e:
            print(f"Error: {e}")
        except Exception as e:
            print(f"Unexpected error: {e}")


main()
```

A sample run, where `letters.txt` contains the line `abc` and `empty.txt` has nothing in it:

```text
Enter the file name: numbrs.txt
Error: file not found. Please try again.
Enter the file name: letters.txt
Error: could not convert string to float: 'abc\n'
Enter the file name: empty.txt
Error: The file is empty.
Enter the file name: numbers.txt
The total is 348.75.
```

---

## Key Takeaways

1. **`open(name, mode)`** returns a file object. The mode is `"r"` to read, `"w"` to write (which **erases** an existing file), or `"a"` to append. Close the file when you are done
2. **`readline()`** reads one line and returns `""` at the end of the file; **`for line in file`** does the same job more simply. Lines are strings and keep their `\n`
3. **`write()`** writes a string and does not add a newline for you
4. **`rstrip()`** removes the newline, **`split()`** breaks a line into a list of words or fields, and `int()` and `float()` convert text to numbers
5. An **exception** stops the program unless it is handled. Put risky code in a **`try`** block and the response in an **`except`** block; `except ValueError as e` gives you the message. End with **`except Exception as e`** to catch anything unexpected, and include `e` in its message
6. Use **`raise`** to signal a problem your function cannot fix. Raise where you detect the problem, and catch where you can respond to it
7. **`with open(name, mode) as file:`** closes the file automatically, even if an exception occurs. Prefer it whenever you work with a file

---

*End of Chapter Seven*

[← Back to Course Index](../table-of-contents.md)
