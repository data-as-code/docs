# Data as Code

<div align="center">
  <img src="img/motto.png" alt="drawing" width="450"/>
</div>

Data as Code (DaC) is a paradigm of distributing versioned data as versioned code.

!!! warning "Disclaimer"

```
Currently the focus is on tabular and batch data, and Python code only.

Future extensions may be possible, depending on the community interest.
```

## Consumer - Data Scientist

??? info "Follow along"

````
The code snippets below can be executed on your machine too!
You just need to configure `pip` to point to the pypi registry where we stored the example DaC package. You can do this by running

```shell
❯ export PIP_EXTRA_INDEX_URL=https://gitlab.com/api/v4/projects/43746775/packages/pypi/simple
```

Of course don't forget to create an isolated environment before using `pip` to install the package:

```shell
❯ python -m venv venv && . venv/bin/activate
```
````

Say that the Data Engineers prepared the DaC `dac-example-energy` for you. Install it with

```shell
❯ python -m pip install dac-example-energy
...
Successfully installed ... dac-example-energy-2.0.2 ...
```

Have you noticed the version `2.0.2`? That is the version of your data. This is very important, you can read
[here](#make-releases) why.

Now, how do you grab the data?

```python
>>> from dac_example_energy import load
>>> df = load()
>>> df
                         nrg_bal_name            siec_name geo  TIME_PERIOD  OBS_VALUE
0      Final consumption - energy use   Solid fossil fuels  AL         1990   6644.088
1      Final consumption - energy use   Solid fossil fuels  AL         1991   3816.945
2      Final consumption - energy use   Solid fossil fuels  AL         1992   1067.475
3      Final consumption - energy use   Solid fossil fuels  AL         1993    525.540
4      Final consumption - energy use   Solid fossil fuels  AL         1994    459.514
...                               ...                  ...  ..          ...        ...
71155          Gross available energy  Non-renewable waste  XK         2015      0.000
71156          Gross available energy  Non-renewable waste  XK         2016      0.000
71157          Gross available energy  Non-renewable waste  XK         2017      0.000
71158          Gross available energy  Non-renewable waste  XK         2018      0.000
71159          Gross available energy  Non-renewable waste  XK         2019      0.000

[71160 rows x 5 columns]
```

One more very valuable tool is the `Schema` class. `Schema` is the implementation of a Data Contract, which is a
contract between the data producer and the data consumer. It describes the data, its structure, and the constraints that
the data must fulfill. At the very least it will have a `validate` method that verifies if a given data set fulfills the
data contract. Loaded data is guaranteed to pass the validation.

Let us see what we can do with the `Schema` in the `dac-example-energy` package.

```python
>>> from dac_example_energy import Schema
>>> import inspect
>>> print(inspect.getsource(Schema))
class Schema(pa.SchemaModel):
    source: Series[str] = pa.Field(
        isin=[
            "Solid fossil fuels",
            ...
            "Non-renewable waste",
        ],
        nullable=False,
        alias="siec_name",
        description="Source of energy",
    )
    value_meaning: Series[str] = pa.Field(
        isin=[
            "Gross available energy",
            ...
            "Final consumption - transport sector - energy use",
        ],
        nullable=False,
        alias="nrg_bal_name",
        description="Meaning of the value",
    )
    location: Series[str] = pa.Field(
        isin=[
            "AL",
            ...
            "XK",
        ],
        nullable=False,
        alias="geo",
        description="Location code, either two-digit ISO 3166-1 alpha-2 code or "
        "'EA19', 'EU27_2020', 'EU28' for the European Union",
    )
    year: Series[int] = pa.Field(
        ge=1990,
        le=3000,
        nullable=False,
        alias="TIME_PERIOD",
        description="Year of observation",
    )
    value_in_gwh: Series[float] = pa.Field(
        nullable=True,
        alias="OBS_VALUE",
        description="Value in GWh",
    )
```

In this case [`pandera`](https://pandera.readthedocs.io/en/stable/index.html) has been used to define the Schema. We can

- see which columns are available and even reference their names in our code without cumbersome hardcoded strings:
    ```python
    >>> df[Schema.value_in_gwh]
    0        6644.088
    1        3816.945
    2        1067.475
    3         525.540
    4         459.514
               ...
    71155       0.000
    71156       0.000
    71157       0.000
    71158       0.000
    71159       0.000
    Name: OBS_VALUE, Length: 71160, dtype: float64
    ```
- for each column, we exactly know what to expect. For example, what is the column type, are `None` values allowed, are
    there specific admitted categorical values, etc.;
- we can read a useful description of the column;
- if we install `pandera[strategies]` with `pip`, we can even generate synthetic data that is guaranteed to pass the
    schema validation. This is very useful for testing our code:

```python
>>> Schema.example(size=5)
            siec_name            nrg_bal_name geo  TIME_PERIOD  OBS_VALUE
0         Natural gas  Gross available energy  AL         1990        0.0
1  Solid fossil fuels  Gross available energy  AL         1990        0.0
2  Solid fossil fuels  Gross available energy  AL         1990        0.0
3  Solid fossil fuels  Gross available energy  AL         1990        0.0
4  Solid fossil fuels  Gross available energy  AL         1990        0.0
```

!!! hint "Example data does not look right"

```
The example data above does not look right. Does this mean that there is something wrong in the implementation of the `example` method? Not really! Read [here](#nice-to-have-schemaexample-method).
```

## Producer - Data Engineer

Data as Code is a paradigm, it does not require any special tool or library. Anyone is free to implement it in his own
way and, by the way, may do so in programming languages other than Python. The tools that we built and describe below
(template and CLI tool `dac`) are just **convenience** tools, meaning that they may accelerate your development process,
but are not strictly necessary.

!!! hint "Use [`pandera`](https://pandera.readthedocs.io/en/stable/index.html) to define the Schema"

```
If the dataframe engine (pandas/polars/dask/spark...) you are using is supported by [`pandera`](https://pandera.readthedocs.io/en/stable/index.html), consider  using a [`DataFrameModel`](https://pandera.readthedocs.io/en/stable/dataframe_models.html) to define the Schema.
```

### Write the library

#### 1. Start from scratch

!!! warning "This approach expects you to be familiar with python packaging"

Build you own library, respecting the following constraints:

##### Public function `load`

A public function named `load` is available at the root of package. For example, if you build the package
`dac-my-awesome-data`, it should be possible to do the following:

```python
>>> from dac_my_awesome_data import load
>>> df = load()
```

Notice that it must be possible to call `load()` without any argument, and the version of the returned data must
correspond to the version of the package. This means that data will be different at every build.

##### Data fulfill the `Schema.validate()` method

A public class named `Schema` is available at the root of package and implements the Data Contract. `Schema` has a
`validate` method which takes data as input and raises an error if the Contract is not fulfilled, and returns the data
otherwise.

Notice that the Data Contract should be verified at building time, therefore ensuring that given a Data as Code package,
the data coming from `load()` will always fulfill the Data Contract.

This means that, for example, it must be possible to do the following:

```python
>>> from dac_my_awesome_data import load, Schema
>>> Schema.validate(load())
```

and will never raise an error.

##### [Nice to have] `Schema` contains column names

It is possible to reference the column names from the `Schema` class. For example:

```python
>>> from dac_my_awesome_data import load, Schema
>>> df = load()
>>> df[Schema.column_1]
```

##### [Nice to have] `Schema.example` method

It is possible to generate synthetic data that fulfill the Data Contract. For example:

```python
>>> from dac_my_awesome_data import Schema
>>> Schema.example(size=5)
   column_1  column_2
0         1         2
1         3         4
2         5         6
3         7         8
4         9        10
```

Ideally synthetic data should be such to really stretch the limits of the Data Contract. By this we mean that the
generated data should be as far aways as possible to the real data, but still fulfill the Data Contract. This is useful
to make the tests built using this feature as robust as possible. Also, this will push the developers to improve the
Data Contract, and therefore will make it as reliable as possible. For example, in the
[Consumer](#consumer-data-scientist) section, you may have noticed that the rows have nearly alwyas the same values.
This is unlikely to be the case in the real data, but as it is not envoded in the Schema it may still happen! It would
be probably a good idea to add meaningful constraint checks to the `Schema` class.

#### 2. Use the template

We provide a [Copier](https://copier.readthedocs.io/en/stable/) template to get started quickly.

[Take me to the template :material-cursor-default-click:](https://gitlab.com/data-as-code/template/src){ .md-button }

#### 3. Use the [`dac`](https://github.com/data-as-code/dac) CLI tool

Out CLI `dac` tool is a convenience tool capable of building a python package that respects the Data as Code paradigm.

[Take me to the `dac` CLI tool :material-cursor-default-click:](https://github.com/data-as-code/dac){ .md-button }

#### Compare template and `dac pack`

Which one should you use?

|                              |         Template          |      `dac pack`       |
| :--------------------------: | :-----------------------: | :-------------------: |
|          Simplicity          | :material-thumbs-up-down: |  :material-thumb-up:  |
| Possibility of customization |    :material-thumb-up:    | :material-thumb-down: |

### Make Releases

Choosing the right release version plays a crucial role in the Data as Code paradigm.

Semantic versioning is used to communicate significant changes:

|           | Reason                                                                               |
| :-------: | :----------------------------------------------------------------------------------- |
| __Patch__ | Fix in the data. Intended content unchanged                                          |
| __Minor__ | Change in the data that does not break the Data Contract - typically, new batch data |
| __Major__ | Change in the Data Contract, or any other breaking change                            |

__Typically Patch and Major releases involve a manual process, while Minor releases can be automated.__ Our
[`dac` CLI tool](https://github.com/data-as-code/dac) can help you with the automated releases. Explore the command
`dac next-version`.

## Why distributing Data as Code?

- __Data Scientists have a very convenient way to ensure that their code is not run on incompatible data.__ They can
    simply include the Data as Code into their dependencies (e.g. `dac-example-energy~=1.0`), and then installation of
    their code with incompatible data will fail (e.g. `dac-example-energy` version `2.0.0`).
- __Data pipelines can receive data updates without breaking.__ Data can subscribe to a major version of the data, and
    can receive updates without breaking changes.
- __It provides a way to maintain multiple release streams__ (e.g. `1.X.Y` and `2.X.Y`). This is useful when a new
    version of the data is released, but some users are still using the old version. In this case, the data engineer can
    keep releasing updates for both versions, until all users have migrated to the new version.
- __The code needed to load the data, the data source, and locations are abstracted away from the consumer.__ This mean
    that the Data Producer can start from local files, transition to SQL database, cloud file storage, or kafka topic,
    without having the consumer to notice it or need to adapt its code.
- _If you provide column names in `Schema`_ (e.g. `Schema.column_1`), __the consumer's code will not contain hard-coded
    column names__, and changes in data source field names won't impact the user.
- _If you provide the `Schema.example` method_, __the consumer will be able to build robust code by writing unit testing
    for their functions__. This will result in a more robust data pipeline.
- _If the description of the data and columns is included in the `Schema`_, __data will be self-documented, from a
    consumer perspective__.
