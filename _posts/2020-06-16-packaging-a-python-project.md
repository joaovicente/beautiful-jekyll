---
layout: post
published: true
title: Packaging a Python project
tags: python
---
# A Packaging a Python project as a source distribution

This setup will be minimalist, for consumption as a Source distribution

## Create your project

Create a directory to hold your package and step into it

```
$ mkdir mypypackage
$ cd mypypackage
```

Now create your module `api.py` 
```python
from art import text2art

def hello():
	print(text2art("Hello Python"))

def goodbye():
	print(text2art("Goodbye Python"))
```

Notice that the module requires `text2art` library

Now create an `__init__.py` file to manage functions classes you want to expose
```
from .api import hello, goodbye
```

Now create a `setup.py` describing your package

```
import setuptools

with open("README.md", "r") as fh:
    long_description = fh.read()

setuptools.setup(
    name="mypypackage",
    version="0.0.1",
    author="Joao Vicente",
    url="https://github.com/joaovicente/mypypackage",
    description="Example on how to package a Python library",
    packages=setuptools.find_packages(),
    classifiers=[
        "Programming Language :: Python :: 3",
        "License :: OSI Approved :: Apache License 2.0",
        "Operating System :: OS Independent",
    ],
    install_requires=[
        "art"
    ],
    python_requires='>=3.6',
)
```

Notice the external dependency definition in `install_requires=["art"]`

Create a `README.md`

```
# mypypackage
The goal of this project is to demonstrate how to create a Python package for distribution
```

You now have all the key ingredients ready for packaging

## Install setuptools and wheel 

```
python3 -m pip install --user --upgrade setuptools testresources wheel
```

## Generate distribution archive

```
python3 setup.py sdist bdist_wheel
```

Check the archive has been built

```bash
$ ls ./dist
mypypackage-0.0.1.tar.gz  mypypackage-0.0.1-py3-none-any.whl
```

## Install from archive

Create a virtual enviroment, to test within a sterile test environment (without your already installed python libraries)

```bash
$ python3 -m venv myenv
$ source myenv/bin/activate
```

Install the package built above

```
$ pip3 install wheel
$ pip3 install ./dist/mypypackage-0.0.1.tar.gz
```

try it
```
$ python3
>>> import mypypackage

>>> mypypackage.hello()
 _   _        _  _          ____          _    _                   
| | | |  ___ | || |  ___   |  _ \  _   _ | |_ | |__    ___   _ __  
| |_| | / _ \| || | / _ \  | |_) || | | || __|| '_ \  / _ \ | '_ \ 
|  _  ||  __/| || || (_) | |  __/ | |_| || |_ | | | || (_) || | | |
|_| |_| \___||_||_| \___/  |_|     \__, | \__||_| |_| \___/ |_| |_|
                                   |___/                           
```

Uninstall the package
```
$ pip3 uninstall mypypackage
```

You can now exit the `myenv` virtual environment

```bash
$ deactivate
```

## Install from github

All the code above is available in [https://github.com/joaovicente/mypypackage](https://github.com/joaovicente/mypypackage)

If you want to install directly from github:

```bash

```

## References
* [python.org Packages](https://docs.python.org/3/tutorial/modules.html#packages)
* [python.org Packaging projects](https://packaging.python.org/tutorials/packaging-projects)
* [python.org Installing from local archives](https://packaging.python.org/tutorials/installing-packages/#installing-from-local-archives)
* [python.org Virtual environments](https://docs.python-guide.org/dev/virtualenvs/)
* [realpython.com Package initialization](https://realpython.com/python-modules-packages/#package-initialization)
* [Create Python package with init](https://timothybramlett.com/How_to_create_a_Python_Package_with___init__py.html)
* [Another Hello World tutorial](https://github.com/gdamjan/hello-world-python-package)