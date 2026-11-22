# Chapter Nine: Lambda and DynamoDB

In Chapter 6 your Python programs ran on **your** computer and used S3 to store files in the cloud. In this chapter the code itself moves to the cloud. **AWS Lambda** runs a Python function for you on AWS: no computer of yours has to be on, and you never set up a server. **Amazon DynamoDB** is a database that stores **items** — records that look just like Python dictionaries — and finds any one of them by its key in an instant. Along the way you will learn **JSON**, the text format that Lambda, DynamoDB, and almost every web service use to send dictionaries from one program to another. By the end of the chapter you will have a Lambda function that looks up products in a DynamoDB table, and a program on your computer that calls it.

[← Back to Course Index](../table-of-contents.md)

---

## Chapter Goals

In this chapter you will **learn**:

- To read and write **JSON**, and to convert between dictionaries and JSON text with `json.dumps` and `json.loads`
- To create an **AWS Lambda** function in the console
- To develop a Lambda function in PyCharm, test it locally with a payload, and upload it as a `.zip` file
- To create a **DynamoDB** table with a partition key
- To add, read, change, list, and delete DynamoDB items from Python
- To use DynamoDB from inside a Lambda function

---

## Chapter Contents

- **9.1 JSON: Dictionaries as Text** — What JSON is, its values and rules, `json.dumps`, `json.loads`, and saving JSON in a file
- **9.2 AWS Lambda** — Serverless functions, the handler, creating a function, and environment variables
- **9.3 Developing a Function on Your Computer** — Testing locally with a payload, packaging (with and without external libraries), uploading, and logs
- **9.4 Amazon DynamoDB** — Tables, items, attributes, and the partition key
- **9.5 DynamoDB from Python** — `put_item`, `get_item`, `scan`, `delete_item`, and `Decimal`
- **9.6 Lambda and DynamoDB Together** — A Lambda function that reads from a table

---

## 9.1 JSON: Dictionaries as Text

### What Is JSON?

A dictionary lives in your program's memory. When the program ends, it is gone, and another program — on another computer, perhaps written in another language — cannot reach into your memory to read it. To save a dictionary or send it somewhere, you first have to turn it into **text**.

**JSON** (JavaScript Object Notation, pronounced "jay-son") is the most widely used text format for this. It started in JavaScript, but today every programming language can read and write it. You will meet it everywhere:

- **Web services and APIs** send their answers as JSON.
- **AWS** uses it for almost everything. You have already seen it: in §6.3 you chose `json` as the AWS CLI's output format, and `aws sts get-caller-identity` printed its answer as JSON.
- **Lambda** (§9.2) receives its input as JSON and sends its result back as JSON, and the SQS messages in Chapter 10 are JSON too.
- **Settings files** of many programs are written in JSON.

Here is an order written in JSON:

```json
{
    "customer": "Emma",
    "product_id": "P300",
    "quantity": 2,
    "price": 24.0,
    "paid": true,
    "note": null,
    "tags": ["office", "urgent"]
}
```

If that looks like a Python dictionary, that is no accident: JSON was designed to look like the way programming languages write their data. But it is **text** — a string of characters that you can print, save in a file, or send across the internet.

### The Building Blocks

A JSON text is built from six kinds of **values**:

| JSON value                | Example                    | Becomes in Python   |
| ------------------------- | -------------------------- | ------------------- |
| **Object**                | `{"customer": "Emma"}`     | A dictionary        |
| **Array**                 | `["office", "urgent"]`     | A list              |
| **String**                | `"Emma"`                   | A `str`             |
| **Number**                | `2`, `24.0`, `-3.5`        | An `int` or a `float` |
| **`true`** / **`false`**  | `true`                     | `True` / `False`    |
| **`null`**                | `null`                     | `None`              |

An **object** is a list of `"key": value` pairs inside braces, separated by commas — a dictionary. An **array** is a list of values inside square brackets. Objects and arrays can hold any values, including other objects and arrays, so JSON can describe data that is **nested**, just as a Python dictionary can hold a list.

### The Rules

JSON is stricter than Python. These are the rules that trip people up:

| Rule                                                         | Valid JSON                   | Not JSON                     |
| ------------------------------------------------------------ | ---------------------------- | ---------------------------- |
| Strings use **double** quotes only                           | `"Emma"`                     | `'Emma'`                     |
| Keys of an object must be **strings**                        | `{"101": "Emma"}`            | `{101: "Emma"}`              |
| `true`, `false`, and `null` are written in **lower case**    | `true`                       | `True`                       |
| No comma after the **last** item                             | `[1, 2, 3]`                  | `[1, 2, 3,]`                 |
| No comments                                                  |                              | `{"quantity": 2}  # two`     |

Spaces and line breaks between the values do not matter. The order above could just as well be written on one line, and that is how programs usually send it. Lining it up over several lines, as above, only makes it easier for **people** to read.

### Dictionary to Text: `json.dumps`

The `json` module comes with Python, so there is nothing to install. **`json.dumps`** ("dump to string") turns a dictionary into a JSON string:

```python
import json

order = {
    "customer": "Emma",
    "product_id": "P300",
    "quantity": 2,
    "price": 24.0,
    "paid": True,
    "note": None,
    "tags": ["office", "urgent"],
}

text = json.dumps(order)
print(text)
print(type(text))
```

```text
{"customer": "Emma", "product_id": "P300", "quantity": 2, "price": 24.0, "paid": true, "note": null, "tags": ["office", "urgent"]}
<class 'str'>
```

Look at what changed: `True` became `true`, `None` became `null`, and the result is one long **string**, not a dictionary. Python followed the JSON rules for you.

To make the text easier for people to read, add **`indent`**, the number of spaces to indent each level:

```python
print(json.dumps(order, indent=4))
```

```text
{
    "customer": "Emma",
    "product_id": "P300",
    "quantity": 2,
    "price": 24.0,
    "paid": true,
    "note": null,
    "tags": [
        "office",
        "urgent"
    ]
}
```

A dictionary key that is not a string is turned into one: `json.dumps({101: "Emma"})` gives `{"101": "Emma"}`. A tuple becomes a JSON array.

### Text to Dictionary: `json.loads`

**`json.loads`** ("load from string") does the reverse. It reads JSON text and builds a new dictionary from it:

```python
order_copy = json.loads(text)
print(order_copy)
print(order_copy["tags"][0])
print(order_copy["price"] * order_copy["quantity"])
```

```text
{'customer': 'Emma', 'product_id': 'P300', 'quantity': 2, 'price': 24.0, 'paid': True, 'note': None, 'tags': ['office', 'urgent']}
office
48.0
```

The result is an ordinary dictionary again. The numbers come back as numbers, so you can do arithmetic with them, and the array comes back as a list, so `order_copy["tags"][0]` is its first element.

The two functions are opposites. Remember them by the **`s`** at the end, which stands for **string**: `dumps` makes a string, `loads` reads one.

### Saving JSON in a File

Because JSON is text, you can write it to a file and read it back later with what you learned in Chapter 7. Write the whole dictionary on **one line**, and one `readline()` reads it back:

```python
with open("order.json", "w") as output_file:
    output_file.write(json.dumps(order))

with open("order.json", "r") as input_file:
    saved_order = json.loads(input_file.readline())

print(saved_order["customer"])   # Emma
```

The dictionary survived the end of the program: run a second program tomorrow, and it can read `order.json` and get the same dictionary back. Open `order.json` in PyCharm to see the JSON text inside it.

### ⚠️ Common Error: Text That Is Not JSON

If the text is not valid JSON, `json.loads` raises a `json.JSONDecodeError`:

```python
json.loads("customer: Emma")
```

```text
json.decoder.JSONDecodeError: Expecting value: line 1 column 1 (char 0)
```

Single quotes are a common cause, because they are fine in Python but not in JSON:

```python
json.loads("{'customer': 'Emma'}")
```

```text
json.decoder.JSONDecodeError: Expecting property name enclosed in double quotes: line 1 column 2 (char 1)
```

`JSONDecodeError` is a kind of **`ValueError`**, so `except ValueError as e:` catches it (§7.3).

---

## 9.2 AWS Lambda

### What Is Lambda?

**AWS Lambda** runs **one Python function** in the cloud whenever something asks it to. You give Lambda your function, and AWS does everything else: it finds a computer, starts Python, runs your function, and shuts it all down again.

This is called **serverless** computing. There are still servers, but they are AWS's problem, not yours:

- **No setup.** You never install Python or boto3 on a server; Lambda already has both.
- **Pay per use.** You pay only while your function runs, measured in milliseconds. When nobody calls it, it costs nothing.
- **Scales by itself.** If a thousand requests arrive at once, Lambda runs a thousand copies of your function.
- **Short-lived.** Each run starts fresh and should finish within seconds (the default limit is 3 seconds, the maximum 15 minutes). A Lambda function does not remember anything between runs, so anything it must keep goes into storage such as S3 or DynamoDB.

> **Note:** AWS Lambda has nothing to do with Python's `lambda` keyword, which you may see in other courses. In this course, "Lambda" always means the AWS service.

### The Handler

A Lambda function is a Python file with one special function in it, the **handler**. Lambda calls the handler every time the function runs:

```python
def lambda_handler(event, context):
    name = event.get("name", "world")
    message = f"Hello, {name}!"
    print(message)
    return {"message": message}
```

| Part       | Meaning                                                                                       |
| ---------- | --------------------------------------------------------------------------------------------- |
| `event`    | The **input**. Lambda turns the JSON it receives into a **dictionary** and passes it in.      |
| `context`  | Information about this run (time left, request id). You will not need it in this course, but the parameter must be there. |
| `return`   | The **output**. Lambda turns the value you return into JSON and sends it back to the caller.  |
| `print`    | Goes to the function's **log** (see below), not to the caller.                               |

Notice what is **missing**: there is no `main()` and no call to `lambda_handler` at the bottom of the file. **Lambda** calls the handler; your file only defines it.

### Creating a Function

Before you write any code, create the function in the console. It will hold the code you upload in §9.3.

1. Start your AWS Academy lab and open the **AWS Console** (§6.3).
2. Search for **Lambda** and open it. Check that the region at the top right is **N. Virginia (us-east-1)**.
3. Click **Create function** and choose **Author from scratch**.
4. **Function name:** `hello`
5. **Runtime:** the newest **Python 3** version in the list (for example, Python 3.13).
6. Open **Change default execution role**, choose **Use an existing role**, and select **`LabRole`**.
7. Click **Create function**.

> **Why `LabRole`?** The **execution role** decides which AWS services your function may use. AWS Academy does not let you create roles, so every Lambda function in this course uses `LabRole`, which already allows Lambda to write logs and to use S3, DynamoDB, and SQS. If you leave the default, creating the function fails with a permissions error.

The page that opens has a **Code** tab with a file named `lambda_function.py`, holding a small sample handler. You will replace it with your own code in §9.3.

Lambda finds the handler through the **Handler** setting (under **Runtime settings** on the Code tab). The default, `lambda_function.lambda_handler`, means "the function `lambda_handler` in the file `lambda_function.py`". Keep both names as they are, and use the same names on your computer.

### Environment Variables in Lambda

A Lambda function can have environment variables, just like a PyCharm run configuration (§6.1). Use them for the same things: names of buckets and tables, and settings that differ between copies of a function.

1. Open the **Configuration** tab and choose **Environment variables**.
2. Click **Edit**, then **Add environment variable**, and enter a key and a value.
3. Click **Save**.

The code reads them with `os.environ`, exactly as before. You will use this in §9.6.

> **No credentials in Lambda.** Your function does not use your `credentials` file — it is not on your computer. When boto3 runs inside Lambda, it gets credentials from the execution role, `LabRole`, automatically. Do **not** put keys in Lambda code or environment variables.

---

## 9.3 Developing a Function on Your Computer

### The Workflow

You could type your code straight into the browser, but PyCharm is a much better place to write it: it points out mistakes as you type, and when something goes wrong you get the full error with its line number. So you write and test the function **on your computer**, and copy it to Lambda only when it works:

1. **Write** the handler in a file named `lambda_function.py`.
2. **Test** it locally with a small program that calls the handler with a **test payload** — a dictionary that plays the part of `event`.
3. **Package** `lambda_function.py` into a `.zip` file.
4. **Upload** the `.zip` file to your Lambda function.
5. **Test** it once more in Lambda, with the same payload.

When you change the code, you repeat steps 2 to 5.

### Step 1: Write the Handler

Create a new PyCharm project named `hello`. In it, create a file named exactly **`lambda_function.py`** and type in the `hello` handler from §9.2:

```python
def lambda_handler(event, context):
    name = event.get("name", "world")
    message = f"Hello, {name}!"
    print(message)
    return {"message": message}
```

If you run this file, nothing happens: it only **defines** `lambda_handler` and never calls it. On AWS, Lambda calls it. On your computer, a test program will.

### Step 2: Write a Test Program

In the same project, create **`test_lambda.py`**:

```python
"""Runs the Lambda handler on this computer with test payloads."""

import json

from lambda_function import lambda_handler


def run_test(event):
    """Calls the handler with one event and prints what it returns, as JSON."""
    print(f"Event:    {json.dumps(event)}")
    response = lambda_handler(event, None)
    print(f"Response: {json.dumps(response)}")
    print()


def main():
    run_test({"name": "Emma"})
    run_test({})


main()
```

- **`from lambda_function import lambda_handler`** works just like `from math import sqrt` (§1.5), except that the module is **your own file**. Python finds `lambda_function.py` because it is in the same folder as `test_lambda.py`. Importing a file runs its top-level code; since `lambda_function.py` only defines a function, nothing else happens.
- Each **test payload** is a dictionary: the same dictionary Lambda will pass as `event` when it receives this JSON.
- The handler never uses `context`, so the test passes **`None`** in its place.
- **`json.dumps(response)`** does what Lambda does with your return value: it turns it into JSON. If the handler returns something that cannot become JSON, you find out here, on your computer, instead of on AWS (§9.6).

### Step 3: Run the Tests

Run **`test_lambda.py`** (not `lambda_function.py`):

```text
Event:    {"name": "Emma"}
Hello, Emma!
Response: {"message": "Hello, Emma!"}

Event:    {}
Hello, world!
Response: {"message": "Hello, world!"}
```

The line `Hello, Emma!` comes from the `print` in the handler. Because the handler uses `event.get("name", "world")`, the empty payload gets `Hello, world!` instead of a `KeyError`.

Fix and re-run until every test gives the response you expect. This is quick: no uploading, no browser, and no lab session needed for a function that does not use AWS.

### Step 4: Package the Function

Lambda takes your code as a **`.zip` file** with `lambda_function.py` at the top level. The test program is only for your computer, so leave it out.

- **In File Explorer:** right-click `lambda_function.py` (in your project folder) and choose **Compress to ZIP file** (on Windows 10: **Send to → Compressed (zipped) folder**). Name the zip `lambda_function.zip`.
- **In PyCharm's terminal (PowerShell):**

  ```text
  Compress-Archive -Path lambda_function.py -DestinationPath lambda_function.zip -Force
  ```

  `-Force` replaces the zip if it already exists, so you can run the same command after every change.

Zip the **file**, not the project folder. If the zip contains a folder with `lambda_function.py` inside it, Lambda cannot find the handler (see the common errors below).

> **macOS / Linux:** `zip lambda_function.zip lambda_function.py`

#### Packaging a Function That Uses External Libraries

Lambda's Python has the **standard library** (`json`, `os`, `math`, …) and **boto3**, and nothing else. Your project's `.venv` (§6.2) stays on your computer; it is not uploaded. So if your handler imports an external library that Lambda does not have, the library must travel **inside the zip**, next to `lambda_function.py`.

**boto3 is the exception.** It is already installed in Lambda, so do **not** put it in the zip. A function that uses only boto3 — like the one in §9.6 — is packaged exactly as above: one file. You still `pip install boto3` in PyCharm, but only so that you can test on your computer. (Packaging boto3 anyway would work, but it makes the zip much bigger, and the console stops showing your code once a zip is larger than about 3 MB.)

Here is a function that does need an extra library. It uses **`requests`** (§6.2) to check whether a website is up:

```python
import requests


def lambda_handler(event, context):
    url = event.get("url", "https://www.python.org")
    response = requests.get(url, timeout=2)
    print(f"{url} answered with status {response.status_code}")
    return {"url": url, "status_code": response.status_code}
```

`timeout=2` makes `requests` give up after 2 seconds, inside Lambda's 3-second limit. Develop it in a new PyCharm project named `site_check`:

1. **Install the library for testing.** In PyCharm's terminal: `pip install requests`. Write `lambda_function.py` and a `test_lambda.py` with `run_test({})` and test it as in steps 2 and 3. The response is `{"url": "https://www.python.org", "status_code": 200}`.

2. **Install the library again, into a `package` folder.** In PyCharm's terminal, in the project folder:

   ```text
   pip install requests -t package
   ```

   **`-t package`** ("target") tells pip to put the library and its dependencies into a folder named `package` instead of into `.venv`. pip creates the folder, and you see it appear in the project.

3. **Copy your handler into the same folder:**

   ```text
   copy lambda_function.py package
   ```

4. **Zip what is inside `package`** — its contents, not the folder itself:

   - **In File Explorer:** open the `package` folder, select everything in it (**Ctrl+A**), right-click, and choose **Compress to ZIP file** (Windows 10: **Send to → Compressed (zipped) folder**). Name it `lambda_function.zip`, then move the zip out of `package`, into the project folder.
   - **In PowerShell:**

     ```text
     Compress-Archive -Path package\* -DestinationPath lambda_function.zip -Force
     ```

     `package\*` means "everything inside `package`".

The zip must look like this, with `lambda_function.py` and the libraries side by side at the **top level**:

```text
lambda_function.zip
├── lambda_function.py      ← your handler
├── requests/               ← the library you installed
├── urllib3/                ← libraries that requests itself needs
├── certifi/
├── charset_normalizer/
├── idna/
└── ...-dist-info/          ← pip's notes about each library
```

pip installed four libraries you never asked for: `requests` needs them (its **dependencies**, §6.2). That is why you let pip fill the folder instead of copying `requests` by hand. Upload the zip as in step 5, and test with the event `{}`.

**When you change your code**, you only repeat steps 3 and 4: copy `lambda_function.py` into `package` again and re-zip. The libraries are already there. Do not edit the copy inside `package` — always edit the one in your project, test it, then copy it.

If your project has a **`requirements.txt`** (§6.2), install everything in it into the folder in one go: `pip install -r requirements.txt -t package`. Leave `boto3` out of the file for Lambda, or simply delete it from `package` before zipping.

> **Libraries with compiled code.** Lambda runs on **Linux**, but pip installs the **Windows** version of a library. For libraries written purely in Python, such as `requests`, the two are the same. Some libraries — for example `numpy` and `pandas` — contain compiled code that only runs on the system it was built for, and the function fails with an `ImportModuleError` naming the library. For those, ask pip for the Linux version, matching your function's Python version:
>
> ```text
> pip install numpy -t package --platform manylinux2014_x86_64 --only-binary=:all: --python-version 3.13
> ```

### Step 5: Upload It to Lambda

1. Open your `hello` function and its **Code** tab.
2. Click **Upload from → .zip file**, choose `lambda_function.zip`, and click **Save**.

The code editor now shows your `lambda_function.py`. Uploading a zip **deploys** it right away, so the function now runs your code.

> **Alternative: copy and paste.** For a function that is a single file, you can also copy the contents of `lambda_function.py`, paste them over the code in the browser editor, and click **Deploy**. Editing in the browser does **not** change the function until you click Deploy. The zip is the safer habit: what you upload is exactly the file you tested.

### Step 6: Test It in Lambda

Check that it works on AWS too, with the same payload:

1. Click **Test** (or open the **Test** tab).
2. Choose **Create new event**, name it `emma`, and replace the sample JSON with the payload, written as JSON (§9.1):

   ```json
   {"name": "Emma"}
   ```

3. Save it, then click **Test** again.

The result shows the **response**, which is the same as on your computer:

```json
{
  "message": "Hello, Emma!"
}
```

and the **function logs**, which contain everything you printed:

```text
START RequestId: 6b8e1c2d-... Version: $LATEST
Hello, Emma!
END RequestId: 6b8e1c2d-...
REPORT RequestId: 6b8e1c2d-...  Duration: 1.42 ms  Billed Duration: 2 ms  Memory Size: 128 MB  Max Memory Used: 32 MB
```

`START`, `END`, and `REPORT` are written by Lambda itself. Your ids and times will differ.

### Logs in CloudWatch

The test page shows the logs of the run you just started. Every run, however it was started, also writes its logs to **Amazon CloudWatch**. Open the **Monitor** tab and click **View CloudWatch logs**. Each **log stream** holds the logs of a group of runs, newest at the top. When a function is started by something other than the Test button — such as a queue in Chapter 10 — CloudWatch is where you see what it printed.

### Example: Totaling an Order

Here is a more useful function. Given a product id and a quantity, it computes the total price, using a **lookup table** (§8.3). Create a new PyCharm project named `order_total`, with this `lambda_function.py`:

```python
PRICES = {"P100": 12.50, "P200": 8.99, "P300": 24.00, "P400": 3.75}


def lambda_handler(event, context):
    product_id = event["product_id"]
    quantity = event["quantity"]

    if product_id not in PRICES:
        print(f"Unknown product: {product_id}")
        return {"error": f"Unknown product {product_id}"}

    total = PRICES[product_id] * quantity
    print(f"{quantity} x {product_id} = {total:.2f}")
    return {"product_id": product_id, "quantity": quantity, "total": total}
```

Copy `test_lambda.py` from the `hello` project and change only its `main`:

```python
def main():
    run_test({"product_id": "P100", "quantity": 3})
    run_test({"product_id": "P999", "quantity": 1})
```

Run it:

```text
Event:    {"product_id": "P100", "quantity": 3}
3 x P100 = 37.50
Response: {"product_id": "P100", "quantity": 3, "total": 37.5}

Event:    {"product_id": "P999", "quantity": 1}
Unknown product: P999
Response: {"error": "Unknown product P999"}
```

An unknown product is not a crash. It is a normal answer that says what went wrong, so whoever called the function can show it to the user.

When both tests pass, create a function named **`order-total`** in the console (with `LabRole`), package and upload `lambda_function.py` (steps 4 and 5), and test it there with the event `{"product_id": "P100", "quantity": 3}`. The response is the same as the first one above.

### ⚠️ Common Error: An Exception in the Handler

Add a third test with no quantity, `run_test({"product_id": "P100"})`, and run the tests again. On your computer, the handler stops with an ordinary Python error:

```text
  File "...\order_total\lambda_function.py", line 6, in lambda_handler
    quantity = event["quantity"]
               ~~~~~^^^^^^^^^^^^
KeyError: 'quantity'
```

In Lambda, the same mistake makes the run **fail**, and the test result shows the error as JSON instead of a response:

```json
{
  "errorMessage": "'quantity'",
  "errorType": "KeyError",
  "requestId": "...",
  "stackTrace": [
    "  File \"/var/task/lambda_function.py\", line 6, in lambda_handler\n    quantity = event[\"quantity\"]\n"
  ]
}
```

`errorType` is the exception, `errorMessage` is its message (here, the missing key), and `stackTrace` gives the line number. It is the same information, but it is much easier to find and fix the problem in PyCharm — which is why you test there first.

### ⚠️ Common Errors When Uploading

| What you see                                                                                              | Cause and fix                                                                                                         |
| --------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `"errorType": "Runtime.ImportModuleError"`, `"errorMessage": "Unable to import module 'lambda_function': No module named 'lambda_function'"` | Lambda cannot find `lambda_function.py` at the top of the zip. You zipped the project **folder**, or named the file differently. Zip the file itself, named exactly `lambda_function.py`. |
| `"errorType": "Runtime.ImportModuleError"`, `"errorMessage": "Unable to import module 'lambda_function': No module named 'requests'"` | The handler was found, but a library it imports is not in the zip. Install it with `pip install <library> -t package` and zip the **contents** of `package` (not the folder). |
| `"errorType": "Runtime.HandlerNotFound"`                                                                  | The file is there, but it has no function named `lambda_handler`. Check the spelling of the `def` line.               |
| Lambda still runs your old code                                                                           | You changed the file but did not package and upload it again (or, after pasting, did not click **Deploy**).           |
| Your local test fails with `ModuleNotFoundError: No module named 'lambda_function'`                       | `test_lambda.py` and `lambda_function.py` are not in the same folder, or the file name is misspelled.                 |

### On Your Computer vs. in Lambda

The same code runs in both places, but its surroundings are different:

|                        | On your computer (`test_lambda.py`)               | In Lambda                                         |
| ---------------------- | ------------------------------------------------- | ------------------------------------------------- |
| Who calls the handler  | Your test program                                 | Lambda, when something starts the function        |
| `event`                | The dictionary you pass to `run_test`             | Lambda's input, turned from JSON into a dictionary |
| `print` output         | PyCharm's run window                              | The function logs and CloudWatch                  |
| boto3                  | Installed with `pip install boto3` (§6.2)         | Already installed                                 |
| AWS credentials        | Your `credentials` file (§6.3)                    | `LabRole`                                         |
| Environment variables  | The run configuration of `test_lambda.py`         | **Configuration → Environment variables**         |

A function that uses AWS — like the one in §9.6 — therefore needs its environment variables set in **both** places, and your local tests need a running lab session, because they use the real DynamoDB table.

---

## 9.4 Amazon DynamoDB

### Tables, Items, and Attributes

**Amazon DynamoDB** is a **database**: a service that stores data permanently and finds it again quickly. You do not install or manage it — you create a table, and AWS keeps it running.

| Term              | Meaning                                                  | Python equivalent             |
| ----------------- | -------------------------------------------------------- | ----------------------------- |
| **Table**         | A collection of items                                    | A dictionary of dictionaries  |
| **Item**          | One record, such as one product                          | A dictionary                  |
| **Attribute**     | One field of an item, such as its price                  | One key–value pair            |
| **Partition key** | The attribute that identifies each item; unique per item | The outer dictionary's key    |

Here is one item of a `products` table, as you will write it in Python:

```python
{"product_id": "P300", "name": "ink", "price": Decimal("24.00"), "quantity": 10}
```

`product_id` is the **partition key**. Every item **must** have it, and no two items can have the same value. That is what lets DynamoDB go straight to an item, like `d[key]` in a dictionary (§8.1).

Apart from the key, items do not all need the same attributes. One product could have a `color` and another not. DynamoDB only insists on the key.

### Creating a Table

1. In the AWS Console, search for **DynamoDB** and open it (region **us-east-1**).
2. Choose **Tables** on the left, then **Create table**.
3. **Table name:** `products`
4. **Partition key:** `product_id`, type **String**.
5. Leave **Sort key** empty and **Table settings** at **Default settings**.
6. Click **Create table** and wait until its status is **Active** (a few seconds).

Table names only need to be unique **in your own account**, so everyone in the class can name a table `products`. (Bucket names were different — those are unique across all of AWS.)

> **From the CLI:** the same table in one command (type it on one line):
>
> ```text
> aws dynamodb create-table --table-name products --attribute-definitions AttributeName=product_id,AttributeType=S --key-schema AttributeName=product_id,KeyType=HASH --billing-mode PAY_PER_REQUEST
> ```
>
> `S` means String, `HASH` is DynamoDB's other name for the partition key, and `PAY_PER_REQUEST` means you pay per read and write instead of reserving capacity.

The attribute names are **your** names, so they follow the course's naming rules: `product_id`, not `productId` or `ProductID`.

### Looking at the Items

Choose the table, then **Explore table items**. The items appear as rows, one column per attribute. You can add, edit, and delete items here by hand, which is useful for checking what your programs did. The table is empty for now; you will fill it from Python next.

---

## 9.5 DynamoDB from Python

### Connecting to a Table

DynamoDB is easiest to use through a boto3 **resource** instead of a client. A resource gives you an object for the table itself, and its items are plain dictionaries:

```python
import boto3

dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table("products")
```

As in Chapter 6, the table name is a setting, so the programs below read it from a `TABLE_NAME` environment variable. Add `TABLE_NAME=products` to each run configuration (§6.1).

### Numbers: Use `int` or `Decimal`, Not `float`

DynamoDB stores numbers **exactly**, and a `float` is not exact: Python stores `0.1` as a binary fraction that is really `0.1000000000000000055511151231257827021181583404541015625`. So boto3 refuses floats:

```text
TypeError: Float types are not supported. Use Decimal types instead.
```

Whole numbers are fine as `int`. For numbers with a decimal point, such as prices, use **`Decimal`** from the `decimal` module, and create it from a **string**:

```python
from decimal import Decimal

price = Decimal("12.50")
print(price * 3)             # 37.50
print(f"{price * 3:.2f}")    # 37.50
```

A `Decimal` works like any other number: you can do arithmetic with it, compare it, and format it in an f-string. `Decimal(12.5)` (from a float) would carry the float's error along with it, which is why you pass a string.

### Adding Items: `put_item`

`put_item` saves one item. Its argument, `Item`, is the dictionary to save:

```python
"""Adds the store's products to the products table."""

import os
from decimal import Decimal

import boto3


def add_product(table, product_id, name, price, quantity):
    """Saves one product. The price is a string such as "12.50"."""
    table.put_item(
        Item={
            "product_id": product_id,
            "name": name,
            "price": Decimal(price),
            "quantity": quantity,
        }
    )
    print(f"Added {product_id}: {name}")


def main():
    table_name = os.environ.get("TABLE_NAME")
    if table_name is None:
        print("Set the TABLE_NAME environment variable first.")
        return

    dynamodb = boto3.resource("dynamodb")
    table = dynamodb.Table(table_name)

    add_product(table, "P100", "stapler", "12.50", 25)
    add_product(table, "P200", "paper", "8.99", 40)
    add_product(table, "P300", "ink", "24.00", 10)
    add_product(table, "P400", "pens", "3.75", 100)


main()
```

```text
Added P100: stapler
Added P200: paper
Added P300: ink
Added P400: pens
```

Refresh **Explore table items** in the console: the four products are there.

`put_item` **adds** an item when the key is new and **replaces** the whole item when the key already exists — exactly like `d[key] = value` (§8.1). Running this program twice still leaves four products.

### Reading One Item: `get_item`

`get_item` finds an item by its key. The `Key` argument is a dictionary holding just the partition key:

```python
import os

import boto3

table = boto3.resource("dynamodb").Table(os.environ["TABLE_NAME"])

for product_id in ["P300", "P999"]:
    response = table.get_item(Key={"product_id": product_id})
    item = response.get("Item")
    if item is None:
        print(f"{product_id}: no such product")
    else:
        print(f"{item['name']}: ${item['price']:.2f}, {item['quantity']} in stock")
```

```text
ink: $24.00, 10 in stock
P999: no such product
```

The item is in `response["Item"]`. When there is no item with that key, DynamoDB does not raise an exception; it leaves `"Item"` out of the response, just as S3 left out `"Contents"` for an empty bucket (§6.5). `response.get("Item")` returns `None` in that case.

### What Comes Back: `Decimal`

Print a whole item and you see that **every** number comes back as a `Decimal`, even the ones you saved as `int`:

```python
print(response["Item"])
```

```text
{'quantity': Decimal('10'), 'price': Decimal('24'), 'product_id': 'P300', 'name': 'ink'}
```

Two more things to notice: DynamoDB drops trailing zeros (`24.00` is stored as `24`), and the attributes may come back in any order. Neither matters, because you look attributes up by name and format numbers with an f-string. Where you need a plain `int`, convert: `int(item["quantity"])`.

### Changing an Item

To change one attribute, read the item, change the dictionary, and save it again with `put_item`:

```python
response = table.get_item(Key={"product_id": "P300"})
item = response["Item"]
item["quantity"] = item["quantity"] + 5
table.put_item(Item=item)
print(f"{item['name']} now has {item['quantity']} in stock")
```

```text
ink now has 15 in stock
```

Because `put_item` replaces the whole item, you must save the **whole** dictionary, not just the attribute you changed. (DynamoDB also has `update_item`, which changes single attributes, but it needs a special expression syntax that is beyond this course.)

### Listing Every Item: `scan`

`scan` reads the **whole** table and returns its items as a list in `response["Items"]`. DynamoDB returns them in no particular order, so this program puts them in a dictionary keyed by id (§8.3) and loops over `sorted` keys:

```python
"""Prints every product in the products table, sorted by product id."""

import os

import boto3


def main():
    table_name = os.environ.get("TABLE_NAME")
    if table_name is None:
        print("Set the TABLE_NAME environment variable first.")
        return

    table = boto3.resource("dynamodb").Table(table_name)
    response = table.scan()
    items = response["Items"]

    products = {}
    for item in items:
        products[item["product_id"]] = item

    print(f"{len(products)} product(s):")
    for product_id in sorted(products):
        item = products[product_id]
        print(f"{product_id}  {item['name']:10}{item['price']:>8.2f}{item['quantity']:>6}")


main()
```

```text
4 product(s):
P100  stapler      12.50    25
P200  paper         8.99    40
P300  ink          24.00    15
P400  pens          3.75   100
```

A scan reads every item, so it gets slower and more expensive as a table grows. That is fine for a table of a few products. To find **one** item, always use `get_item`.

### Deleting an Item: `delete_item`

```python
table.delete_item(Key={"product_id": "P400"})
```

Like `get_item`, it takes the key. Deleting a key that does not exist is not an error. Run `add_products.py` again afterward to put the pens back — you will need all four products in §9.6.

### Reference: Common Table Operations

| Operation                                         | Meaning                                                         |
| ------------------------------------------------- | --------------------------------------------------------------- |
| `table = boto3.resource("dynamodb").Table(name)`  | Connects to the table.                                          |
| `table.put_item(Item=item)`                       | Adds the item, or replaces the item with the same key.          |
| `table.get_item(Key={"product_id": id})`          | Returns a response; the item is in `"Item"`, if it exists.      |
| `table.scan()`                                    | Returns a response; all items are in `"Items"`, as a list.      |
| `table.delete_item(Key={"product_id": id})`       | Deletes the item with that key.                                 |

### ⚠️ Common Errors with DynamoDB

| Last line of the error                                                                                                     | Cause and fix                                                                                                     |
| -------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `TypeError: Float types are not supported. Use Decimal types instead.`                                                     | An attribute is a `float`. Use `Decimal("12.50")` or an `int`.                                                    |
| `... (ValidationException) when calling the PutItem operation: One or more parameter values were invalid: Missing the key product_id in the item` | The item has no partition key, or its name is spelled differently from the table's key (`productId`, `id`). |
| `... (ResourceNotFoundException) when calling the GetItem operation: Requested resource not found`                         | The table name is wrong, the table is in another region, or it is not **Active** yet.                             |
| `KeyError: 'Item'`                                                                                                          | You used `response["Item"]` for a key that does not exist. Use `response.get("Item")` and check for `None`.       |
| `... (ExpiredToken) ...`                                                                                                   | Your lab session ended. Paste fresh credentials (§6.3).                                                           |

---

## 9.6 Lambda and DynamoDB Together

### Prices from the Table

The `order-total` function in §9.3 kept its prices in a dictionary in the code. To change a price, you would have to edit, package, and upload the function again. Now that the prices are in DynamoDB, the function can look them up there instead, and it can also check that there is enough in stock. In your `order_total` project, replace `lambda_function.py` with:

```python
import os

import boto3

dynamodb = boto3.resource("dynamodb")


def lambda_handler(event, context):
    table = dynamodb.Table(os.environ["TABLE_NAME"])
    product_id = event["product_id"]
    quantity = event["quantity"]

    response = table.get_item(Key={"product_id": product_id})
    item = response.get("Item")
    if item is None:
        print(f"Unknown product: {product_id}")
        return {"error": f"Unknown product {product_id}"}

    if quantity > item["quantity"]:
        print(f"Not enough {item['name']}: {item['quantity']} in stock")
        return {"error": f"Only {item['quantity']} {item['name']} in stock"}

    total = item["price"] * quantity
    print(f"{quantity} x {item['name']} = {total:.2f}")
    return {
        "product_id": product_id,
        "name": item["name"],
        "quantity": quantity,
        "total": float(total),
    }
```

### Test It Locally

Add a third test to `main` in `test_lambda.py`:

```python
def main():
    run_test({"product_id": "P100", "quantity": 3})
    run_test({"product_id": "P999", "quantity": 1})
    run_test({"product_id": "P300", "quantity": 50})
```

This handler uses AWS, so before you run the tests:

1. Start your lab session and make sure your `credentials` file is fresh (§6.3).
2. Add `TABLE_NAME=products` to the run configuration of **`test_lambda.py`** (§6.1).

```text
Event:    {"product_id": "P100", "quantity": 3}
3 x stapler = 37.50
Response: {"product_id": "P100", "name": "stapler", "quantity": 3, "total": 37.5}

Event:    {"product_id": "P999", "quantity": 1}
Unknown product: P999
Response: {"error": "Unknown product P999"}

Event:    {"product_id": "P300", "quantity": 50}
Not enough ink: 10 in stock
Response: {"error": "Only 10 ink in stock"}
```

The handler is running on **your** computer, but it reads the **real** `products` table, using your `credentials` file. (The stock of ink depends on how often you ran the restocking example in §9.5.) Change the price of the stapler in **Explore table items** and run the tests again: the new total appears without touching any code.

### Upload It

1. Package `lambda_function.py` and upload the zip to `order-total` (§9.3, steps 4 and 5).
2. Under **Configuration → Environment variables**, add `TABLE_NAME` with the value `products`. The run configuration on your computer does not travel with the zip.
3. Test it in the console with `{"product_id": "P100", "quantity": 3}`. The response is the same as on your computer.

The code did not change between your computer and Lambda, but three things around it did:

- **No `pip install`.** boto3 is already part of Lambda's Python.
- **No credentials.** boto3 uses `LabRole` (§9.2) instead of your `credentials` file.
- **The environment variable** comes from the function's configuration instead of PyCharm.

Notice also that **`dynamodb` is created outside the handler.** Code outside the handler runs once, when Lambda first loads the function. Later runs reuse the same connection, which makes them faster.

### ⚠️ Common Error: Returning a `Decimal`

Why does the handler return `float(total)` instead of `total`? Lambda turns your return value into JSON with `json.dumps`, and the `json` module does not know about `Decimal`. Change the return value to `"total": total` and run `test_lambda.py`. The `json.dumps` in `run_test` fails just as Lambda would:

```text
TypeError: Object of type Decimal is not JSON serializable
```

That is why the test program prints the response with `json.dumps`: it catches this mistake before you upload. In Lambda, the same mistake makes the run fail with an error like this:

```json
{
  "errorMessage": "Unable to marshal response: Object of type Decimal is not JSON serializable",
  "errorType": "Runtime.MarshalError",
  ...
}
```

Any number you read from DynamoDB is a `Decimal`, so convert it before you return it: `float(...)` for prices, `int(...)` for whole numbers. The same is true whenever you pass an item to `json.dumps` yourself. Returning `item` directly, or `item["price"]`, fails the same way.

### ⚠️ Common Errors in Lambda

| What you see                                                                      | Cause and fix                                                                                            |
| --------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| Your change has no effect                                                         | You did not package and upload the new zip (§9.3).                                                       |
| `"errorType": "KeyError"`, `"errorMessage": "'TABLE_NAME'"`                       | The environment variable is not set in Lambda, or its name is misspelled. Check **Configuration → Environment variables**. (On your computer, the same error means it is missing from the run configuration.) |
| `"errorMessage": "Task timed out after 3.00 seconds"`                             | The function took longer than its time limit. Check for an endless loop. If the function really needs longer, raise **Timeout** under **Configuration → General configuration**. |
| `... is not authorized to perform: dynamodb:GetItem ...` (AccessDeniedException) | The function is not using `LabRole`. Change it under **Configuration → Permissions**.                    |
| `"errorType": "Runtime.MarshalError"`                                             | The return value contains a `Decimal`. Convert it with `float` or `int`.                                 |


---

## Key Takeaways

1. **JSON** is the standard text format for sending dictionaries between programs. **`json.dumps`** turns a dictionary into JSON text and **`json.loads`** turns it back. JSON uses double quotes, `true`/`false`, and `null`
2. **AWS Lambda** runs a Python **handler**, `lambda_handler(event, context)`, in the cloud. `event` is the input dictionary, the **return** value is the output, and **`print`** goes to the logs in **CloudWatch**
3. In AWS Academy, every function uses the **`LabRole`** execution role. Inside Lambda, boto3 gets its credentials from that role; no keys go into the code
4. Develop the handler in PyCharm as **`lambda_function.py`**, and test it with a **test program** that imports it and calls `lambda_handler(payload, None)` with test payloads. Print the response with `json.dumps`, as Lambda does
5. When the tests pass, **zip** `lambda_function.py` (the file, not the folder). If it imports an external library other than boto3, install it with **`pip install <library> -t package`**, copy the handler into `package`, and zip the folder's **contents**. **Upload** the zip with **Upload from → .zip file**, set the function's environment variables, and run one **test event** in the console. An uncaught exception makes a run fail, and the error shows `errorType` and `errorMessage`
6. **DynamoDB** stores **items** — dictionaries — in **tables**. Every item must have the table's **partition key**, which is unique
7. With **`boto3.resource("dynamodb").Table(name)`**: **`put_item`** adds or replaces an item, **`get_item`** reads one by key (check `response.get("Item")` for `None`), **`scan`** reads all items in no particular order, and **`delete_item`** removes one
8. DynamoDB numbers are **`Decimal`**, never `float`. Create them from strings, `Decimal("12.50")`, and convert them with `float` or `int` before returning them from Lambda or passing them to `json.dumps`

---

*End of Chapter Nine*

[← Back to Course Index](../table-of-contents.md)
