---
title: "Principle of Clean Python Code - Naming, Comments, Functions"
date: 2023-09-28T20:42:31+08:00
tags: ["Python"]
---

Writing clean code is a fundamental goal to enhance code readability, comprehension, and maintainability. It is considered a best practice to adhere to clean code principles when developing in Python. In this article, we will explore key principles of clean Python code, focusing on naming conventions, commenting practices, and function design.
<!--more-->

# Naming

[PEP8](https://peps.python.org/pep-0008/#naming-conventions) has shown comprehensive naming conventions for Python code. Here are some key points:

1. variable names should be lowercase, with words separated by underscores. But, if the variable is a global variable, it should be all uppercase and declared at the top of the file

```python
import random

# global variable
RANDOM_SEED = 42

random.seed(RANDOM_SEED)
random_number = random.randint(0, 100)
```

2. Avoid using single-character variable names like `i` or `j`. Instead, opt for meaningful names such as `index` or `index_of_something`. This practice ensures better code readability, especially as your code grows or includes other counter variables. It benefits both you and your readers in the long run.

3. Class names should adhere to the CapWords convention, as demonstrated in the code with the class name `FortuneTeller`. Additionally, instance names should align with the class name for consistency. Avoid creating instances with names that deviate from the class name, as it can lead to confusion.

```python
import random

class FortuneTeller:
    def tell_fortune(self) -> None:
        random_number = random.randint(0, 10)
        if random_number < 5:
            print("You will have a good day!")
        else:
            print("You will have a bad day!")

fortune_teller = FortuneTeller()
fortune_teller.tell_fortune()

forecaster = FortuneTeller() # A bad instance name
```

Moreover, carefully use abbreviations and acronyms in naming. Data scientists often use `df` as the variable name for dataframes, which may be unclear to non-data science developers. Instead, choose meaningful names like `customer` for a dataframe containing customer data.

Similarly, for class instances, avoid tempting but ambiguous abbreviations like `fs` for `FortuneTeller`. In a complex codebase with instances from different classes, like `sf` for `SalesForecast`, clear and consistent naming is crucial for code comprehension. Use abbreviations and acronyms when readers have the necessary context and understand their meanings.


# Comments

Comments should provide additional information that cannot be inferred from the code itself, such as business requirements or algorithm background. If a function is named `read_data_from_s3`, there is no need for a comment like `Read data from S3` as the function name already conveys its purpose.

Furthermore, type hints are a valuable tool for reducing the need for comments. By specifying the types of function arguments and return values using type hints, you can provide clear documentation within the code itself. I recall encountering code containing comments such as "... returns a list of strings or None." Upon closer inspection, it became evident that the function exclusively returned a list of strings, and the comment had not been updated, rendering it both redundant and potentially confusing. By incorporating type hints, you can not only eliminate the need for such comments but also ensure accurate and reliable code documentation.


```python
# bad comment since it is misleading
def get_strings():
    """Returns a list of strings or None."""
    return ["a", "b", "c"]

# better
def get_strings() -> list[str]:
    return ["a", "b", "c"]
```

# Functions

Functions serve to extract named concepts from procedural code. Typically, when you encounter a repetitive code block, you should consider extracting it into a function, following the **D**on't **R**epeat **Y**ourself (DRY) principle. In the example below, we notice that the calculation of the average score for students occurs twice. To eliminate code duplication, we can create a function for this purpose, or even implement it as a method within the Student class.

```python
class Student:
    def __init__(self, name: str, english_score: float, math_score: float):
        self.name = name
        self.english_score = english_score
        self.math_score = math_score

def print_scores(students: list[Student]) -> float:
    """Display the average scores of students in descending order."""
    students_ranking = sorted(students, key=lambda student: (student.english_score + student.math_score) / 2, reverse=True)

    for student in students_ranking:
        print(f"Student Name: {student.name}, Average score: {(student.english_score + student.math_score) / 2}")

students = [Student("Sara", 80, 90), Student("Tom", 85, 70)]
print_scores(students)
```

```python
class Student:
    def __init__(self, name: str, english_score: float, math_score: float):
        self.name = name
        self.english_score = english_score
        self.math_score = math_score

    @property # property decorator allows us to access the method as a read-only attribute
    def average_score(self) -> float:
        return (self.english_score + self.math_score) / 2

def print_scores(students: list[Student]) -> float:
    """Display the average scores of students in descending order."""
    students_ranking = sorted(students, key=lambda student: student.average_score, reverse=True)

    for student in students_ranking:
        print(f"Student Name: {student.name}, Average score: {student.average_score}")
    
students = [Student("Sara", 80, 90), Student("Tom", 85, 70)]
print_scores(students)
```

As previously mentioned, incorporating type hints significantly enhances code readability. Type hints provide clear information about a function's input and output, making it easier for readers to understand its usage. Additionally, tools like MyPy can be employed to check for correct usage, ensuring that the code adheres to the specified types.


When creating functions, it's crucial to adhere to the principle of "Separation of Concerns," meaning that a function should perform a single, well-defined task. Using flag variables to alter a function's behavior, as seen in the function `get_data` below with the `is_test` flag, can lead to issues. It's easy to forget to modify the flag when the function is used differently, and it makes the function's behavior less clear. A more effective approach is to split the function into two separate functions: one for test data and another for training data.

```python
# bad
def get_data(is_test: bool) -> str:
    if is_test:
        return "test data"
    else:
        return "training data"

# better
def get_test_data() -> str:
    return "test data"

def get_training_data() -> str:
    return "training data"
```

The final principle I'd like to emphasize is "Do not reinvent the wheel." Python offers an extensive range of built-in functions and libraries. If a specific functionality isn't available natively, it's often worth searching for third-party packages to check if someone has already implemented it. However, when utilizing third-party packages, exercise caution, and ensure they are reliable and well-maintained to avoid potential issues in your code.
