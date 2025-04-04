# Periods

When you run an OpenFisca simulation to compute a variable, you need to define the period of interest for this variable.

OpenFisca gives you a convenient string syntax to define periods.

The general structure is `unit:instant:quantity` (see the [definition of a period](../key-concepts/periodsinstants.md)). Some of the structure can be omitted.

The main valid period formats are referenced in this table:

| Period format     | Period type     | Example             | Represents                                       | Disambiguation                                                        |
|-------------------|-----------------|---------------------|--------------------------------------------------|-----------------------------------------------------------------------|
| `AAAA`            | Calendar year   | `'2010'`            | The year 2010.                                   | From the 1st of January 2010 to the 31st of December 2010, inclusive. |
| `AAAA-MM`         | Month           | `'2010-04'`         | The month of April 2010.                         | From the 1st of April 2010 to the 30th of April 2010, inclusive.      |
| `AAAA-MM-DD`      | Day             | `'2010-04-06'`      | The day of April 6th 2010.                | The whole day of the 6th of April 2010.      |
| `year:AAAA-MM`    | Rolling year    | `'year:2010-04'`    | The 1 year period starting in April 2010. | From the 1st of April 2010 to the 31st of March 2011, inclusive       |
| `year:AAAA:N`     | N years         | `'year:2010:3'`     | The years 2010, 2011 and 2012.                   | From the 1st of January 2010 to the 31st of December 2012, inclusive. |
| `year:AAAA-MM:N`  | N rolling years | `'year:2010-04:3'`  | The three years period starting in April 2010.   | From the 1st of April 2010 to the 31st of March 2013, inclusive.      |
| `month:AAAA-MM:N` | N months        | `'month:2010-04:3'` | The three months from April to June 2010.        | From the 1st of April 2010 to the 30th of June 2010, inclusive.       |
| `month:AAAA-MM-DD:N` | N months        | `'month:2010-04-15:3'` | The three months from mid April to mid July 2010.    | From the April 15th 2010 to the 14th of July 2010, inclusive. |
| `day:AAAA-MM-DD:N` | N days        | `'day:2010-04-01:15'` | The first 15 days of April 2010. | From the 1st of April 2010 to the 15th of April 2010, inclusive.       |
| `ETERNITY` | Forever        | `ETERNITY` | All of time.        | All past, present and future day, month or year. |

The starting instant can be shortened to a month or a year. Then, OpenFisca will implicitly use the first day of the first month.

> Internally, periods come from `openfisca_core.periods` and are stored as:
> - a unit (`DAY`, `MONTH`, `YEAR`)
> - a starting `Instant`
> - a quantity of units.

## Using periods in a simulation

As an example, here is how to calculate a `housing_allowance` on a given month:

```py
simulation.calculate('housing_allowance', '2019-05')
```

This is equivalent to this more precise variant:

```py
simulation.calculate('housing_allowance', 'month:2019-05-01:1')
```

## Periods in variable definitions

With OpenFisca you can calculate variables such as `salary` that can change over time. You can also evaluate the impact of this `salary` on the `income_tax`.

These variables can evolve through time at a different periodicity. Here for example, `salary` changes from month to month while the `income_tax` is calculated on a yearly basis.

Therefore, all OpenFisca variables have a `definition_period` attribute:
  - `definition_period = DAY`: The variable may have a different value each day.
  - `definition_period = MONTH`: The variable may have a different value each month. *For example*, the salary of a person. When `formula` is executed, the parameter `period` will always be a whole month. Trying to compute `salary` with a period that is not a month will raise an error before entering `formula`.
  - `definition_period = YEAR`: The variable is defined for a year or it has always the same value every months of a year. *For example*, if taxes are to be paid yearly, the corresponding variable is yearly. When `formula` is executed, the parameter `period` will always be a whole year (from January 1st to December 31th).
  - `definition_period = ETERNITY`: The value of the variable is constant. *For example*, the date of birth of a person never changes. `period` is still the 2nd parameter of `formula`. However when `formula` is executed, the parameter `period` can be anything and it should not be used.

```py
...

class salary(Variable):
    value_type = float
    entity = Person
    label = u"Salary for a month"
    definition_period = MONTH

    def formula(person, period):
        ...
```

Each formula calculates the value of a variable for a period the size of **the given definition period**. This period is always the second argument of the formulas.

## Periods in formulas

### Calculate dependencies for a period different than the variable's `definition_period`

Calling a formula with a period that is incompatible with the attribute `definition_period` will cause an error. For instance, if we assume that a person `salary` is paid monthly:

```py
class taxes(Variable):
    value_type = float
    entity = Person
    label = u"Taxes for a whole year"
    definition_period = YEAR

    def formula(person, period):  # period is a year because definition_period = YEAR
        salary_past_year = person('salary', period)  # salary is a monthly variable. This will cause an error.
        ...
```

However, sometimes, we do need to estimate a variable for a different period than the one it is defined for.

We may for example want to get the sum of the salaries perceived on the past year, or the past 3 months. The option `ADD` tells OpenFisca to split the period into months, compute the variable for each month and sum up the results:

```py
class taxes(Variable):
    value_type = float
    entity = Person
    label = "Taxes for a whole year"
    definition_period = YEAR

    def formula(person, period):  # period is a year because definition_period = YEAR
        salary_last_year = person('salary', period, options = [ADD])
        ...
```

The option `DIVIDE` allows you to do the opposite: evaluating a quantity for a month while the variable is defined for a year. OpenFisca computes the variable for the whole year that contains the specified month and then divides the result by 12.

```py
class salary_net_of_taxes(Variable):
    value_type = float
    entity = Person
    label = u"Monthly salary, net of taxes"
    definition_period = MONTH

    def formula(person, period):  # period is a month because definition_period = MONTH
        # The variable taxes is computed on a year, monthly_taxes equals the 12th of that result
        monthly_taxes = person('taxes', period, options = [DIVIDE])

        # salary is a monthly variable, period is a month: no option is required
        salary = person('salary', period)

        return salary - monthly_taxes
```


### Calculate dependencies for a specific period

It happens that the formula to calculate a variable at a given period needs the value of another variable for another period. Usually, the second period is defined relatively to the first one (previous month, last three month, current year).

For instance, we want to compute an unemployment benefit that equals half of last year's salary, if the person had no income for the past 3 months.

```py
class unemployment_benefit(Variable):
    value_type = float
    entity = Person
    label = u"Unemployment benefit"
    definition_period = MONTH

    def formula(person, period):
        salary_last_3_months = person('salary', period.last_3_months)
        salary_last_year = person('salary', period.last_year)

        is_unemployed = (salary_last_3_months == 0)
        return 0.5 * salary_last_year * is_unemployed
```

You can generate any period with the following properties and methods:

| Period                            | Meaning                                                      |
|-----------------------------------|--------------------------------------------------------------|
| `period.first_month`              | First month-length period that includes the start of `period`|
| `period.last_month`               | Month preceding `period.first_month`                          |
| `period.this_year`                | First year-length period that includes the start of `period` |
| `period.last_year`                | Year preceding `period.this_year`                            |
| `period.n_2`                      | 2 years before `period.this_year`                            |
| `period.last_3_months`            | The three-month period preceding `period.this_month`         |
| `period.offset(n, 'month')`       | `period` translated by n months (backwards if n <0)          |
| `period.offset(n, 'year')`        | `period` translated by n years (backwards if n <0)           |
| `period.start.period('year')`     | Year-long period starting a the same time than `period`      |
| `period.start.period('month')`    | Month-long period starting a the same time than `period`     |
| `period.start.period('year', n)`  | n-year-long period starting a the same time than `period`    |
| `period.start.period('month', n)` | n-month-long period starting a the same time than `period`   |

You can find more information on the `Period` object in the [reference documentation](../openfisca-python-api/periods).


## Automatically process variable inputs defined for periods not matching the `definition_period`

By default, when you provide a simulation input, you won't be able to set a variable value for a period that doesn't match its `definition_period`.

For instance, if the `definition_period` of `salary` is `MONTH`, and you input a value for `salary` for `2015` or more than one month, an error will be raised.

It is however possible to define an automatic behaviour to cast yearly inputs into monthly values. To do this, add a `set_input` class attribute to a variable.

* `set_input = set_input_divide_by_period`: the 12 months are set equal to 1/12th of the input value,
* `set_input = set_input_dispatch_by_period`: the 12 months are set equal to input value.

For instance, let's slightly modify the code of `salary`:
```py
class salary(Variable):
    value_type = float
    entity = Person
    label = u"Salary for a month"
    definition_period = MONTH
    set_input = set_input_divide_by_period

    def formula(person, period):
        ...
```

We can now provide an input for `2015` for `salary`: no error will be raised, and the value will be automatically split between the 12 months of `2015`.

This example sets 3 years salaries for two persons and calculates how much they get for one month:

```py
# Each person earned 60,000 between 2014 and 2016
simulation.set_input('salary', 'year:2014:3', [60000.0, 60000.0])

one_month_salaries = simulation.calculate('salary', '2014-01')
print(one_month_salaries)  # prints [1666.6666 1666.6666]
```

This syntax is also available in [YAML tests](writing_yaml_tests.md):

```yaml
- name: Income tax over time
  period: 2016-01
  input:
    salary:
      year:2014:3: 100000 # This person earned 100,000 between 2014 and 2016
  output:
    income_tax:
      2014-01: 388.8889
      2015-01: 416.6667 # The income tax rate changes in 2015
      2016-01: 416.6667
      2017-01: 0 # The salary is not set for this period and defaults to 0
```
