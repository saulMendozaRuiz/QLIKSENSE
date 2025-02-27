# PIVOT Function for QlikView/Qlik Sense
### by SAUL ENRIQUE MENDOZA RUIZ @saulMendozaRuiz

This script function provides an efficient way to create dynamic pivot tables in QlikView or Qlik Sense, allowing data transformation from a long format to a wide format.

## Description

The `PIVOT` subroutine converts data from a vertical table into a horizontal pivoted format, similar to the pivot table functionality available in Excel. This is particularly useful for creating more readable visualizations where categories are organized into columns rather than rows.

## Syntax

```qlikview
CALL PIVOT(vTable, xAxis, yAxis, value, aggType);
```

## Parameters

| Parameter | Description |
|-----------|-------------|
| `vTable`  | The name of the table containing the data to pivot. This table will be replaced by the pivoted version. |
| `xAxis`   | The field that will become columns in the resulting table. |
| `yAxis`   | The field that will remain as rows in the resulting table. |
| `value`   | The field containing the values to aggregate. |
| `aggType` | Aggregation type: `0` to count occurrences, `1` to sum values. |

## How It Works

1. Creates a temporary table that groups data by the X and Y axes with the specified aggregation
2. Builds a final table with the unique values from the Y axis
3. Iterates over the unique values of the X axis, adding them as columns to the final table
4. Removes the temporary tables and renames the final table with the original table name

## Usage Example

```qlikview
// Load some sample data
Sales:
LOAD * INLINE [
    Region, Product, Sales
    North, Apples, 150
    North, Pears, 120
    South, Apples, 200
    South, Pears, 180
    East, Apples, 170
    East, Pears, 110
    West, Apples, 190
    West, Pears, 140
];

// Pivot the table to have products as columns
CALL PIVOT('Sales', 'Product', 'Region', 'Sales', 1);
```

### Result before PIVOT:

| Region | Product | Sales |
|--------|---------|-------|
| North  | Apples  | 150   |
| North  | Pears   | 120   |
| South  | Apples  | 200   |
| South  | Pears   | 180   |
| East   | Apples  | 170   |
| East   | Pears   | 110   |
| West   | Apples  | 190   |
| West   | Pears   | 140   |

### Result after PIVOT:

| Region | Apples | Pears |
|--------|--------|-------|
| East   | 170    | 110   |
| North  | 150    | 120   |
| West   | 190    | 140   |
| South  | 200    | 180   |

## Limitations

- The function replaces the original table with the pivoted version
- Does not automatically handle null values
- Column names will be exactly the values from the X axis field

## Code

```qlikview
Sub PIVOT(vTable, xAxis, yAxis, value, aggType)
    // aggType: 0 is for count, 1 is for sum
    If IsNull('$(vTable)') Or IsNull('$(xAxis)') Or IsNull('$(yAxis)') Then
        Trace ### Error: Missing required parameters ###;
        Exit Script;
    Endif
    Let xAxis = '$(xAxis)';
    Let yAxis = '$(yAxis)';
    Let value='$(value)';
    LeT aggType= '$(aggType)';
    NoConcatenate
    temporal:
    Load 
        [$(yAxis)],
        [$(xAxis)],
        If($(aggType)=1,Sum([$(value)]),Count([$(value)])) as aggt        
    Resident $(vTable)
    Group By [$(yAxis)], [$(xAxis)];
    FinalTable:
    Load Distinct [$(yAxis)]
    Resident $(vTable)
    Order by [$(yAxis)];
    FOR i = 1 to FieldValueCount('$(xAxis)')
        LET vState = FieldValue('$(xAxis)', $(i));
        Left Join (FinalTable)
        Load Distinct [$(yAxis)],
             aggt as [$(vState)]
        Resident temporal
        Where [$(xAxis)] = '$(vState)';
    Next
    Drop Table $(vTable), temporal;
    Rename table FinalTable TO $(vTable);
End sub
```

## Contributions

Contributions are welcome. Please feel free to fork, improve the code, and submit a pull request.

