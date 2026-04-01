# 💄 D2C Beauty & Personal Care Analytics: An Executive Command Center (Power BI + Power Query)

## 📌 Project Overview

In the fast-paced Direct-to-Consumer (D2C) Beauty and Personal Care sector, data fragmentation is a silent profit killer. This project was engineered to solve a critical business bottleneck for a D2C brand's C-Suite: **a 5-to-6-day lag in manual Excel reporting** across disjointed sales channels, complex vendor manufacturing contracts, and siloed marketing agency data.

This solution completely replaces static PowerPoint decks with a dynamic, end-to-end Power BI application. It seamlessly consolidates ERP, Marketplace, and Ad Platform data into a single source of truth, empowering the CEO, CFO, and CMO to make real-time, data-driven decisions for their next Investor Committee meeting.

-----

## 🛠️ Technical Architecture & Tools

  * **BI Platform:** Power BI Desktop & Power BI Service
  * **Data Engineering (ETL):** Power Query (Advanced M-Code), dynamic folder consolidation, schema standardization.
  * **Data Modeling:** Star Schema architecture, handling mixed granularities, disconnected parameter tables, and inactive relationships.
  * **Calculations (DAX):** Slowly Changing Dimensions (SCD Type 2), Virtual Relationships (`TREATAS`), Iterative Functions (`SUMX`), and What-If Parameter generation.
  * **UI/UX Design:** Dark mode glassmorphism, contextual navigation, optimized data-ink ratio.

-----

## ⚙️ Data Engineering & Power Query (M-Code)

The foundation of this model required extracting, transforming, and loading (ETL) messy, fragmented data from multiple sources without requiring manual code changes when new monthly data drops.

### 1\. Dynamic Folder Consolidation (Sales Data)

To fulfill the CEO's requirement of "zero manual intervention" for new datasets, I engineered a dynamic folder extraction script. It automatically loops through the internal D2C and Marketplace folders, expands the binaries, and standardizes the schema across Scheduled Deliveries and Quick Commerce formats.

```powerquery
let
    Source = #"Case Data Source",

    // select only SALE files
    #"select Sales files" =
        Table.SelectRows(
            Source,
            each Text.Contains(Text.Upper([Name]), "SALE")
        ),

    // process each file row
    #"Processed Content col" =
        Table.TransformRows(
            #"select Sales files",
            each
                let
                    extension = Text.Upper(_[Extension]),
                    content   = _[Content],

                    result =
                        if extension = ".CSV" then Table.PromoteHeaders(Csv.Document(content))

                        else if extension = ".XLSX" then
                            let
                                wb = Excel.Workbook(content),
                                onlySheets = Table.SelectRows(wb, each [Kind] = "Sheet"),
                                expandedSheets =
                                    List.Transform(
                                        onlySheets[Data],
                                        each Table.PromoteHeaders(_)
                                    ),
                                combined = Table.Combine(expandedSheets)
                            in
                                combined

                        else
                            null // for unexpected file types
                in
                    result
        ),
    
    // total sales data
    #"total raw data" = Table.Combine(#"Processed Content col"),
    
    // remove unnecessary columns, will will fetch them later when will need
    #"Removed Columns" = Table.RemoveColumns(#"total raw data",{"Month", "Year", "Week", "Delivery_Days"}),

    // trim, clean, capitalize
    #"Final Data" = 
        Table.TransformColumns(
            #"Removed Columns",
            List.Transform(
                Table.ColumnNames(#"Removed Columns"),
                each {_, each Text.Proper(Text.Trim(Text.Clean(Text.From(_))))}
            )
        ),
    #"Changed Type" = Table.TransformColumnTypes(#"Final Data",{{"Sales_Date", type datetime}, {"Delivery_Date", type datetime}, {"SKU_Code", type text}, {"Qty", type number}, {"Unit_Price", type number}, {"Line_Amount", type number}, {"From_Location", type text}, {"To_Location", type text}, {"Channel", type text}, {"Sub_Channel", type text}, {"Delivery_Type", type text}, {"Courier", type text}, {"City_Tier", type text}, {"Category", type text}, {"Order_ID", type text}, {"Customer_ID", type text}, {"Payment_Method", type text}, {"Is_Returned", Int64.Type}, {"Rider_Distance_Km", type any}})
in
    #"Changed Type"
```

### 2\. Marketing Budget Granularity Transformation

Marketing budgets were provided at a Category/Month level, which conflicted with daily transaction data. Power Query was utilized to clean and structure the budget data to integrate flawlessly with the shared `DimDate` table.

```powerquery
let
    Source = #"Case Data Source",
    #"Filtered Rows" = Table.SelectRows(Source, each ([Name] = "Performance Marketing Ad Spends Budget.xlsx")),
    #"uncleared data" = Table.Combine(Table.TransformColumns(#"Filtered Rows", {{"Content", each Table.Combine(Excel.Workbook(_)[Data])}})[Content]),
    #"removed junk rows" = 
        let
            innersource = #"uncleared data",
            // is blank rows function
            isblankrow = 
                (r as record) => List.ContainsAny(Record.FieldValues(r), {"", " ", null}),
            // first junk skip
            firstskip = Table.Skip(innersource, each isblankrow(_)),
            // reverse the table
            reverserow = Table.ReverseRows(firstskip),
            // last junk skip
            lastskip = Table.Skip(reverserow, each isblankrow(_)),
            // reverse the table to get original table structure
            final = Table.ReverseRows(lastskip)
        in
            final,
    // promoting top row as header and removed grand total column as we can calculate the same in dax if needed
    #"header promote" = Table.PromoteHeaders(#"removed junk rows", [PromoteAllScalars=true]),
    #"remove grand total col & row" = Table.FirstN(Table.RemoveColumns(#"header promote", "Grand Total"), each _[Category] <> "Grand Total"),
    #"Unpivoted Other Columns" = Table.UnpivotOtherColumns(#"remove grand total col & row", {"Category"}, "Date", "Amount"),
    #"cleaned data" = Table.TransformColumns(
            #"Unpivoted Other Columns",
            List.Transform(
                Table.ColumnNames(#"Unpivoted Other Columns"),
                each {_, each Text.Proper(Text.Trim(Text.Clean(Text.From(_))))}
            )
        ),
    #"Changed Type" = Table.TransformColumnTypes(#"cleaned data",{{"Category", type text}, {"Date", type date}, {"Amount", type number}})
in
    #"Changed Type"
```

-----

## 🧠 Advanced DAX & Business Logic Engine

This model goes far beyond basic aggregations. I engineered complex DAX solutions to replicate real-world financial and marketing formulas.

### 1\. Handling Slowly Changing Dimensions (SCD Type 2) for Vendor Costs

**The Business Problem:** The brand utilizes a Just-In-Time (JIT) manufacturing model where vendor pricing changes dynamically based on quarterly negotiated contracts. A standard physical relationship in Power BI would cause a Many-to-Many failure.
**The Engineering Solution:** I built an inactive relationship model and utilized `SUMX` and `CALCULATE` to force the DAX engine to evaluate the exact transaction `Sales_Date` against the active `Start_Date` and `End_Date` of the vendor contract.

```dax
_Total Cost = SUMX(FactSales, FactSales[Qty] * [_Unit Cost])
```
```dax
_Unit Cost = 
VAR SKU = SELECTEDVALUE(FactSales[SKU_Code])
-- month will filter by slicers, though cost not fixed but it obvious that cost will remain same for a single month
VAR MonthStart =
    CALCULATE(
        MIN(DimDate[Date]), 
        ALLSELECTED(DimDate)
    )
VAR MonthEnd =
    CALCULATE(
        MAX(DimDate[Date]), 
        ALLSELECTED(DimDate)
    )

RETURN
CALCULATE(
    MAX(Dim_Cost_Structure[Price]),
    FILTER(
        Dim_Cost_Structure,
        Dim_Cost_Structure[SKU_Code] = SKU &&
        Dim_Cost_Structure[Start_Date] <= MonthEnd &&
        Dim_Cost_Structure[End_Date] >= MonthStart
    )
)
```

### 2\. The CFO's Dynamic Negotiation Tool (What-If Parameters)

**The Business Problem:** The CFO negotiates a 10% volume discount with vendors if a product hits a certain monthly unit threshold. However, this threshold is actively being renegotiated.
**The Engineering Solution:** I deployed a numeric What-If parameter (`Discount Threshold`). The CFO can drag a slider on the dashboard, and the DAX engine instantly recalculates the entire P\&L based on the new hypothetical threshold.

```dax
_Volume Discount = 
SUMX(
    FactSales,
    FactSales[Qty]
        * [_Unit Cost]
        * 0.10
        * [_2 Is Discount Applicable]
)
```
```dax
_2 Is Discount Applicable = 
VAR dis = SELECTEDVALUE('Discount Threshold'[Discount Threshold])
VAR monthlyQty =
    CALCULATE(
        SUM(FactSales[Qty]),
        ALLEXCEPT(FactSales, FactSales[SKU_Code], DimDate[Year], DimDate[Month])
    )
RETURN
IF(monthlyQty >= dis, 1, 0)
```
```dax
Discount Threshold = GENERATESERIES(100, 300, 25)
```

### 3\. Virtual Relationships for Ad Attribution (`TREATAS`)

**The Business Problem:** The CMO needs to slice Ad Spend by Product Category, but the Marketing agency data has no physical relationship to the internal Product Dimension table.
**The Engineering Solution:** Rather than building clunky, bidirectional many-to-many bridge tables, I utilized `TREATAS` to virtually propagate the Category filter from the Product table directly into the Ad Campaign Fact table.

```dax
_Category Ad Spend = 
CALCULATE(
    SUM(Dim_Ad_Campaign_Data[Spend]),
    TREATAS(VALUES(Dim_Ad_Campaign_Data[YearMonth]), DimDate[YearMonth]),
    TREATAS(VALUES(Dim_Ad_Campaign_Data[Category]), FactSales[Category]),
    TREATAS(VALUES(Dim_Ad_Campaign_Data[Medium]), Dim_Ad_Campaign_Data[Medium])
)
```

### 4\. Dynamic Customer Acquisition Cost (CAC)

Ensuring accurate marketing efficiency metrics while handling potential divide-by-zero errors in months with zero sales.

```dax
_CAC = DIVIDE([_Total Ad Spend], [_Total Orders]) // Customer Acquisition Cost
```

-----

## 🚀 Dashboard Deep Dives & Impact

### 🌐 1. The Executive Command Center

The landing page designed for the entire C-Suite to get an immediate, 30-second pulse on the organizational health.

  * **Business Impact:** Merges siloed sales and marketing data into a single view. Shows instantly how Revenue, Margins, and Return on Ad Spend (ROAS) are trending Month-over-Month.
  * **Key Visuals:** High-contrast KPI scorecards, Revenue by Sub-Channel (Website vs. Quick Commerce), and Category-level sales distribution, Pareto Analysis.
  * <img width="1366" height="768" alt="EXECUTIVE" src="https://github.com/user-attachments/assets/bb54427d-b594-499b-84dc-2303306f597a" />


### 📈 2. The CEO View (Growth & Strategy)

Designed explicitly to replace static PowerPoint slides during Investor Committee pitches.

  * **Business Impact:** Answers the ultimate investor question: *"Where is the profitable growth coming from?"*
  * **Key Visuals:** A dynamic **Category Growth Matrix** (Scatter Plot) that visually isolates "Star" categories (high volume + high margin), and a dual-axis trendline proving that revenue is scaling efficiently alongside ad spend.
  * <img width="1366" height="768" alt="CEO" src="https://github.com/user-attachments/assets/7f82328c-f2f0-4d26-89b2-fe76daa39263" />


### 💰 3. The CFO View (Vendor Margins & Settlements)

A highly operational financial tool engineered to speed up monthly vendor payments and track contract profitability.

  * **Business Impact:** Transforms a multi-day manual Excel reconciliation process into an instant, automated view. Empowers the CFO to negotiate better volume discounts using the interactive threshold slider.
  * **Key Visuals:** A financial Waterfall Chart bridging Base COGS to Net Payable, and a conditionally formatted Vendor Settlement Matrix detailing exact savings per SKU.
  * <img width="1366" height="768" alt="CEO" src="https://github.com/user-attachments/assets/f31647a9-4a42-41ce-86f2-43848fb418f3" />


### 🎯 4. The CMO View (Attribution & Funnel)

Built to defend the performance marketing budget and evaluate agency efficiency.

  * **Business Impact:** Provides granular visibility into the cost of acquiring a customer (CAC) across Google (Search, CTV, YouTube) and Meta platforms.
  * **Key Visuals:** A multi-stage Customer Acquisition Funnel tracking percentage drop-offs, an Ad Efficiency Matrix highlighting the most profitable channels, and a Campaign Scatter Plot to identify winning creatives.
  * <img width="1366" height="768" alt="CMO" src="https://github.com/user-attachments/assets/5348dac6-ed63-4c71-bd76-14a8bf19d2ee" />


-----

## 👨‍💻 About the Author

**Suman Dass**
*Data Assistant Manager | Kolkata, India*

I specialize in bridging the gap between raw data architecture and executive decision-making. I build end-to-end business intelligence solutions that don't just display numbers, but tell a compelling, actionable business story.

  * 🌐 **Portfolio:** [Portfolio](https://sumanndass.github.io/)
