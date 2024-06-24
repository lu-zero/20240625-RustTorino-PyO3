---
marp: true
size: 16:9
style: |
    .columns {
      display: grid;
      grid-template-columns: repeat(2, minmax(0, 1fr));
      gap: 1rem;
    }
    section > h1 {
      flex: 1 0 auto;
      padding: 0;
      margin: 0;
      order: -999999;
    }
    section:has(> h1)::before {
      flex: 1 0 auto;
      display: block;
      content: '';
      order: 999999;
    }
---

![bg left:40% 80%](assets/rust-torino.svg)

# <!-- fit --> Wrapping Rust
## with PyO3 and Maturin

---
<!-- footer: '![height:1.5em](assets/rust-torino.svg) **Wrapping Rust with PyO3 and Maturin** - Luca Barbato' -->
# <!-- fit --> Intro

---
<!-- header: '**Intro**: Who am I?' -->
# <!-- fit --> Who am I?

* Luca Barbato
    * lu_zero@gentoo.org
    * lu_zero@videolan.org

---
<!-- header: '**Intro**: Rust?' -->
# <!-- fit --> Why Rust?
* Among the most loved languages
    * According to stack overflow
* Performance, Reliability, Productivity
    * Pick 3
* Write once, run almost everywhere

---
<!-- header: '**Intro**: Python?' -->
# <!-- fit --> Why Python?

* Very popular as well
    * More than Rust?
* Easy to grasp
    * Hard to master?
* Faster to write
    * Slower to execute

---
<!-- header: '**Intro**: Rust + Python?' -->
# <!-- fit --> Rust + Python = ??

* It is common practice to rewrite python inner loops in C for speed
    * But at what cost?
* Rust is as fast as C
    * But with better abstractions
* Can Rust make Python developers more at home?

---

# Some comparisons

<div class="columns">
<div>

## Rust

``` rust
fn foo(a: u16, b: u32) -> u64 {
    a as u64 + b as u64
}

struct S {
    a: String,
    b: Vec<u8>,
}

impl S {
    fn blah(&self) {
        ...
    }
}

```
</div>
<div>

## Python

``` python
def foo(a: int, b: int) -> int:
    a + b


class S:
    a: str
    b: bytes



    def blah(self) -> None:
        ...



```
</div>
</div>

---
# <!-- fit --> They are different enough

* Not as different as C
    * Python type annotations are similar to Rust types
    * Iterators, Lamdas, and more exist
    * `cargo` and `pip`/`uv` aren't that apart
* But not **that** similar
    * Compile vs Interpret
    * Traits vs Inheritance
    * Static typing vs Dynamic typing

---
# <!-- fit --> How to bridge between the two?

* Python has many ways to bind with C
    * [CFFI](https://pypi.org/project/cffi/)
    * [Cython](https://pypi.org/project/Cython/) and [mypyc](https://mypyc.readthedocs.io)
* Rust can use the same C ABI easily
    * If you want to go the hard way you can use directly [cbindgen](https://crates.io/crates/cbindgen)
        * It can generate `Cython` bindings directly or you use `cffi` straight.
    * If you want to have a better experience
        * [uniffi](https://mozilla.github.io/uniffi-rs/latest/) is good if you target multiple languages
        * [PyO3](https://pyo3.rs) if you want to interact with Python in both ways.
* In any case you want to build everything together
    * [maturin](https://crates.io/crates/maturin) takes care of it for you
---
<!-- header: '' -->

# <!-- fit --> Maturin + PyO3

* [maturin](https://crates.io/crates/maturin) is a Rust crate that supports [PEP 621](https://peps.python.org/pep-0621/).
    * So you can build, package and ship to [pypi](https://pypi.org) your Rust code.
    * Incidentally it ship [itself](https://pypi.org/project/maturin) to pypi
    * And since it supports `PEP 621` you can use it with the usual tools (e.g. [build](https://pypi.org/project/build)
* [PyO3](https://crates.io/crates/pyo3) is a bridge between Python and Rust that goes both direction
    * You can expose Rust to Python
    * You can call Python code from Rust
    * It provides pre-made mappings between common [types](https://pyo3.rs/latest/conversions/tables)

---
# <!-- fit --> Case Study: oruuid

* [oruuid](https://github.com/lu-zero/oruuid) is an implementation of [Python's uuid](https://docs.python.org/3/library/uuid.html) using the [uuid crate](https://crates.io/crates/uuid)
    * [Riccardo](https://x.com/rmistaken) noted that `uuid_v7` is missing from Python.
    * The API is good to start simple and gradually increase complexity
    * Since the code itself is minimal we can use it to also try to measure the overhead
* Ingredients
    * `PyO3` to write the bindings
    * `maturin` to build everying
    * `pytest` to have a couple of tests going
    * `timeit` to quickly get some numbers

---
<!-- header: '**oruid**: Setup' -->
# Setting up the project
* [maturin](https://crates.io/crates/maturin) has a `new` command we can use
    ``` sh
    $ maturin new oruuid
    ```
* It will ask the kind of bindings, we are going to use `PyO3` and we get
    ```
    ├── .github
    │   └── workflows
    │       └── CI.yml
    ├── .gitignore
    ├── Cargo.toml
    ├── pyproject.toml
    └── src
        └── lib.rs
    ```
* **Notice** we have a `pyproject.toml` and `Cargo.toml` generated for us.
---
* We add a dependency on [uuid](https://crates.io/crates/uuid):
    ``` sh
    $ cargo add uuid --features v1,v4,v7
    ```
    * The crate let you enable specific implementation using the [features](https://doc.rust-lang.org/cargo/reference/features.html) system.
* We start enabling `v1`, `v4` and `v7`.
    * `v1` and `v7` both rely on a timestamp provided, or `now` is used as implicit input.
    * `v4` is just encoding a random number and in both Rust and Python it is a thin layer over the OS `getrandom(2)`.
    * `v7` is not implemented yet so it is our excuse to write all of this.
* We want to stay compatible with Python [uuid](https://docs.python.org/3/library/uuid.html)
    * Nothing to add here, it is part of the standard library.
---
<!-- header: '**oruid**: API' -->

The two APIs are fairly similar:

<div class="columns">
<div>

## Python
``` python
class UUID:
    __slots__ = ('int', 'is_safe', '__weakref__')

    def __init__(self, hex=None, bytes=None, bytes_le=None, fields=None,
                   int=None, version=None,
                   *, is_safe=SafeUUID.unknown):
    ...


def uuid1(node=None, clock_seq=None):
    ...

    if node is None:
        node = getnode()
    return UUID(fields=(time_low, time_mid, time_hi_version,
                        clock_seq_hi_variant, clock_seq_low, node), version=1)

def uuid4():
"""Generate a random UUID."""
return UUID(bytes=os.urandom(16), version=4)

```

</div>
<div>

## Rust
``` rust
pub struct Uuid(Bytes);

impl Uuid {
    ...
    pub const fn as_u128(&self) -> u128 {
        u128::from_be_bytes(*self.as_bytes())
    }
    ...
// in src/v1.rs
    pub fn new_v1(ts: Timestamp, node_id: &[u8; 6]) -> Self {
        ...
    }
    pub fn now_v1(node_id: &[u8; 6]) -> Self {
        let ts = Timestamp::now(crate::timestamp::context::shared_context());

        Self::new_v1(ts, node_id)
    }
    ...
// in src/v4.rs
    pub fn new_v4() -> Uuid {
        Uuid::from_u128(
            crate::rng::u128() & 0xFFFFFFFFFFFF4FFFBFFFFFFFFFFFFFFF | 0x40008000000000000000,
        )
    }
}
```
</div>
</div>

---
# High level
* Ideally we should provide a `uuid7` that returns a [UUID](https://docs.python.org/3/library/uuid.html#uuid.UUID)
    * The constructor for [UUID](https://docs.python.org/3/library/uuid.html#uuid.UUID) can take a u128 (big endian)
    * [Uuid](https://docs.rs/uuid/latest/uuid/struct.Uuid.html) can provide a `u128`
    * We need to call the Rust `Uuid::now_v7`, feed the result to the `UUID(int=)` and we are done!
* In steps we want to:
    1) Have our `oruuid` module
    2) Add a `uuid7()` function to it
    3) Make the function return the Python UUID correctly populated
---

<!-- header: '**oruid**: Code' -->

# Writing Rust for Python with PyO3

* PyO3 provides all the building blocks we need
    * An API to interact with the Python interpreter
    * A way to map Rust types to Python types and back
    * [procedural macros](https://doc.rust-lang.org/reference/procedural-macros.html) to make all simpler
* We need to decorate our function with `#[pyfunction]` and register it to the module
    ``` rust
    #[pyfunction]
    fn uuid7() -> PyResult<Py<PyAny>> { ... }

    #[pymodule]
    fn oruuid(m: &Bound<'_, PyModule>) -> PyResult<()> {
        m.add_function(wrap_pyfunction!(uuid7, m)?)?;
        Ok(())
    }
    ```

---

Since we want to return the Python `UUID` we have to instantiate it
``` rust
#[pyfunction]
fn uuid7() -> PyResult<Py<PyAny>> {
    let uuid = uuid::Uuid::now_v7();
    Python::with_gil(|py| {
        let kwargs = PyDict::new_bound(py);
        kwargs.set_item("int", uuid.as_u128())?;
        let pyuuid = PyModule::import_bound(py, "uuid")?;
        pyuuid
            .getattr("UUID")?
            .call((), Some(&kwargs))
            .map(|u| u.unbind())
    })
}
```
* **Note:** for simplicity I'm glossing over receiving arguments

---
<!-- header: '**oruid**: Building and Testing' -->
# Building using maturin

* So far we have our bit of code in `src/lib.rs` and `Cargo.toml` with `uuid`
    * We should build it
    * And also test it
* Time to use `maturin build`
    * first let's set up our environment:
        ``` sh
        $ python -m venv .venv
        $ . .venv/bin/activate
        ```
    * we can use either `maturin build`
        ``` sh
        $ maturin build
        ```
        And see if everything compiles

---
# Testing with pytest

* `maturin` is all about building
    * If we want to test we can use any testing harness we like
* I picked `pytest`:
    ``` sh
    $ pip install pytest
    ```
* Since we do not have a default layout I use `test` for Python and `tests` for Rust integration tests
    ```
    $ mkdir test/
    $ touch test/__init__.py
    $ vim test/test_uuid7.py
    ```
---
* We cannot test much beside at least making sure it is version 7:
    ``` python
    # test/test_uuid7.py
    from oruuid import uuid7

    def test_uuid7():
      u = uuid7()
      assert u.version == 7
    ```
* We use `maturin develop` to have our package reachable:
    ``` sh
    $ maturin develop
    ```
    * **NOTE:** if you want to benchmark remember to pass `-r` to it.
* If everything goes well a green test will greet us.
  ```
  $ pytest
  ```

---
<!-- header: '**oruid**: Extra' -->
# Exercise 1

* Let's try to implement `uuid4`
    * by calling the Python code and exposing it as `uuid4p`.
    * by calling the Rust code and exposing it as `uuid4r`.
* Let's benchmark both using `timeit` and compare with Python:
    ``` sh
    $ maturin develop -r
    $ python -m timeit "import oruuid; oruuid.uuid4r()"
    $ python -m timeit "import oruuid; oruuid.uuid4p()"
    $ python -m timeit "import uuid; uuid.uuid4()"
    ```
* This way we can see the overhead

---
# Exercise 2

* Let's add a [signature](https://pyo3.rs/v0.21.2/function/signature) to `v7`
    * Rust has the [Option](https://doc.rust-lang.org/std/option/enum.Option.html) enum for it
        * PyO3 helpfully maps it to a `arg=None` by default
    * `#[pyo3(signature = {argument string})]` can be used to express more complex signatures
* We have to wrap the [Timestamp](https://docs.rs/uuid/latest/uuid/timestamp/struct.Timestamp.html) and [ContextV7](https://docs.rs/uuid/latest/uuid/timestamp/context/struct.ContextV7.html) structs and expose them
    * Chopping the methods to those that apply to `v7`.

---
<!-- header: '' -->
# <!-- fit --> Conclusion

* We saw how PyO3 makes simple to use Python from Rust and to expose Rust functions to Python
* We saw how maturin makes simple building Rust crates as Python packages
---

# <!-- fit --> Questions?
---

# <!-- fit --> Thank you!
