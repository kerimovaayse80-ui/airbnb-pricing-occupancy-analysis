# Seattle Airbnb Pricing and Occupancy Report

## 1. Project Overview

This project analyzes Seattle Airbnb listing prices, calendar-based occupancy, seasonal patterns, room types, instant-bookable status, and neighbourhood-level comparisons using Power BI. The report is designed to help a host or property investor compare listings and identify neighbourhoods for further investigation.

**Dataset source:** Seattle Airbnb listings and calendar data provided in the assignment: https://www.kaggle.com/datasets/airbnb/seattle

## 2. Source Tables

### `dim_listing` — listing dimension
Contains listing-level information, including listing ID, cleaned neighbourhood, room type, listing price, instant-bookable status, number of reviews, latitude, and longitude.

### `fact_calendar` — calendar fact table
Contains daily listing records with listing ID, date, availability, calendar price, and the `is_booked` flag created in Power Query. The calendar dataset contains approximately 1.39 million listing-date records.

### `date_table` — date dimension
A calculated calendar table containing `date`, `year`, `month_number`, `month_name`, and `quarter`.

The listings dataset contains approximately 3,818 listings. Exact row counts in visuals can vary according to filters and measures.

## 3. Power Query Cleaning

- Removed `$` symbols and thousands separators (`,`) from listing and calendar price fields, then converted values to numeric decimal types.
- Removed the original `neighbourhood` column because it contained missing values; retained `neighbourhood_cleansed` for neighbourhood analysis.
- Created `is_booked` from the calendar `available` field:
  - `available = "f"` → `is_booked = 1` (booked)
  - `available = "t"` → `is_booked = 0` (available)
- Kept blank calendar prices blank rather than inventing historical prices.
- Set appropriate types for retained fields, including prices, dates, IDs, review counts, and categorical listing attributes.

## 4. Data Model

The report uses a star-schema-style model with listing and date dimensions connected to the calendar fact table.

| One-side table and column | Many-side table and column | Purpose |
|---|---|---|
| `dim_listing[id]` | `fact_calendar[listing_id]` | Connects calendar records to listing attributes such as neighbourhood, room type, price band, and instant-bookable status. |
| `date_table[date]` | `fact_calendar[date]` | Supports monthly and seasonal analysis using a proper date dimension. |

Both relationships are active and use single-direction filtering. The date table was marked as the official date table, and `month_name` was sorted by `month_number` for chronological ordering.

### Date table DAX

```DAX
date_table =
CALENDAR(
    MINX(ALL(fact_calendar), fact_calendar[date]),
    MAXX(ALL(fact_calendar), fact_calendar[date])
)
```

The date table was extended with year, month number, month name, and quarter fields.

## 5. DAX Measures and Calculated Columns

The formulas below document the DAX used in the report and why each item was created.

### 5.1 `active_listings`

```DAX
active_listings =
DISTINCTCOUNT(fact_calendar[listing_id])
```

**Purpose:** Counts unique listing IDs represented in the current filter context.

### 5.2 `booked_nights`

```DAX
booked_nights =
SUM(fact_calendar[is_booked])
```

**Purpose:** Sums the booked flag across selected listing-date records. Each booked record contributes 1.

### 5.3 `available_nights`

```DAX
available_nights =
COUNTROWS(fact_calendar)
```

**Purpose:** Counts all calendar rows in the current context, including both booked and available records. It is the denominator used by the occupancy measure, despite the measure's name.

### 5.4 `occupancy_rate`

```DAX
occupancy_rate =
DIVIDE([booked_nights], [available_nights])
```

**Purpose:** Calculates the proportion of calendar listing-date records classified as booked. `DIVIDE` handles a zero denominator safely.

**Interpretation:** The report currently shows approximately **32.94%** overall occupancy. The assignment reference of around 67% may represent availability share or use a different definition. This report preserves the source mapping `available = "f"` → booked and `"t"` → available rather than inverting the flag to match the reference.

### 5.5 `avg_price`

```DAX
avg_price =
AVERAGE(dim_listing[price])
```

**Purpose:** Calculates average listing price in the current filter context.

### 5.6 `median_price`

```DAX
median_price =
MEDIANX(
    VALUES(dim_listing[id]),
    CALCULATE(AVERAGE(dim_listing[price]))
)
```

**Purpose:** Calculates a price per distinct listing and then takes the median, so each listing contributes once to the listing-level median.

**Why use `MEDIANX` over distinct listing IDs?** The calendar fact table has repeated daily rows per listing. A plain median over the fact table can weight listings repeatedly and use only nonblank calendar prices, answering a different question from the median listing price. Iterating over distinct listing IDs makes each listing the unit of analysis.

### 5.7 `revenue_potential`

```DAX
revenue_potential =
SUMX(
    fact_calendar,
    fact_calendar[price] * fact_calendar[is_booked]
)
```

**Purpose:** Attempts to sum calendar prices for booked rows.

**Limitation:** Many booked calendar rows have blank prices, so this measure returns zero or substantially undercounts. It is not reliable actual revenue.

### 5.8 `estimated_revenue_potential`

```DAX
estimated_revenue_potential =
SUMX(
    fact_calendar,
    IF(
        fact_calendar[is_booked] = 1,
        COALESCE(
            fact_calendar[price],
            RELATED(dim_listing[price])
        ),
        0
    )
)
```

**Purpose:** Uses the calendar price when present and falls back to the related listing-level price for booked rows with a blank calendar price. Available rows contribute zero.

The report shows approximately **$59.42 million**.

**Limitation:** This is an estimate, not verified revenue. It assumes listing-level price approximates the price on a booked date and excludes discounts, fees, taxes, and booking-specific rates.

### 5.9 `price_band` calculated column

```DAX
price_band =
SWITCH(
    TRUE(),
    ISBLANK(dim_listing[price]), "Unknown",
    dim_listing[price] < 50, "Under $50",
    dim_listing[price] < 100, "$50–$99",
    dim_listing[price] < 150, "$100–$149",
    dim_listing[price] < 250, "$150–$249",
    "$250+"
)
```

**Purpose:** Groups listing prices into ranges for the price-band slicer. `SWITCH(TRUE())` checks conditions in order and returns the first matching category.

### 5.10 `calendar_median_price`

```DAX
calendar_median_price =
MEDIANX(
    FILTER(
        fact_calendar,
        NOT ISBLANK(fact_calendar[price])
    ),
    fact_calendar[price]
)
```

**Purpose:** Excludes blank calendar prices before calculating the median in the current date/month context. Used in the monthly median calendar price chart.

### 5.11 `listing_occupancy_rate`

```DAX
listing_occupancy_rate =
DIVIDE(
    SUM(fact_calendar[is_booked]),
    COUNTROWS(fact_calendar)
)
```

**Purpose:** Calculates the booked share in the current visual context. When the scatter plot is grouped by listing, it provides a listing-level occupancy estimate for comparison with review count.

### 5.12 `overall_median_occupancy`

```DAX
overall_median_occupancy =
CALCULATE(
    MEDIANX(
        VALUES(dim_listing[neighbourhood_cleansed]),
        [occupancy_rate]
    ),
    ALL(dim_listing[neighbourhood_cleansed])
)
```

**Purpose:** Calculates the median of neighbourhood occupancy rates after removing the current neighbourhood filter. Used as a benchmark for conditional formatting.

### 5.13 `occupancy_color`

```DAX
occupancy_color =
IF(
    [occupancy_rate] > [overall_median_occupancy],
    "#D9EAD3",
    "#FCE8E6"
)
```

**Purpose:** Returns light green for occupancy above the benchmark and light red otherwise. Used for conditional formatting in the neighbourhood table.

### 5.14 `neighbourhood_median_price_benchmark`

```DAX
neighbourhood_median_price_benchmark =
MEDIANX(
    ALLSELECTED(dim_listing[neighbourhood_cleansed]),
    CALCULATE([median_price])
)
```

**Purpose:** Calculates the median of neighbourhood-level median prices among neighbourhoods in the current selection.

### 5.15 `neighbourhood_median_occupancy_benchmark`

```DAX
neighbourhood_median_occupancy_benchmark =
MEDIANX(
    ALLSELECTED(dim_listing[neighbourhood_cleansed]),
    CALCULATE([occupancy_rate])
)
```

**Purpose:** Calculates the median of neighbourhood-level occupancy rates among neighbourhoods in the current selection.

### 5.16 `investment_target`

```DAX
investment_target =
IF(
    [median_price] > [neighbourhood_median_price_benchmark]
        && [occupancy_rate] > [neighbourhood_median_occupancy_benchmark],
    1,
    0
)
```

**Purpose:** Flags neighbourhoods whose median price and occupancy rate are both above their respective neighbourhood medians. The dashboard includes a separate investment-target table with the visual-level filter `investment_target is 1`. This filter intentionally limits that table to neighbourhoods meeting both benchmark conditions; it does not apply to the general neighbourhood comparison table or the Top 5/Bottom 5 price visuals.

**Interpretation:** This is a screening rule, not a profit calculation. High prices and occupancy do not guarantee returns because operating costs, acquisition costs, taxes, and regulations are not included. The filter helps users focus on neighbourhoods that score above the selected benchmarks on both measures, rather than showing every neighbourhood in that particular table.

### 5.17 `price_color_group` calculated column

```DAX
price_color_group =
SWITCH(
    TRUE(),
    ISBLANK(dim_listing[price]), "Unknown",
    dim_listing[price] < 75, "Low price",
    dim_listing[price] < 150, "Medium price",
    "High price"
)
```

**Purpose:** Classifies individual listings into Low, Medium, and High price categories so the map can display category colors. These are listing-price bands, not neighbourhood median-price coloring.

## 6. Report Pages

### 6.1 Neighbourhood Benchmark

Includes:
- KPI cards: active listings, median price, occupancy rate, estimated revenue potential.
- Median price by neighbourhood.
- Occupancy rate by neighbourhood.
- Top 5 and Bottom 5 neighbourhoods by median price.
- Comparison table: neighbourhood, listing count, median price, occupancy rate.
- Room-type and price-band slicers.
- Conditional formatting for occupancy.
- A separate investment-target table filtered by `investment_target is 1`, showing only neighbourhoods whose median price and occupancy are both above their respective neighbourhood-median benchmarks. This is a visual-level filter for that table only; the general comparison and Top 5/Bottom 5 visuals are intended to remain unfiltered by `investment_target`.
- Map titled **Airbnb Listings: Location and Price**, using latitude and longitude to plot Seattle listings and `price_color_group` for price categories.

The current report screenshot shows approximately 4K active listings, a $100 overall median listing price, 32.94% occupancy, and $59.42M estimated revenue potential. The Top 5 visual uses a Top 5 filter by `median_price`; the Bottom 5 visual uses a Bottom 5 filter by `median_price` and is sorted from lower to higher price.

**Map note:** The map plots individual listing coordinates. Its legend shows listing-price categories (`Low price`, `Medium price`, `High price`), not a continuous colour scale for neighbourhood median price. Individual points may overlap. It should not be described as a neighbourhood-aggregated bubble map unless the data is actually grouped by neighbourhood.

### 6.2 Seasonality

Includes two line charts:
- Monthly median calendar price.
- Monthly occupancy rate.

In the current view, median calendar prices rise around spring and early summer, peak in July, and generally ease later in the year. Occupancy is highest in January, declines into March, rises around July, and then trends down toward December. These are descriptive patterns and do not prove causation.

### 6.3 Listing Characteristics

Includes:
- Median price by room type.
- Occupancy rate by room type.
- Median price for instant-bookable versus non-instant-bookable listings.
- Occupancy for instant-bookable versus non-instant-bookable listings.
- Scatter plot of number of reviews versus listing occupancy rate.

Values visible in the report screenshots:
- Entire home/apt: median price about **$126**, occupancy about **36.04%**.
- Private room: median price about **$69**, occupancy about **27.59%**.
- Shared room: median price about **$40**, occupancy about **18.60%**.
- Instant-bookable and non-instant-bookable listings have similar displayed median prices (about **$99–$100**) and occupancy (about **32.6–33.0%**).

These are descriptive comparisons and do not establish causation.

### 6.4 Neighbourhood Listing Details (Drill-through)

This page uses `dim_listing[neighbourhood_cleansed]` as the drill-through field with **Keep all filters** enabled. It displays active listing count, median price, occupancy rate, and a listing-level table with room type, price, instant-bookable status, and number of reviews.

The supplied screenshot shows a selected neighbourhood with **234 active listings**, a **$139 median price**, and a **37.57% occupancy rate**. These figures apply to the selected neighbourhood only.

## 7. Five Preliminary Investor Insights

1. **Neighbourhood prices vary.** Median listing prices differ across neighbourhoods. Higher prices may suggest revenue potential, but should be considered alongside occupancy, operating costs, and competition. *Visual: Median Price by Neighbourhood and the comparison table.*
2. **Occupancy differs by neighbourhood.** Some neighbourhoods show higher occupancy than the overall report value and may merit further investigation. Occupancy alone does not determine profitability. *Visual: Occupancy Rate by Neighbourhood.*
3. **Prices show a seasonal pattern.** In the current view, median calendar prices peak around July and are lower in several later months. Hosts may consider seasonal patterns when planning prices and cash flow. *Visual: Monthly Median Listing Price.*
4. **Room type is associated with different price and occupancy levels.** Entire homes/apartments show higher median prices and occupancy than private and shared rooms in the current comparison, but may also have higher operating costs. *Visuals: Median Price by Room Type and Occupancy Rate by Room Type.*
5. **Instant booking shows little difference in this comparison.** The displayed median prices and occupancy rates are similar for instant-bookable and non-instant-bookable listings. This does not prove instant booking has no effect. *Visuals: Instant Bookable vs Non-Instant Price and Occupancy charts.*

## 8. Optional Bonus Features

The report prioritizes the assignment’s core requirements: a star-schema-style model with a marked Date table, filter-responsive occupancy calculations, chronological month sorting, neighbourhood drill-through, report visuals, and documented investor insights. Optional bonus features are not required for the core submission.

- **Map visual:** Added to the neighbourhood benchmark page. It uses latitude and longitude and a `price_color_group` legend. The current map colors individual listings by price category; it does not use a continuous colour scale for neighbourhood median price.
- **Conditional formatting:** Implemented in the neighbourhood comparison table using `occupancy_color` to highlight occupancy relative to the median benchmark.
- Other optional enhancements, such as a what-if price parameter, a dynamic TOPN selector, a bookmark-driven price/occupancy toggle, and Power BI Service publishing with scheduled refresh, are outside the current scope. The report does not claim these enhancements are implemented.

## 9. Limitations and Assumptions

- Overall occupancy is approximately **32.94%** under the current `is_booked` definition. The assignment reference of around 67% may represent availability share or use a different definition. The report retains the source mapping rather than inverting the flag to match the reference.
- `available_nights` counts all calendar rows, including booked and available records, despite its name.
- Calendar prices are blank for many booked dates, so actual booked revenue cannot be calculated reliably from calendar prices alone.
- Estimated revenue potential uses listing-level price as a proxy for missing booked-date prices and must not be described as actual revenue.
- The report is descriptive and does not establish causation or calculate full investment returns.
- A complete investment analysis would require operating expenses, cleaning fees, taxes, platform fees, acquisition costs, local regulations, and verified booking revenue.
- The map displays individual listing points and price bands, not an aggregated neighbourhood bubble map with a continuous median-price colour scale.
- Top 5 and Bottom 5 results should be interpreted using the actual median-price filters and sort order.

## 10. Submission Contents

The assignment deliverables are:
- `airbnb_report.pbix`
- PNG exports of all report pages, including the drill-through page and map page if it remains separate
- PNG export of the model view
- This `note.md`, documenting data cleaning, model decisions, DAX formulas and rationale, occupancy definition, median-price methodology, five investor insights, and bonus-feature status
