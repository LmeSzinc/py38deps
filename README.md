# py38deps

As older versions of Python gradually end of like, many Python dependencies have raised their minimum python version requirements. `py38deps` backports their latest versions to older Python versions, aiming to support at least Python 3.8.

## Maintained dependencies

| Official Repo                                                | Our Repo                                                     | Latest Version | Backport Low To |
| ------------------------------------------------------------ | ------------------------------------------------------------ | -------------- | --------------- |
| [msgspec/msgspec](https://github.com/msgspec/msgspec)        | [LmeSzinc/msgspec](https://github.com/LmeSzinc/msgspec)      | 0.21.1         | cp38            |
| [indygreg/python-zstandard](https://github.com/indygreg/python-zstandard) | [LmeSzinc/python-zstandard](https://github.com/LmeSzinc/python-zstandard) | 0.25.0         | cp38            |
| [python-hyper/hyperframe](https://github.com/python-hyper/hyperframe) | [LmeSzinc/hyperframe](https://github.com/LmeSzinc/hyperframe) | 6.1.0          | cp38            |
| [python-hyper/hpack](https://github.com/python-hyper/hpack) | [LmeSzinc/hpack](https://github.com/LmeSzinc/hpack) | 4.2.0          | cp38            |
| [python-hyper/h2](https://github.com/python-hyper/h2) | [LmeSzinc/h2](https://github.com/LmeSzinc/h2) | 4.4.1          | cp38            |
