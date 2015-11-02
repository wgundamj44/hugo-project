---
title: "Solve pylint import error in Django"
categories: ["tech"]
date: 2015-10-23
tags: ["Django"]
---

I'm using emacs flycheck with pylint to lint my Django project. Everything goes well except I constantly reports
import error for my user-defined modules. The system modules are ok, the Django modules are ok too. We know that python uses
sys.path to search for modules, so the problem must be python interpreter sees different sys.path value to pylint.

So I created a pylintrc file, and put it in the base folder of my project(the folder contains manage.py), in content is:

```python
[Master]
init-hook='import sys; sys.path.append("/base/folder/of/my/project")'
```
Then I found import error disappears. So the reason is clear, pylint lacks the path to the base folder of my project.

As mentioned in the pydocs, when script invokes the interpreter, the path to this script will be added to the head of sys.path.
manage.py is the entry script of Django project, so the folder of manage.py will be added to sys.path, as a result, all the modules can be
import relative to this path. That is why our Django project can import self-made module properly. To mimic this behavior, a properer way of
writing pylintrc above is:

```python
[Master]
init-hook='import sys; sys.path = ["/base/folder/of/my/project"] + sys.path'
```
