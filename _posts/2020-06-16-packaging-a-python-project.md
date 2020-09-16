---
layout: post
published: true
title: Packaging a Python project
tags: python
---
# A Packaging a Python project as a source distribution

This setup will be minimalist, for consumption as a Source distribution

## Create your project

hello.py
```python
def greet():
	print("Hello Python package")
```

README.md
```
Something useful
```

setup.py
```
import setuptools

with open("README.md", "r") as fh:
    long_description = fh.read()

setuptools.setup(
    name="hello-joao-vicente",
    version="0.0.1",
    author="Joao Vicente",
    author_email="joao.vicente@gmail.com",
    description="Says hello",
    packages=setuptools.find_packages(),
    classifiers=[
        "Programming Language :: Python :: 3",
        "License :: OSI Approved :: MIT License",
        "Operating System :: OS Independent",
    ],
    python_requires='>=3.6',
)
```

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
hello-joao-vicente-0.0.1.tar.gz  hello_joao_vicente-0.0.1-py3-none-any.whl
```

## Install in target environment

Create a virtual enviroment

```bash
$ python3 -m venv myenv
$ source myenv/bin/activate
```

When finished with the environment deactivate it

```bash
$ deactivate
```

Install the package built above

```
$ pip3 install wheel
$ pip3 install ./dist/hello-joao-vicente-0.0.1.tar.gz
```

try it
```
$ python3
>>> import hello

>>> hello.greet()
Hello Python package
```

Uninstall the package
```
pip3 uninstall hello-joao-vicente
```

## References
* [python.org Packages](https://docs.python.org/3/tutorial/modules.html#packages)
* [python.org Packaging projects](https://packaging.python.org/tutorials/packaging-projects)
* [python.org Installing from local archives](https://packaging.python.org/tutorials/installing-packages/#installing-from-local-archives)
* [python.org Virtual environments](https://docs.python-guide.org/dev/virtualenvs/)
