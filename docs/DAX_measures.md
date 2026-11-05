# DAX measures and calculated objects

Measures are stored in a table called `Measures`. Currency is not specified in the dataset, so values are shown as plain numbers.

## Calendar table

```dax
Calendar =
ADDCOLUMNS(
    CALENDAR(DATE(2023,1,1), DATE(2023,12,31)),
    "Month", FORMAT([Date], "MMM"),
    "MonthNum", MONTH([Date]),
    "Weekday", FORMAT([Date], "ddd"),
    "FY Quarter", "FQ" & (INT(MOD(MONTH([Date]) - 4, 12) / 3) + 1)
)
```
`Month` is sorted by `MonthNum`. Financial year starts in April, so April is FQ1.

## Core measures

```dax
Total Sales = SUM(Orders[Order Value])
Total Orders = COUNTROWS(Orders)
AOV = DIVIDE([Total Sales], [Total Orders])
Total Target = SUM('Sales Targets'[2023 Sales Target])
Achievement % = DIVIDE([Total Sales], [Total Target])
Customers Ordered = DISTINCTCOUNT(Orders[Customer ID])
Not Captured Orders = CALCULATE(COUNTROWS(Orders), Orders[Order Source] = "Not Captured")
```

## Sales target status (Sales Targets table)

```dax
Actual Sales = CALCULATE(SUM(Orders[Order Value]))

Target Status =
IF('Sales Targets'[Actual Sales] < 'Sales Targets'[2023 Sales Target], "Target not met",
   IF('Sales Targets'[Actual Sales] = 'Sales Targets'[2023 Sales Target], "Target Met", "Exceeded Target"))
```

A disconnected helper table keeps all three buckets visible, even though "Target Met" has zero POCs:

```dax
Status Bucket =
DATATABLE("Status", STRING, "Sort", INTEGER,
    {{"Target not met", 1}, {"Target Met", 2}, {"Exceeded Target", 3}})

POC Count =
VAR s = SELECTEDVALUE('Status Bucket'[Status])
RETURN COUNTROWS(FILTER('Sales Targets', 'Sales Targets'[Target Status] = s)) + 0

Avg Shortfall =
AVERAGEX(
    FILTER('Sales Targets', 'Sales Targets'[Target Status] = "Target not met"),
    'Sales Targets'[2023 Sales Target] - 'Sales Targets'[Actual Sales]
)
```

## Top manager per country

`Top N` filters rank across the whole table, not per country, so a rank measure is used and filtered to 1 in each table.

```dax
Rank Sales = RANKX(ALL('Sales Targets'[Sales Manager]), [Total Sales], , DESC, Dense)
Rank Orders = RANKX(ALL('Sales Targets'[Sales Manager]), [Total Orders], , DESC, Dense)
Rank Customers = RANKX(ALL('Sales Targets'[Sales Manager]), [Customers Ordered], , DESC, Dense)
```

## Monthly trend

```dax
Best Month (Sales) =
VAR t = ADDCOLUMNS(ALL('Calendar'[Month]), "@S", CALCULATE([Total Sales]))
VAR mx = MAXX(t, [@S])
RETURN MAXX(FILTER(t, [@S] = mx), 'Calendar'[Month])

Best Month (Orders) =
VAR t = ADDCOLUMNS(ALL('Calendar'[Month]), "@O", CALCULATE([Total Orders]))
VAR mx = MAXX(t, [@O])
RETURN MAXX(FILTER(t, [@O] = mx), 'Calendar'[Month])

Weakest Month (Sales) =
VAR t = ADDCOLUMNS(ALL('Calendar'[Month]), "@S", CALCULATE([Total Sales]))
VAR mn = MINX(t, [@S])
RETURN MAXX(FILTER(t, [@S] = mn), 'Calendar'[Month])
```

## Customers

```dax
Customers No Order = COUNTROWS(FILTER(Customers, ISBLANK(CALCULATE(COUNTROWS(Orders)))))
% Customers No Order = DIVIDE([Customers No Order], COUNTROWS(Customers))
Indian Male Customers = CALCULATE(COUNTROWS(Customers), Customers[Customer Country] = "India", Customers[Gender] = "M") + 0
Avg Age = AVERAGE(Customers[Age])
Sales by Category = [Total Sales] + 0
Delta POCs = CALCULATE(DISTINCTCOUNT(Orders[Sales POC]), 'Sales Targets'[Sales Team] = "Delta") + 0
```

## Order sources

Returns every tied source when there is a tie:

```dax
Top Source =
VAR t = ADDCOLUMNS(VALUES(Orders[Order Source]), "@n", [Total Orders])
VAR mx = MAXX(t, [@n])
RETURN CONCATENATEX(FILTER(t, [@n] = mx), Orders[Order Source], " / ")
```

## Optional: what-if target scenario

```dax
Scenario Achievement % =
DIVIDE([Total Sales], [Total Target] * (1 + [Target Change Value] / 100))
```
`Target Change` is a what-if parameter from -20 to 20 in steps of 5.
