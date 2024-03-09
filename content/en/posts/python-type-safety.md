+++
title = 'How to code in Python with type-safety in mind?'
date = 2024-03-08T15:40:04-08:00
+++

## TL;DR

> Python is a dynamically typed language that is easy for beginners but can lead to hard-to-maintain "tactical" code, especially in research projects. In contrast, good software engineers write well-structured, maintainable "strategic" code.
>
> Adding type hints to Python code makes it much clearer what types of inputs functions expect and what they return. This improves code readability and enables better editor auto-completion.
>
> Using the pydantic library allows validating that inputs match the specified type hints. Custom validation logic can also check things like required DataFrame columns or dictionary keys. This catches errors early.
>
> Including docstrings for classes and functions helps users understand how to use the code and enables auto-generating documentation.
>
> Finally, using the mypy static type checker can catch type errors before they occur at runtime, providing an extra layer of safety.

## Motivation

Python is a dynamically typed language, making it easy for beginners to start coding and a popular choice for data-related tasks in research projects.

However, researchers in the natural sciences often prioritize obtaining results, publishing papers, and completing their academic goals. Due to their focus on these objectives and potentially limited training in software engineering best practices, the code they write can sometimes be difficult for future researchers to maintain, understand, and test, leading to challenges for those working with the codebase in the future. This type of code is often described as "tactical" code, solving a specific problem without considering future maintenance or extension. In contrast, good software engineers write "strategic" code, which is well-planned, structured, and designed for future maintenance and extension.

This post will discuss **best practices for writing Python code with type-safety in mind**, helping you write strategic code that is **easier to maintain and extend in the future**.

## A bad example

```python
class Analysis:
    def __init__(self, data, config):
        self.data = data
        self.config = config

    def run(self, mode):
        result = pd.DataFrame(columns=["result1", "result2", "result3"])
        result["result1"] = self.data["column1"] + self.data["column2"]
        if self.config["key1"]:
            result["result2"] = self.data["column1"] * self.data["column2"]
        else:
            result["result2"] = self.data["column1"] / self.data["column2"]
        if mode == "mode1":
            result["result3"] = self.data["column1"] - self.data["column2"]
        else:
            result["result3"] = self.data["column1"] + self.data["column2"]
        return result
```

This is a very typical code snippet in a poorly typed data analysis project (I've seen a lot of these in research projects). Let's list the problems with this code:

1. We have no idea what `data`, `config`, `mode`, and `results` are. We have to read the code to understand what they are.
2. We can pass invalid values to `data`, `config`, and `mode` without any error or warning until we run the code.
3. The worst part is that we can get wrong results without any error or warning. For example, passing `"mode3"` to `mode` will not raise any error, but it will give us the same result as `"mode2"`.

## Use type hints

Now let's add type hints to this code:

```python
from typing import Any, Dict, Literal
import pandas as pd

class Analysis
    def __init__(self, data: pd.DataFrame, config: Dict[str, Any]):
        self.data = data
        self.config = config

    def run(self, mode: Literal["mode1", "mode2"]) -> pd.DataFrame:
        result = pd.DataFrame(columns=["result1", "result2", "result3"])
        result["result1"] = self.data["column1"] + self.data["column2"]
        if self.config["key1"]:
            result["result2"] = self.data["column1"] * self.data["column2"]
        else:
            result["result2"] = self.data["column1"] / self.data["column2"]
        if mode == "mode1":
            result["result3"] = self.data["column1"] - self.data["column2"]
        else:
            result["result3"] = self.data["column1"] + self.data["column2"]
        return result
```

This looks much better, even without any comments. Now let's see what we have achieved:

1. `data`: We know that `data` is a pandas DataFrame.
2. `config`: We know that `config` is a dictionary with string keys and any values.
3. `mode`: We know that `mode` is a string, and it can only be `"mode1"` or `"mode2"`. You will also get editor autocompletion for these values.
4. `result`: We know that `result` is a pandas DataFrame.

## Use [`pydantic`](https://docs.pydantic.dev/latest/) for data validation

Now that we've added type hints to our function arguments and return values, but this code is still very error-prone. Let's see what can go wrong:

1. Even we have type hints, it is not enforced. We can still pass a list to `data`, a string to `config`, an integer to `mode`, and a string to `result` without any error or warning.
2. We can pass a DataFrame with wrong columns to `data`, a dictionary with wrong keys to `config`, and there is no way to inform the user about this.

This is where `pydantic` comes in. `pydantic` is a data validation library that uses Python type hints to validate data. Let's see how we can use `pydantic` to validate our input:

```python
from pydantic import BaseModel
from typing import Any, Dict, Literal
import pandas as pd

class Analysis(BaseModel):
    data: pd.DataFrame
    config: Dict[str, Any]

    def run(self, mode: Literal["mode1", "mode2"]) -> pd.DataFrame:
        result = pd.DataFrame(columns=["result1", "result2", "result3"])
        result["result1"] = self.data["column1"] + self.data["column2"]
        if self.config["key1"]:
            result["result2"] = self.data["column1"] * self.data["column2"]
        else:
            result["result2"] = self.data["column1"] / self.data["column2"]
        if mode == "mode1":
            result["result3"] = self.data["column1"] - self.data["column2"]
        else:
            result["result3"] = self.data["column1"] + self.data["column2"]
        return result
```

Now, with `pydantic`, we will get an error immediately if we pass a value with the wrong type to `data`, `config`, or `mode`.

### Custom validation

We can even add more validation rules to our model, such as checking the columns of `data` or the keys of `config`. Let's see how we can do this:

```python
from pydantic import BaseModel, validator
from typing import Any, Dict, Literal
import pandas as pd

class Analysis(BaseModel):
    data: pd.DataFrame
    config: Dict[str, Any]

    @validator("data")
    def check_data(cls, v):
      if not all(col in v.columns for col in ["column1", "column2"])
        raise ValueError("data must have columns column1 and column2")
      return v

    @validator("config")
    def check_config(cls, v):
      if "key1" not in v:
        raise ValueError("config must have key1")
      if type(v["key1"]) != bool:
        raise ValueError("config['key1'] must be a boolean")
      return v

    def run(self, mode: Literal["mode1", "mode2"]) -> pd.DataFrame:
        result = pd.DataFrame(columns=["result1", "result2", "result3"])
        result["result1"] = self.data["column1"] + self.data["column2"]
        if self.config["key1"]:
            result["result2"] = self.data["column1"] * self.data["column2"]
        else:
            result["result2"] = self.data["column1"] / self.data["column2"]
        if mode == "mode1":
            result["result3"] = self.data["column1"] - self.data["column2"]
        else:
            result["result3"] = self.data["column1"] + self.data["column2"]
        return result
```

What we have achieved now:

1. Immediate error if we pass a value with the wrong type to `data`, `config`, or `mode`.
2. Immediate error if we pass a DataFrame missing columns `column1` and `column2` to `data`.
3. Immediate error if we pass a dictionary missing key `key1` to `config`.
4. Immediate error if we pass a dictionary with `key1` not being a boolean to `config`.
5. We have a clear and concise code that is easy to understand and maintain.
6. We can get editor autocompletion for `data`, `config`, and `mode`.

## Don't forget docstrings

Finally, don't forget to add docstrings to your functions and classes. This will help your users understand what your code does and how to use it. Docstrings can also be used to generate documentation for your code using tools like [Sphinx](https://www.sphinx-doc.org/en/master/) or standard [pydoc](https://docs.python.org/3/library/pydoc.html).

```python
from pydantic import BaseModel, validator
from typing import Any, Dict, List, Union, Literal
import pandas as pd

class Analysis(BaseModel):
    """A class to analyze data."""
    data: pd.DataFrame
    config: Dict[str, Any]

    @validator("data")
    def check_data(cls, v):
      """Check if data has the right columns."""
      if ["column1", "column2"] not in v.columns:
        raise ValueError("data must have columns column1 and column2")
      return v

    @validator("config")
    def check_config(cls, v):
      """Check if config has the right keys."""
      if "key1" not in v:
        raise ValueError("config must have key1")
      if type(v["key1"]) != bool:
        raise ValueError("config['key1'] must be a boolean")
      return v

    def run(self, mode: Literal["mode1", "mode2"]) -> pd.DataFrame:
        """
        Run the analysis.

        Args:
            mode (Literal["mode1", "mode2"]): The mode to run the analysis.

        Returns:
            pd.DataFrame: The result of the analysis with columns result1, result2, and result3.
        """
        result = pd.DataFrame(columns=["result1", "result2", "result3"])
        result["result1"] = self.data["column1"] + self.data["column2"]
        if self.config["key1"]:
            result["result2"] = self.data["column1"] * self.data["column2"]
        else:
            result["result2"] = self.data["column1"] / self.data["column2"]
        if mode == "mode1":
            result["result3"] = self.data["column1"] - self.data["column2"]
        else:
            result["result3"] = self.data["column1"] + self.data["column2"]
        return result
```

## Advanced usage: `mypy` for static type checking

While adding type hints and using pydantic for data validation can catch many type-related errors, it's still possible for type inconsistencies to slip through and only be caught at runtime. This is where static type checking tools like `mypy` come in.

`mypy` is a static type checker for Python that analyzes your code and identifies potential type errors before the code is executed. It uses the type hints in your code to infer the types of variables and expressions and reports any type inconsistencies it finds.

To use mypy, first install it via pip:

```bash
pip install mypy
```

Then, run mypy on your Python files:

```bash
mypy your_file.py
```

`mypy` will report any type errors it finds, along with the line numbers where the errors occur. For example, if we had forgotten to return a pd.DataFrame from the run() method in our earlier example, mypy would report an error like this:

```
your_file.py:42: error: Incompatible return value type (got "None", expected "DataFrame")
```

This tells us that on line 42 of `your_file.py`, we're returning `None` when we've declared that the function should return a `pd.DataFrame`.

To get the most benefit from `mypy`, it's a good idea to configure it to run as part of your continuous integration (CI) pipeline. This way, `mypy` will automatically check your code for type errors whenever you push changes, and you can catch and fix type issues before they make it into production.

You can also install the `mypy` [extension for VSCode](https://marketplace.visualstudio.com/items?itemName=ms-python.mypy-type-checker) to get real-time type checking as you write your code.

Using `mypy` in combination with type hints and pydantic provides a powerful way to ensure the type safety and correctness of your Python code, making it easier to maintain and extend over time.
