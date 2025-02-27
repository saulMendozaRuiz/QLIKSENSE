# Incremental Data Loading Framework for QlikView/Qlik Sense
### by SAUL ENRIQUE MENDOZA RUIZ
This framework provides an efficient way to implement incremental data loading in QlikView or Qlik Sense ETL processes, significantly reducing load times and resource consumption by processing only new or modified data.

## Description

The `PATCH` subroutine enables advanced incremental loading capabilities by tracking the latest timestamp in your data and only loading records that have been updated since the previous execution. It also includes automatic handling of both full and incremental loads, making it highly reusable across different tables.

## Syntax

```qlikview
CALL PATCH(vTable, vFolder, vUpdatedAt, vID);
```

## Parameters

| Parameter    | Description |
|--------------|-------------|
| `vTable`     | The name of the table and corresponding loading subroutine (e.g., 'customers' would use customersLoad) |
| `vFolder`    | Folder path where QVD files will be stored |
| `vUpdatedAt` | Field name containing the last-updated timestamp in your data |
| `vID`        | Primary key field used for deduplication during patching |

## How It Works

1. **Determines Load Type**: Checks if a previous maximum timestamp exists
   - If exists → Incremental load
   - If not exists → Full load (going back 2 years by default)

2. **Executes Table-Specific Load**: Calls a custom subroutine named after your table that contains the actual SQL query

3. **Records Maximum Timestamp**: Stores the highest timestamp value for the next execution

4. **Stores Partial Load**: Keeps a backup of each incremental load with timestamp in filename

5. **Patching Process**: Merges new/updated records with existing data, using primary key for deduplication

6. **Updates Main QVD**: Stores the combined dataset as the main QVD file

## Usage Example

### Step 1: Define the PATCH subroutine

```qlikview
sub PATCH(vTable, vFolder, vUpdatedAt, vID)
    SET initLoad=0;
    // Determine load type based on presence of maxUpdatedAt QVD file
    IF FileSize('$(vFolder)/maxUpdatedAt_$(vTable).qvd')> 0 THEN
    
        Trace ** Incremental Load - $(vTable) **;
        // Retrieve the most recent updatedAt value for incremental loading
        maxUpdatedAt:
        LOAD maxUpdatedAt FROM [$(vFolder)/maxUpdatedAt_$(vTable).qvd] (qvd);
        Let vmaxUpdatedAt = Peek('maxUpdatedAt');
        Drop Table maxUpdatedAt;  
    ELSE
        // Initialize vmaxUpdatedAt for a full load case
        Trace ** Full Load - $(vTable) **;
        Let vmaxUpdatedAt = Date(Addyears(YearStart(Today(1)), -2), 'YYYY-MM-DD 00:00:00');
        Let initLoad=1;
    ENDIF
    
    //Load the SQL query
    CALL $(vTable)Load('$(vUpdatedAt)', '$(vmaxUpdatedAt)');
    
    //Store the updatedAt timestamp in the proper folder
    maxUpdatedAt:
    LOAD TimeStamp(Max($(vUpdatedAt)) - Interval(0.0005),'YYYY-MM-DD hh:mm:ss') as maxUpdatedAt Resident $(vTable);
    Store maxUpdatedAt into [$(vFolder)/maxUpdatedAt_$(vTable).qvd](QVD);
    Drop Table maxUpdatedAt; 
    
    //Store the .qvd table
    IF initLoad=0 then
        LET vNow = Date(Now(1), 'YY.MM.DD_HH.MM.SS');
        Store $(vTable) into [$(vFolder)/partial_loads/$(vTable)_$(vNow).qvd](QVD);
        Concatenate($(vTable))
        LOAD * FROM [$(vFolder)/$(vTable).qvd](qvd) Where Not Exists($(vID)); // patch step
    end if
    
    Store $(vTable) into [$(vFolder)/$(vTable).qvd](QVD);
    Drop Table $(vTable);
end sub
```

### Step 2: Create a table-specific load subroutine

```qlikview
Sub customersLoad(vUpdatedAtField, vThreshold)
    customers:
    SQL SELECT 
        customer_id,
        customer_name,
        email,
        phone,
        created_at,
        updated_at
    FROM `customers`
    WHERE $(vUpdatedAtField) >= '$(vThreshold)';
End sub
```

### Step 3: Call the PATCH function in your main script

```qlikview
// Load customers with incremental loading
CALL PATCH('customers', 'lib://QVD_Storage/', 'updated_at', 'customer_id');
```

## Benefits

- **Efficiency**: Drastically reduces load times by only processing new or modified data
- **Historization**: Automatically maintains history of incremental loads
- **Resilience**: Handles both first-time loads and subsequent incremental loads
- **Reusability**: Works with any table by providing a consistent interface
- **Audit Trail**: Creates timestamped partial load files for troubleshooting or rollback

## Technical Details

1. The `vmaxUpdatedAt - Interval(0.0005)` formula creates a tiny buffer (about 0.7 seconds) to prevent edge cases with records having the exact timestamp

2. The framework expects:
   - A standard SQL table with a timestamp field for tracking updates
   - A unique identifier field for proper patching
   - Adequate permissions to read from the SQL source and write to the QVD folder

## File Structure

The script will create and maintain the following file structure:

```
vFolder/
├── maxUpdatedAt_vTable.qvd    # Stores the latest timestamp
├── vTable.qvd                 # Main data file
└── partial_loads/             # History of incremental loads
    ├── vTable_23.05.20_14.30.25.qvd
    ├── vTable_23.05.21_14.15.10.qvd
    └── ...
```

## Limitations

- Requires a reliable timestamp field that is always updated when records change
- Assumes data is not physically deleted from the source system (logical deletes preferred)
- Does not handle schema changes automatically

## Contributions

Contributions are welcome. Please feel free to fork, improve the code, and submit a pull request.

