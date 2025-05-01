# Mypy's PEP 561 NumPy issue repro

From <https://typing.python.org/en/latest/spec/distributing.html#import-resolution-ordering> (which superseeds PEP 561):

> 1. [Stubs](https://typing.python.org/en/latest/spec/glossary.html#term-stub) or Python source manually put in the beginning of the path. Type checkers SHOULD provide this to allow the user complete control of which stubs to use, and to patch broken stubs or [inline](https://typing.python.org/en/latest/spec/glossary.html#term-inline) types from packages. In mypy the `$MYPYPATH` environment variable can be used for this.
> 2. User code - the files the type checker is running on.
> 3. Typeshed stubs for the standard library. These will usually be vendored by type checkers, but type checkers SHOULD provide an option for users to provide a path to a directory containing a custom or modified version of typeshed; if this option is provided, type checkers SHOULD use this as the canonical source for standard-library types in this step.
> 4. **[Stub](https://typing.python.org/en/latest/spec/glossary.html#term-stub) packages - these packages SHOULD supersede any installed inline package. They can be found in directories named `foopkg-stubs` for package `foopkg`.**
> 5. **Packages with a `py.typed` marker file - if there is nothing overriding the installed package, and it opts into type checking, the types bundled with the package SHOULD be used (be they in `.pyi` type stub files or inline in `.py` files).**
> 6. If the type checker chooses to additionally vendor any third-party stubs (from typeshed or elsewhere), these SHOULD come last in the module resolution order.

Here, [NumPy](https://github.com/numpy/numpy)'s bundled stubs fall under 5., and [NumType](https://github.com/numpy/numtype/)'s [`numpy-stubs`](https://github.com/numpy/numtype/tree/2fb0af907b68558e4b0778c9dc4b21262105adb2/src/numpy-stubs) fall under 4..
So NumType should be prioritized over the NumPy.

Pyright behavior confirms that this is indeed what should happen.

But mypy appears to incorrectly prioritize 5. (`numpy/__init__.pyi`) over 4. (`numpy-stubs/__init__.pyi` from NumType)  &mdash;
it should be the other way around.

## Env setup

Using python 3.10+ and [`uv`](https://docs.astral.sh/uv/)

```bash
uv venv .venv
source .venv/bin/activate
```

If you don't have `uv` installed, then replace `uv` with `python -m`, and later on you can replace `uv pip` with `pip`.

### Install  (without NumType)

Install `mypy 1.5.0`, `pyright 1.1.400`, and `numpy 2.2.5`:

```bash
uv pip uninstall numtype
uv pip install --reinstall -r requirements.txt
```

(the `--reinstall` flag is only needed after the nuclear workaround)

To confirm:

```bash
$ uv pip list
Package           Version
----------------- -------
mypy              1.15.0
mypy-extensions   1.1.0
nodeenv           1.9.1
numpy             2.2.5
pyright           1.1.400
typing-extensions 4.13.2
```

### Install  (with NumType)

Install `mypy 1.5.0`, `pyright 1.1.400`, `numpy 2.2.5`, and `numtype @ 2fb0af9`

```bash
uv pip install -r requirements-numtype.txt
```

To confirm:

```bash
$ uv pip list
Package           Version
----------------- ------------
mypy              1.15.0
mypy-extensions   1.1.0
nodeenv           1.9.1
numpy             2.2.5
numtype           2.2.5.0.dev0
pyright           1.1.400
typing-extensions 4.13.2
```

## Bug demo

In NumPy's bundled stubs (`numpy==2.2.5`), the type of `np.True_` ([src](https://github.com/numpy/numpy/blob/7be8c1f9133516fe20fd076f9bdfe23d9f537874/numpy/__init__.pyi#L1161)) is called `numpy.bool` (shadowing `builtin.bool`).

In NumType's `numpy-stubs` (`2fb0af9`), the same `numpy.True_` is defined in [`numpy-stubs/_core/numeric.pyi`](https://github.com/numpy/numtype/blob/2fb0af907b68558e4b0778c9dc4b21262105adb2/src/numpy-stubs/_core/numeric.pyi#L623) (re-exported in [`__init__.pyi`](https://github.com/numpy/numtype/blob/2fb0af907b68558e4b0778c9dc4b21262105adb2/src/numpy-stubs/__init__.pyi#L77)), and its type is a `np.bool_` (note the trailing underscore).

So with a `reveal_type(np.True_)` (in the `main.pyi` of this repo) we'll be able to tell which stubs used.

### Pyright (without NumType)

NumType: No

```bash
$ pyright .
/home/joren/Workspace/mypy-numtype/main.pyi
  /home/joren/Workspace/mypy-numtype/main.pyi:5:17 - information: Type of "np.True_" is "bool[Literal[True]]"
```

`numpy.bool` => NumPy's bundled stubs are used :white_check_mark:

### Pyright (with NumType)

```bash
$ pyright .
/home/joren/Workspace/mypy-numtype/main.pyi
  /home/joren/Workspace/mypy-numtype/main.pyi:3:13 - information: Type of "np.True_" is "bool_[Literal[True]]"
```

`numpy.bool_` => NumType's `numpy-stubs` are used :white_check_mark:

---

### Mypy `2.2.5` (without NumType)

NumType: No

```bash
$ rm -rf .mypy_cache
$ mypy .
main.pyi:3: note: Revealed type is "numpy.bool[Literal[True]]"
```

`numpy.bool` => NumPy's bundled stubs are used :white_check_mark:

### Mypy `2.2.5` (with NumType)

```bash
$ rm -rf .mypy_cache
$ mypy main.pyi
main.pyi:3: note: Revealed type is "numpy.bool[Literal[True]]"
```

`numpy.bool` => NumPy's bundled stubs are used :x:

### Mypy (`jorenham:fix-18997`) (with NumType)

Install the PR branch with the fix (python/mypy#19001):

```bash
uv pip install "git+https://github.com/jorenham/mypy@fix-18997"
```

```bash
$ rm -rf .mypy_cache
$ mypy main.pyi
main.pyi:3: note: Revealed type is "numpy.bool_[Literal[True]]"
```

`numpy.bool_` => NumType's `numpy-stubs` are used :white_check_mark:

## The nuclear workaround

<details>
<summary>The only workaround I was able to find (after several full days of trying), was to remove all <code>.pyi</code> from numpy's local installation directory. But this is not something I can realistically ask the users of NumPy to do.</summary>

First do a dry run:

```bash
$ find ./.venv/**/site-packages/numpy -name "*.pyi" -type f
./.venv/lib/python3.13/site-packages/numpy/matlib.pyi
./.venv/lib/python3.13/site-packages/numpy/fft/_helper.pyi
./.venv/lib/python3.13/site-packages/numpy/fft/_pocketfft.pyi
[...]
```

If all is good, press the red button:

```bash
$ find ./.venv/**/site-packages/numpy -name "*.pyi" -type f -delete
```

Re-run pyright:

```bash
$ pyright .
/home/joren/Workspace/mypy-numtype/main.pyi
  /home/joren/Workspace/mypy-numtype/main.pyi:3:13 - information: Type of "np.True_" is "bool_[Literal[True]]"
```

Good; that's exactly the same result as before (with NumType).

Now run mypy again:

```bash
$ rm -rf .mypy_cache
$ mypy .
main.pyi:3: note: Revealed type is "numpy.bool_[Literal[True]]"
Success: no issues found in 1 source file
```

This is indeed the correct output, which for the 2nd time demonstrates that mypy does not comply with PEP 561.
</details>
