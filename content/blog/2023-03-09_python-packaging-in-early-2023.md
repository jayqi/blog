---
title: The Basics of Python Packaging in Early 2023
date: 2023-03-09
summary: Explaining the basic concepts and best practices for creating Python packages in early 2023 using pyproject.toml build standards.
canonicalURL: https://drivendata.co/blog/python-packaging-2023
showCanonicalLink: true
---

You may have heard there are new, modern standards in Python packaging (`pyproject.toml`!) that have been adopted over the last few years. There are now several popular and shiny modern tools for managing your packaging projects. (Poetry! Hatch! PDM!) However, the documentation is scattered and much of it is specific to these competing tools. What are the recommended best practices when creating a Python package? What is the _minimal_ amount that you need to do in order to follow the best practices?

This blog post covers the following topics as they are in early 2023:

1. A quick how-to guide on what you _minimally_ need to do to adopt the modern packaging standards for _most cases_ (with further reading for more complicated cases).
2. Explanations of the concepts so that you can make informed decisions about the most appropriate tools for you.

First, some quick definitions: **packaging** refers to the general activity of creating and distributing Python **packages**, which are bundles of Python code (usually as a compressed file archive) in a particular format that can be distributed to other people and installed by a tool like [pip](https://docs.python.org/3/tutorial/venv.html#managing-packages-with-pip). The action of turning your Python source code into the package thing is often referred to as **building** the package.

There are many other activities that are _not_ packaging but are related, such as [virtual environment](https://docs.python.org/3/tutorial/venv.html) management and dependency management. Most of this post will not be about those topics, but they will turn up later when we discuss popular tools that handle packaging _and_ these other things.

## What are the standards?

The "new standards" refer to a standardized way to specify package metadata (things like package name, author, dependencies) in a `pyproject.toml` file and the way to build packages from source code using that metadata. You often see this referred to as "pyproject.toml-based builds." If you have created Python packages in the past and remember using a `setup.py` or a `setup.cfg` file, the new build standards are replacing that.

The standards were defined through a series of PEPs—short for Python Enhancement Proposals, which are design documents about new standards or features in Python. PEPs are referenced using numeric identifiers, and these ones in particular are usually what people consider to constitute the pyproject.toml package standards: [PEP 517](https://peps.python.org/pep-0517/), [PEP 518](https://peps.python.org/pep-0518/), [PEP 621](https://peps.python.org/pep-0621/), and [PEP 660](https://peps.python.org/pep-0660/). Two of them—PEP 517 and PEP 621—are most relevant to what _you_ as a package author need to do to adopt these modern standards. The other ones are mostly relevant to the people making tools _for_ package authors.

## How do I use these standards?

There are two main things you need to do: (1) declare a build system and (2) declare your package metadata. Both of these are sections that must exist in the `pyproject.toml` file.

### 1. Declaring a build backend (PEP 517)

<span style="color:#999">(Technically, this is addressed in both PEP 517 and PEP 518, but _mostly_ in PEP 517. You will often see compliant build backends called "PEP 517 build backends.")</span>

To do this, include a section (called ["table"](https://toml.io/en/v1.0.0#table) in TOML vocabulary) in your `pyproject.toml` that looks like this:

```toml
[build-system]
requires = ["flit_core >=3.2,<4"]
build-backend = "flit_core.buildapi"
```

That's it! A `[build-system]` table with those two fields are the minimum that you need. You basically need to choose a build backend—some Python library that does the package building. That choice determines what you put into the two fields. The backend needs to be declared as a build-time dependency under `requires`. The backend will have some specific API for building—you just look up the name in the backend's documentation, and you put that under `build-backend`.

How do you choose a backend? What does "backend" even mean? This can get complicated and is explained in more detail later, but the short answer is that it doesn't really matter for most simple cases. If your package is pure Python and is compliant with PEP 621 (the next section), then compatible backends will all generally do the same thing. Popular options at the time of this post include: [flit-core](https://github.com/pypa/flit/tree/main/flit_core), [hatchling](https://pypi.org/project/hatchling/), [pdm-backend](https://pdm-backend.fming.dev/), and [setuptools (>=61)](https://setuptools.pypa.io/en/latest/userguide/pyproject_config.html).

A few notes:

- **Did you say setuptools?** Yes! You may be familiar with [setuptools](https://setuptools.pypa.io/en/latest/index.html) as the thing that used your `setup.py` files to build packages. Setuptools now also fully supports `pyproject.toml`-based builds since version 61.0.0. You can do everything in `pyproject.toml` and no longer need `setup.py` or `setup.cfg`.
- **What about poetry?** [Poetry](https://python-poetry.org/) has a PEP 517 build backend ([poetry-core](https://pypi.org/project/poetry-core/)), but it is _not_ PEP 621 compliant (the next section). This is because Poetry was developed before PEP 621 was adopted, and it hasn't yet migrated from its own custom way of declaring project metadata. It probably will someday—follow along with [this issue](https://github.com/python-poetry/roadmap/issues/3). Poetry is popular for other reasons discussed later, but—as of the time of this post—it doesn't work with what's discussed in the next section.
- **What if I don't have a simple pure-Python package?** If your package is _not_ just a pure-Python package and/or you need to do something else complicated to build your package, see the later section ["More complicated build needs"](#more-complicated-build-needs) for more discussion.

### 2. Declaring project metadata (PEP 621)

To do this, you include a `[project]` table (section) in your `pyproject.toml`. This will contain fields that will feel familiar if you're used to using `setup()` with `setup.py`. A light example adapted from [PyPA's "Packaging Python Projects"](https://packaging.python.org/en/latest/tutorials/packaging-projects/) tutorial looks like this:

```toml
[project]
name = "example_package"
version = "0.0.1"
description = "A small example package"
authors = [
  { name = "Example Author", email = "author@example.com" },
]
license = { file = "LICENSE" }
readme = "README.md"
classifiers = [
    "Programming Language :: Python :: 3",
    "License :: OSI Approved :: MIT License",
    "Operating System :: OS Independent",
]
requires-python = ">=3.7"
dependencies = [
  "requests>2",
]

[project.urls]
"Homepage" = "https://github.com/pypa/sampleproject"
"Bug Tracker" = "https://github.com/pypa/sampleproject/issues"
```

One note here is that the tables with dotted names like `[project.urls]` are actually just more expansive ways of writing key-value mappings nested under a field. You can equivalently have a `urls` field under `project` with the entries written in the [inline dictionary-like syntax](https://toml.io/en/v1.0.0#inline-table) that you see inside the array in the `authors` field.

You can find a detailed guide ["Declaring Project Metadata"](https://packaging.python.org/en/latest/specifications/declaring-project-metadata/) about writing project metadata on PyPA's documentation website. The definition of what goes under `[project]` is what PEP 621 standardizes.

As previously mentioned, Poetry/poetry-core does not yet support this standard for project metadata. If you choose to use poetry-core, you will be declaring your project metadata using [Poetry's own custom format](https://python-poetry.org/docs/basic-usage/#project-setup).

### You're done!

If you have a simple pure-Python package and you've followed those two steps, then your package is now using the modern standards. Now any tool that can build or install packages can cleanly and reliably handle it. One easy way to actually build your package is to use the (maybe _too_ simply named) [build](https://github.com/pypa/build) package. It's a simple and minimal tool for building packages maintained by the [PyPA](https://www.pypa.io/en/latest/), a working group closely involved with setting the packaging standards and maintaining associated tooling. After installing build, you can use it by simply calling `python -m build`. This will create a `.whl` [wheel](https://packaging.python.org/en/latest/specifications/binary-distribution-format/) archive and a [`tar.gz` "source distribution"](https://packaging.python.org/en/latest/specifications/source-distribution-format/) archive for your package.

## Explaining the concepts

### What problems do pyproject.toml-based builds solve?

The old way that packaging was done was with a `setup.py` file. You'd write a function call to a `setup` function imported from the setuptools package that defined all of your package metadata.

First, this was unstructured data! You had to run this `setup.py` Python script to even read that data, because it was specified in code. It was more complex than it needed to be, and there were not a lot of guiderails to prevent bad practices. Because it was a script, people could write arbitrary code to dynamically do arbitrary things.

Now, with the `pyproject.toml` file, everything is declared in a static text file. You can use any TOML reader to read it in. (As of Python 3.11, [tomllib](https://docs.python.org/3/library/tomllib.html) now part of the standard library.) It's standardized—there are documented fields that mean specific things and expect certain values.

Another problem with `setup.py` files was dealing with build-time dependencies. Consider the following situation: let's say you have a package that requires PyTorch to build with C++ and CUDA extensions. Following the [documented typical usage](https://pytorch.org/docs/1.13/cpp_extension.html), you would import things from torch in your `setup.py`. That meant you had to install torch in your environment before you could even `pip install` the package from source. Even worse, simply listing torch as a required dependency wasn't enough! Even listing torch together with your package in a `requirements.txt` file wasn't enough! Pip could not even check that package's dependencies: pip would need to run `setup.py` to get the dependencies, but `setup.py` couldn't run because it needs to import torch. Why do you need to install dependencies and run code just to read a bunch of static metadata? Why are there so many steps to install a Python package? This kind of situation has been [very](https://github.com/facebookresearch/detectron2/issues/3323) [confusing](https://github.com/scipy/scipy/issues/9297) [for users](https://github.com/pandas-dev/pandas/issues/32045). Furthermore, if you only needed certain dependencies for building and not at runtime, it would be annoying to your users to have to install and carry around those dependencies in their environments.

PEP 517 and PEP 518 solve these problems by introducing the separate and explicit declaration of build-time dependencies. So, users can simply `pip install` the package they want, and `pip` itself will take care of setting up build-time dependencies like PyTorch, NumPy, or Cython. Additionally, with the new default practice of isolated build environments where those dependencies get installed only when building. Dependencies also don't have to stick around after if they're no longer needed. And, if a user is installing a wheel archive which is pre-built, they don't even need the build-time dependencies at all.

### How do I choose a build backend?

#### What is a build backend?

PEP 517 introduced the distinct concepts of build "frontends" and build "backends". The **backend** is the program that reads your `pyproject.toml` and actually does the work of turning your source code into a package archive that can be installed or distributed. The **frontend** is just a user interface (usually a command-line program) that calls the backend.

- Examples of frontends include: pip, build, poetry, hatch, pdm, flit
- Examples of backends include: setuptools (>=61), poetry-core, hatchling, pdm-backend, flit-core

The design of separating these two pieces in the build workflow means that—in principle—you can mix and match frontends and backends. Under this design, when you do `pip install` pointed at source code (e.g., local source or a `tar.gz` source distribution), or when you run a command like `python -m build` or `poetry build`, you are using a frontend. That frontend looks up which build backend you declared in your `pyproject.toml` (step #1 of the above how-to section), makes a fresh virtual environment by default, installs the backend into the virtual environment, and runs it to build your package. Once again, backends that work this way are called "PEP 517 backends".

#### Okay, so how do I choose a build backend?

As previously stated: if you have a relatively simple package, and you followed the two steps above, then it doesn't really matter!

One reasonable line of questioning might be: why do I even need to choose a backend? If the input `pyproject.toml` is standardized and the output package archive formats are standardized, shouldn't there just be one way to do it? This is basically true in most cases! If you are writing a simple package in pure Python, you will more or less get the same outcome with any backend that is compliant with both PEP 517 (`[build-system]`) and PEP 621 (`[project]`). Once again, popular options at the time of this post include: setuptools (>=61), flit-core, hatchling, and pdm-backend.

So how are these backends actually different? They are mostly different in two ways:

1. Extra features related to the build process
2. Integration with frontend systems that provide additional project and workflow management functionality

#### Extra build features

Firstly, there are details about building packages that are outside of the scope of what PEP 621 defined for `[project]`. This includes, for example, customizing what extra files get included in your built package, running code at build time, and how to handle filling in dynamic fields. PEP 621 allows most fields to be filled dynamically, but it is up to the backend on which fields they may support doing that for and how you'd configure that. The most commonly supported dynamic field is the package version; many backends support reading it from the source code or the repository's version control system.

If you want or need those kinds of features, then you will need a build backend that supports them. Then, you will need to set backend-tool-specific configuration in your `pyproject.toml` for it. For example, you'd include this in your `pyproject.toml` to tell the `hatchling` backend how to include or exclude files in your build:

```toml
[tool.hatch.build]
include = [
  "pkg/*.py",
  "/tests",
]
exclude = [
  "*.json",
  "pkg/_compat.py",
]
```

Notice that the table name is `[tool.hatch.build]`. This configuration's valid options are defined by and specific to use with `hatch`/`hatchling`. If you switch to a different backend with its own custom configuration, you'll need a `[tool.setuptools]` or `[tool.pdm]` table with the appropriate keys and values. There are some differences in precisely what features are offered by the various backends, and how much customizability they offer. If you need those kinds of features, you should compare them between the different tools.

#### Workflow management with the frontends

Secondly in how these choices differ: many of these backends are strongly coupled to powerful workflow management programs. You've probably noticed the pairings with frontends: poetry/poetry-core, hatch/hatchling, pdm/pdm-backend. These frontends actually do significantly more than just act a frontend to the build process. Many of them are comprehensive workflow management tools for Python projects. They include functionality such as:

- Creating and managing your virtual environments (what you might use [venv](https://docs.python.org/3/library/venv.html) or [conda](https://docs.conda.io/projects/conda/en/stable/) for)
- Commands for managing and installing your dependencies (what you might use hand-written requirements files and pip for)
- Dependency lockfiles (what you might use [pip-compile](https://pip-tools.readthedocs.io/en/latest/) for)
- Environment matrices for testing (what you might use [tox](https://tox.wiki/en/latest/) or [nox](https://nox.thea.codes/en/stable/) for)
- Commands for uploading your package to PyPI (what you might use [twine](https://twine.readthedocs.io/en/stable/) or a [CI module](https://github.com/marketplace/actions/pypi-publish) for)

One sign that these tools are bigger than package building is that Poetry, Hatch, and PDM all recommend that you install them in isolated environments outside of your project-specific virtual environments in order to use them as global Python project managers. If you're choosing to use one of these tools, you're generally choosing them for all of this other functionality that is in addition to building a package.

As previously noted, Poetry does not yet support PEP 621 (`[project]`) and continues to use its own project metadata specifications. Still, Poetry is highly popular and well-regarded because people like its design and user experience. Poetry plans to support PEP 621 eventually—you can follow progress in [this issue](https://github.com/python-poetry/roadmap/issues/3).

If you don't need any of the workflow features, then the frontend is fairly unimportant. You don't need to use or even install the frontend associated with the backend you've chosen—you can just always use the minimal options of pip for installing and [build](https://pypa-build.readthedocs.io/en/latest/) for building.

#### More complicated build needs

So far, we've mainly discussed things from the perspective of pure Python packages. If you have a Python package with more complicated needs, you should still use pyproject.toml-based builds and set up your project metadata as instructed above. However, your more complicated build needs mean that you have additional considerations when choosing a backend. Here are a few pointers that may help:

- If your package needs to compile C or C++ extensions, check out the following:
    - If you are using [CMake](https://cmake.org/) for your build system, then check out [scikit-build-core](https://github.com/scikit-build/scikit-build-core), a PEP 517 build backend that is the successor to the older setuptools-based [scikit-build](https://github.com/scikit-build/scikit-build). You may find helpful discussion in the [proposal for creating scikit-build-core](https://iscinumpy.dev/post/scikit-build-proposal/) by its main author Henry Schreiner.
    - If you are using [Meson](https://mesonbuild.com/) for your build system, then check out [meson-python](https://meson-python.readthedocs.io/en/latest/). This is being used by SciPy. You may find helpful discussion in [this post](https://labs.quansight.org/blog/2021/07/moving-scipy-to-meson) by SciPy core developer Ralf Gommers about choosing Meson and setting it up for SciPy.
    - Setuptools still supports [building extensions modules](https://setuptools.pypa.io/en/latest/userguide/ext_modules.html) using a `setup.py` file even when doing a `pyproject.toml`-based build. Your `setup.py` just only contains extensions-related configuration and lives alongside your `pyproject.toml`.
- Some build backends support running custom code as build hooks during the build process. See documentation for [hatchling](https://hatch.pypa.io/1.6/config/build/#build-hooks) or [pdm-backend](https://pdm-backend.fming.dev/hooks/).

## My setup.py still works. Why bother changing?

It is true that `setup.py` still works. You can still use setuptools' `setup.py` commands to build or install your package, and other tools like build and pip still support it. However, this likely won't be true forever!

* Setuptools' `setup.py` commands are already [deprecated](https://setuptools.pypa.io/en/latest/deprecated/commands.html) and you are warned not to use them.
* Pip's support for `setup.py` is labeled as "legacy" functionality and [warns](https://pip.pypa.io/en/stable/reference/build-system/setup-py/) that "projects should not expect to rely on there being any form of backward compatibility".
* Build's support of `setup.py` is also labeled as ["legacy"](https://pypa-build.readthedocs.io/en/latest/differences.html#fallback-backend) functionality.

`pyproject.toml`-based builds are the future, and they promote better practices for reliable package builds and installs. You should prefer to use them!

---

This should cover the basic concepts you need to understand the package tooling landscape in early 2023. In summary:

- If you want to create a Python package using the modern standards:
    1. Declare a build backend in the `[build-system]` table of your `pyproject.toml` file.
    2. Declare a your project metadata in the `[project]` table of your `pyproject.toml` file.
- To choose a backend to use:
    - If you have a simple project, it doesn't really matter. Pick any backend that supports PEP 517 and PEP 621. Currently popular choices include flit-core, hatchling, pdm-backend, and setuptools (>=61).
    - If you have more sophisticated build feature needs, compare the build-specific features that get configured in the `[tool.*]` tables of `pyproject.toml`.
    - If you want a comprehensive workflow tool that manages other things like virtual environments and dependencies, compare the tools on those features. Poetry, Hatch, and PDM are each currently quite popular, though Poetry isn't yet up-to-date on all standards.

For some examples of converting `setup.py`-based packages to using `pyproject.toml`, check out DrivenData's open-source packages such as [cloudpathlib](https://github.com/drivendataorg/cloudpathlib/pull/324) and [nbautoexport](https://github.com/drivendataorg/nbautoexport/pull/113).

If you are interested in the history of the standards and how we ended up with all of these workflow tools, I recommend reading ["Thoughts on the Python packaging ecosystem"](https://pradyunsg.me/blog/2023/01/21/thoughts-on-python-packaging/). The author, Pradyun Gedam, is a `pip` maintainer and a central participant in a lot of Python packaging developments over the last few years.
