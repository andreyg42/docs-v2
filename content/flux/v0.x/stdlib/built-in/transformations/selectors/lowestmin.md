---
title: lowestMin() function
description: The `lowestMin()` function selects the minimum record from each table in the input stream and returns the lowest `n` records.
aliases:
  - /influxdb/v2.0/reference/flux/functions/transformations/selectors/lowestmin
  - /influxdb/v2.0/reference/flux/functions/built-in/transformations/selectors/lowestmin/
  - /influxdb/v2.0/reference/flux/stdlib/built-in/transformations/selectors/lowestmin/
  - /influxdb/cloud/reference/flux/stdlib/built-in/transformations/selectors/lowestmin/
menu:
  flux_0_x_ref:
    name: lowestMin
    parent: built-in-selectors
weight: 501
flux/v0.x/tags: [selectors]
introduced: 0.7.0
---

The `lowestMin()` function selects the minimum record from each table in the input stream and returns the lowest `n` records.
It outputs a single aggregated table containing `n` records.

_**Function type:** Selector, Aggregate_

```js
lowestMin(
  n:10,
  column: "_value",
  groupColumns: []
)
```

{{% warn %}}
#### Empty tables
`lowestMin()` drops empty tables.
{{% /warn %}}

## Parameters

### n
Number of records to return.

_**Data type:** Integer_

### column
Column by which to sort.
Default is `"_value"`.

_**Data type:** String_

### groupColumns
The columns on which to group before performing the aggregation.
Default is `[]`.

_**Data type:** Array of strings_

## Examples
```js
from(bucket:"example-bucket")
  |> range(start:-1h)
  |> filter(fn: (r) =>
    r._measurement == "mem" and
    r._field == "used_percent"
  )
  |> lowestMin(n:10, groupColumns: ["host"])
```

## Function definition
```js
// _sortLimit is a helper function, which sorts and limits a table.
_sortLimit = (n, desc, columns=["_value"], tables=<-) =>
  tables
    |> sort(columns:columns, desc:desc)
    |> limit(n:n)

// _highestOrLowest is a helper function which reduces all groups into a single
// group by specific tags and a reducer function. It then selects the highest or
// lowest records based on the column and the _sortLimit function.
// The default reducer assumes no reducing needs to be performed.
_highestOrLowest = (n, _sortLimit, reducer, column="_value", groupColumns=[], tables=<-) =>
  tables
    |> group(columns:groupColumns)
    |> reducer()
    |> group(columns:[])
    |> _sortLimit(n:n, columns:[column])

lowestMin = (n, column="_value", groupColumns=[], tables=<-) =>
  tables
    |> _highestOrLowest(
        n:n,
        column:column,
        groupColumns:groupColumns,
        reducer: (tables=<-) => tables |> min(column:column),
        _sortLimit: bottom,
      )
```