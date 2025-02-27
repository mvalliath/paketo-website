---
title: "Python Buildpack Reference"
menu:
  main:
    parent: reference
    identifier: python-reference
    name: "Python Buildpack"
---

{{% reference_exec_summary bp_name="Paketo Python Buildpack" bp_repo="https://github.com/paketo-buildpacks/python" howto_docs_path="/docs/howto/python" %}}

## Supported Dependencies
The Python buildpack supports several versions of CPython, as well as tools
like Pip and Pipenv. For more details on the specific versions supported in a
given buildpack version, see the [release
notes](https://github.com/paketo-buildpacks/python/releases).

## Buildpack-Set Environment Variables

The Python Buildpack sets a few environment variables during the `build` and
`launch` phases of the app lifecycle. The sections below describe each
environment variable and its impact on your app.

### PYTHONPATH

The [`PYTHONPATH`](https://docs.python.org/3/using/cmdline.html#envvar-PYTHONPATH)
environment variable is used to add directories where python will look for
modules.

* Set by: `CPython`, `Pip`, `Pipenv`, `Poetry`, and `Poetry-Install`
* Phases: `build` and `launch`

The CPython buildpack sets the `PYTHONPATH` value to its installation location,
and the Pip, Pipenv buildpack prepends their `site-packages` location to it.
`site-packages` is the target directory where packages are installed to.

### PYTHONUSERBASE

The [`PYTHONUSERBASE`](https://docs.python.org/3/using/cmdline.html#envvar-PYTHONUSERBASE)
environment variable is used to set the user base directory.

* Set by: `Pip`, `Pipenv`, `Poetry` `Pip Install`, `Pipenv Install`,
* Phases: `build` and `launch`

The value of `PYTHONUSERBASE` is set to the location where these buildpacks install
the application packages so that it can be consumed by the app source code.

## Start Command

The Python Buildpack sets the default start command `python`. This starts the Python
REPL (read-eval-print loop) at launch.

If the application uses `poetry`, the `Poetry Run` buildpack can also be used
to create a start command from single script defined in the `pyproject.toml`
file. See the [`poetry-run`
documentaton](https://github.com/paketo-buildpacks/poetry-run/blob/main/README.md)
for more details.

The Python Buildpack comes with support for
[`Procfile`]({{< ref "docs/howto/configuration#procfiles" >}})
that lets users set custom start commands easily.
