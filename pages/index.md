<h1 align="center" style="font-size: 2.25rem; font-weight: 700;">
  Creator Analytics Dashboard by Data 123
</h1>



<Details title='Overview'>

  This dashboard provides a high-level view of creator content performance, engagement efficiency, and quality trends across content types from 2019–2021. The data is sourced from data123.online. 
</Details>

```sql social_media_data
SELECT
    CAST(date AS DATE) AS date,
    content_id,
    content_type,
    earnings,
    shares,
    likes,
    rating
FROM creator_data.creator_analytics
```


```sql kpis
SELECT
    SUM(earnings) AS total_earnings,
    SUM(shares) AS total_shares,
    SUM(likes) AS total_likes,
    ROUND(AVG(rating), 1) AS avg_rating
FROM ${social_media_data}
```

```sql categories
  select
      content_type
  FROM creator_data.creator_analytics
  group by content_type
```



```sql monthly_earnings_by_type
  select 
        date_trunc('month', date) AS month,
        round(SUM(earnings), 1) as earnings,
	content_type    
  from creator_data.creator_analytics
  where content_type like '${inputs.content_type.value}'
  and date_part('year', date) like '${inputs.year.value}'
  group by all
  order by earnings desc
```
 
```sql earnings_contribution
  SELECT
    content_type,
    SUM(earnings) AS earnings
FROM creator_data.creator_analytics
GROUP BY content_type
```

```sql donut_data
  SELECT
    content_type AS name,
    earnings AS value
FROM ${earnings_contribution}
```
```sql Earnings_per_Engagement
  SELECT
    content_type,
    SUM(earnings) AS total_earnings,
    SUM(shares + likes) AS total_engagement,
    ROUND(
        SUM(earnings) * 1.0 / NULLIF(SUM(shares + likes), 0),
        3
    ) AS earnings_per_engagement
FROM creator_data.creator_analytics
GROUP BY content_type
ORDER BY earnings_per_engagement DESC

```

```sql avg_rating_trend_monthly
  WITH monthly AS (
    SELECT
        date_trunc('month', CAST(date AS DATE)) AS month,
        AVG(rating) AS monthly_avg_rating
    FROM creator_data.creator_analytics
    GROUP BY month
)
  SELECT
    month,
    ROUND(monthly_avg_rating, 2) AS monthly_avg_rating,
    ROUND(
        AVG(monthly_avg_rating) OVER (
            ORDER BY month
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ),
        2
    ) AS rolling_3mo_avg_rating
FROM monthly
ORDER BY month

```

```sql top_bottom_content
  WITH ranked AS (
  SELECT
    content_id,
    content_type,
    earnings,
    rating,
    -- Composite: prioritize high earnings AND high rating
    ROUND(earnings * rating, 2) AS earnings_rating_product
  FROM creator_data.creator_analytics
),
top_bottom AS (
  (SELECT *, 'Top Performer' AS category
   FROM ranked
   ORDER BY earnings_rating_product DESC
   LIMIT 5)
  UNION ALL
  (SELECT *, 'Needs Attention' AS category
   FROM ranked
   ORDER BY earnings_rating_product ASC
   LIMIT 5)
)
SELECT * FROM top_bottom
ORDER BY category DESC, earnings_rating_product DESC
```
<Grid cols=4 gap=6>
  <!-- Total Earnings -->
  <div style="
      border: 1px solid rgba(150,150,150,0.5);
      background-color: #E67E22;  /* Burnt Orange */
      padding: 1rem;
      box-shadow: 0 2px 6px rgba(0,0,0,0.05);
  ">
    <BigValue data={kpis} value=total_earnings />
  </div>

  <!-- Total Shares -->
  <div style="
      border: 1px solid rgba(150,150,150,0.5);
      background-color: #F1C40F;  /* Sunny Yellow */
      padding: 1rem;
      box-shadow: 0 2px 6px rgba(0,0,0,0.05);
  ">
    <BigValue data={kpis} value=total_shares />
  </div>

  <!-- Total Likes -->
  <div style="
      border: 1px solid rgba(150,150,150,0.5);
      background-color: #7DCE82;  /* Sage Green */
      padding: 1rem;
      box-shadow: 0 2px 6px rgba(0,0,0,0.05);
  ">
    <BigValue data={kpis} value=total_likes />
  </div>

  <!-- Average Rating -->
  <div style="
      border: 1px solid rgba(150,150,150,0.5);
      background-color: #B0B0B0;  /* Earthy Brown */
      padding: 1rem;
      box-shadow: 0 2px 6px rgba(0,0,0,0.05);
  ">
    <BigValue data={kpis} value=avg_rating />
  </div>
</Grid>



<Dropdown data={categories} name=content_type value=content_type>
    <DropdownOption value="%" valueLabel="All Content"/>
</Dropdown>

<Dropdown name=year>
    <DropdownOption value=% valueLabel="All Years"/>
    <DropdownOption value=2019/>
    <DropdownOption value=2020/>
    <DropdownOption value=2021/>
</Dropdown>

<BarChart
    data={monthly_earnings_by_type}
    title="Earning by Type, {inputs.content_type.label}"
    x=month
    y=earnings
    series=content_type
    yFmt=usd
/>


<Grid cols=2>
    <ECharts
  config={{
    title: {
      text: 'Earnings Contribution by Content Type',
      left: 'center',     // positions the title horizontally
      top: 'top',         // positions the title vertically
      textStyle: {
        fontSize: 16,
        fontWeight: 'bold'
      }
    },
    tooltip: {
      formatter: '{b}: ${c} ({d}%)'
    },
    series: [
      {
        name: 'Earnings Contribution',
        type: 'pie',
        radius: ['30%', '60%'],
        avoidLabelOverlap: true,
        itemStyle: {
          borderRadius: 4,
          borderColor: '#fff',
          borderWidth: 1
        },
        label: {
          show: true,
          formatter: '{b}\n{d}%'
        },
        emphasis: {
          label: {
            show: true,
            fontSize: 14,
            fontWeight: 'bold'
          }
        },
        data: [...donut_data]
      }
    ]
  }}
/>

    <BarChart
    data={Earnings_per_Engagement}
    title="Earnings per Engagement by Content Type"
    x=content_type
    y=earnings_per_engagement
    series=content_type
    swapXY=true
    yFmt=usd
/>
</Grid>  

<AreaChart
    data={avg_rating_trend_monthly}
    title="Monthly Average Rating with 3-Month Rolling Average"
    x=month
    y={["monthly_avg_rating", "rolling_3mo_avg_rating"]}
    seriesName={{
        monthly_avg_rating: "Monthly Avg Rating",
        rolling_3mo_avg_rating: "3-Month Rolling Avg"
    }}
    yFmt=".2f"
    smooth={true}
    showPoints={true}
    areaOpacity={0.2}
/>

<DataTable
    data={top_bottom_content}
    title="Top & Bottom Content"
/>