---
name: "sql-server-postgre-with-complex-params"
description: "convert complex query sql server"
---
 
 
 
# SQL Server to PostgreSQL Query Converter
 
When the user provides a SQL Server query or asks for conversion help, execute this workflow:
 
## Step 1: Analyze the SQL Server Query
- Identify all parameters (TVP - Table-Valued Parameters, scalar parameters)
- List all SQL Server specific syntax:
  - `CAST(... AS BIT)` → boolean
  - `DATEDIFF(DAY, date1, date2)` → `(DATE(date2) - DATE(date1))::INTEGER`
  - String concatenation: `'text' + @param + 'text'` → `'text' || @param || 'text'`
  - `NOT EXISTS (SELECT 1 FROM @param)` → `CARDINALITY(@param) = 0`
  - `IN (SELECT [Value] FROM @param)` → `= ANY(@param)`
  - TVP handling logic with `NOT EXISTS` → Convert to `CARDINALITY`
## Step 2: Extract Parameters
Create a WITH "P" AS (...) CTE block with:
- All parameters cast to proper PostgreSQL types
- Array parameters as `::text[]`
- Scalar parameters with their types
- Default values or `@paramName::type` placeholder format
Format:
```sql
WITH "P" AS (
    SELECT
        @DateNow::timestamp AS "DateNow",
        @MainDealerAhmCodes::text[] AS "MainDealerAhmCodes",
        NULL::timestamp AS "DateFrom",
        ...
)
```
 
## Step 3: Convert Query Structure
 
### Boolean Logic
- `CAST(x AS BIT)` → `(condition)::boolean` or `1` → `TRUE`, `0` → `FALSE`
- Boolean comparisons: `= 0` → `= FALSE`, `= 1` → `= TRUE`
### Date Functions
- `DATEDIFF(DAY, startDate, endDate)` → `(DATE(endDate) - DATE(startDate))::INTEGER`
- `GETDATE()` → `now()`
### String Operations
- Concatenation: `'text' + @param + 'text'` → `'text' || @param || 'text'`
- Case-insensitive: wrap columns in double quotes `"ColumnName"`
### Parameter Handling
- TVP logic:
  ```sql
  -- SQL Server
  (NOT EXISTS (SELECT 1 FROM @param) OR column IN (SELECT [Value] FROM @param))
  
  -- PostgreSQL
  (CARDINALITY(@param) = 0 OR column = ANY(@param))
  ```
 
- Empty array handling: `CARDINALITY(@param) = 0` means no filter (return all rows)
- Array comparisons: `IN (...)` → `= ANY(...)`
### Type Casting
- All custom type columns (citext, uuid) need `::text` casting when compared with text parameters
- Example: `"ALA"."AssigneedAhmCode"::text = ANY(@MainDealerAhmCodes)`
## Step 4: Identifier Quoting
- All identifiers in double quotes: `"TableName"."ColumnName"`
- Prevents case-sensitivity issues
## Step 5: Output Format
Provide:
1. **Converted PostgreSQL Query** - with CTE parameter block at top
2. **Parameter JSON Template** - format for monitoring_overdue.json
3. **Key Changes** - list of conversions made
4. **Usage Instructions** - how to use with QueryHandler.js
---
 
## Example Conversion
 
**Input (SQL Server):**
```sql
WHERE
    (NOT EXISTS (SELECT 1 FROM @MainDealerAhmCodes) 
     OR ALA.AssigneedAhmCode IN (SELECT [Value] FROM @MainDealerAhmCodes))
    AND DATEDIFF(DAY, ALA.StartDate, @DateNow) > 0
    AND ALA.IsActive = 1
```
 
**Output (PostgreSQL):**
```sql
WITH "P" AS (
    SELECT
        @DateNow::timestamp AS "DateNow",
        @MainDealerAhmCodes::text[] AS "MainDealerAhmCodes"
)
SELECT ...
WHERE
    (CARDINALITY("P"."MainDealerAhmCodes") = 0 OR "ALA"."AssigneedAhmCode"::text = ANY("P"."MainDealerAhmCodes"))
    AND (DATE("P"."DateNow") - DATE("ALA"."StartDate"))::INTEGER > 0
    AND "ALA"."IsActive" = TRUE
```
 
---
 
## Special Handling
 
### Nested Case When with NULL
Simplify `CASE WHEN ... THEN ... ELSE NULL END` → `COALESCE(...)`
 
### LEFT JOIN with Complex Logic
Maintain all JOINs but ensure:
- All comparison columns cast to same type (usually `::text`)
- Example: `"ALA"."GroupId"::text = "FilterSelection"."GroupId"::text`
### GROUP BY with Aggregates
Include all non-aggregated columns in GROUP BY
 
---
 
## Output Checklist
- ✅ CTE "P" block with all parameters
- ✅ All `@paramName` replaced with `"P"."paramName"` in query body
- ✅ All SQL Server syntax converted to PostgreSQL
- ✅ All identifiers in double quotes
- ✅ CARDINALITY used for empty array checks
- ✅ ANY() used for array membership
- ✅ Proper type casting (::text, ::timestamp, etc.)
- ✅ Parameter JSON template provided