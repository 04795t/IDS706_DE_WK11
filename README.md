# IDS706_DE_WK11

# PySpark Data Processing Pipeline

## Dataset Description

**Name:** eCommerce behavior data from multi category store 
**Source:** [Kaggle eCommerce Behavior Dataset](https://www.kaggle.com/datasets/mkechinov/ecommerce-behavior-data-from-multi-category-store?resource=download&select=2019-Oct.csv)  
**File:** 2019-Oct.csv
**Size:** 5.67 GB  
**Format:** CSV  
**Total Records:** 42,448,764 events

### Dataset Overview

This dataset contains user behavior data from a large multi-category online store collected during October 2019. Each row represents a user interaction event with the following attributes:

- event_time: Timestamp when the event occurred
- event_type: Type of event (view, cart, purchase)
- product_id: Unique product identifier
- category_id: Numerical category identifier
- category_code: Hierarchical category name (e.g., electronics.smartphone)
- brand: Product brand name
- price: Product price in USD
- user_id: Unique user identifier
- user_session: Session identifier

The dataset captures the complete customer journey from browsing to purchase, which makes this dataset ideal for analyzing e-commerce conversion funnels, user behavior patterns, and revenue optimization strategies.

---

## Pipeline Architecture

### Data Processing

The pipeline processes 42,448,764 e-commerce events through a series of transformations and aggregations:

1. Data Loading: Loaded 5.67GB CSV file using PySpark with schema inference
2. Data Cleaning: Applied two filter operations to remove invalid records and focus on conversion events
3. Feature Engineering: Created derived columns for temporal analysis and price categorization
4. Aggregation Analysis: Performed multi-dimensional groupBy operations to extract business insights
5. SQL Analytics: Executed complex SQL queries for brand performance and temporal patterns
6. Results Persistence: Saved processed data to Parquet and CSV formats

### Transformations Applied

**Filter Operations:**
- Filter 1: Removed records with invalid prices, missing brands, or null user IDs (reduced from 42.4M to 36.3M rows)
- Filter 2: Focused on cart and purchase events only, excluding view-only behavior (reduced to 1.6M rows, 96.2% reduction)

**Column Transformations:**
- Extracted event_hour from timestamp for temporal analysis
- Created price_category classification (budget, mid-range, premium, luxury)

**Aggregations:**
- Brand performance metrics (total events, unique users, average price, total revenue)
- Hourly shopping patterns with multi-dimensional grouping (hour and event type)
- Price category analysis across event types

---

## Requirements

- **PySpark 4.0.1** - Distributed data processing framework
- **Python 3.12** - Programming language
- **Spark SQL** - SQL query engine
- **MLlib** - Machine learning library (for optional price prediction model)

---

## Setup

### Prerequisites

```bash
pip install pyspark jupyter
```

### Steps to run analysis

1. Download the 2019-Oct.csv dataset from Kaggle
2. Place the CSV file in the Dataset/ directory
3. Open the Jupyter notebook
4. Run all cells in sequence
5. Review generated outputs in the output/ directory

> **Note:** This project was ran locally since the dataset was too large to upload to Databricks. The notebook uses a local PySpark session with 8 GB memory and all CPU cores for distributed processing.


> Due to file size limitations, the dataset is not included in this repository. Please download the dataset and place it in the `dataset/` folder before running the project.

### Required Files:
`dataset/2019-Oct.csv` (5.4 GB)

### Download Instructions:
1. Download the dataset from [Kaggle eCommerce Behavior Dataset](https://www.kaggle.com/datasets/mkechinov/ecommerce-behavior-data-from-multi-category-store?resource=download&select=2019-Oct.csv)
2. Create a `dataset/` folder in the project root if it doesn't exist:
```bash
   mkdir -p dataset
```
3. Place the downloaded `2019-Oct.csv` file in the `dataset/` folder

> Note: The `dataset/` folder is included in `.gitignore` to prevent accidentally committing large files.

---

## Performance Analysis

### Query Optimization

Spark's Catalyst optimizer generated an optimized physical execution plan from the logical query plan. An analysis of the execution plan indicated that filter operations had been successfully pushed down to the data source level. Instead of loading the entire data into memory, `price`, `brand`, and `user_id` filters were applied during the initial scan of the CSVs, pushing down the filter operation. This reduced the data read from disk by roughly 14.4% in the first filter stage alone, and up to 95.6% when filtering for cart and purchase events only. Spark combined multiple filter conditions into one scan operation, minimizing I/O operations and reducing overall execution time.

Physical plan analysis by the `explain()` method shows the intelligent query optimization of Spark. It can be observed in this pipeline that column pruning has been used all along, where only the required columns are selected for any operation. In the brand aggregation query, Spark optimizes the `groupBy` operation by doing a partial aggregation before the shuffle stage. This reduces the amount of data transferred across the cluster. Observe a two-step aggregation here, 1. a local pre-aggregation phase; 2. a final global aggregation. The `orderBy` operation has been pushed down and combined with the aggregation where appropriate. This reduces the number of different stages that need to be processed.

### Performance Bottlenecks and Optimizations

The first pipeline had some performance bottlenecks with the multi-dimensional aggregations, particularly the hourly pattern analysis that groups by the two columns: `event_hour` and `event_type`. This was an expensive operation that caused a shuffle, which redistributed 1.6 million rows across partitions-a necessary but costly step. To make this more efficient, filters were ordered in such a way as to execute before the expensive `groupBy` operations, reducing the volume of data before shuffling. This filter pushdown optimization was quite effective since the execution plan showed filters integrated right into the FileScan with the notation "PushedFilters" appearing at the scan stage.

Caching optimization provided performance improvements for repeated operations. When `brand_stats` DataFrame is cached and used multiple times, the second execution from the cache achieved a 3.2x speedup over recomputation from original data. First execution after caching took similar time to build the cache, but subsequent operations referring to the cached data completed in about 1/3 of the original time.

The partitioning was appropriate for the pipeline. By default, Spark created 43 partitions for the 5.67 GB input file. The number of partitions gave good parallelism across the available CPU cores but did not incur excessive task-scheduling overhead. By default, in case of any aggregation operations, Spark transparently repartitions the data by the grouping keys so that all records with the same key will fall into the same partition for efficient aggregation. It was also evident from the execution plan that shuffle partitions had an appropriate size with no skew and a well-balanced workload across the tasks.

---

## Key Findings

### Business Intelligence Insights

**Top Revenue-Generating Brands**

Apple emerged as the dominant brand with total revenue of $111.2 million from 351,957 events, showing strong market presence even at an average product price of $766. Samsung ranked second with $46.4 million in revenue from 476,145 events at a much lower average price point of $260. This gap between Apple and Samsung revenue, despite Samsung's higher event volume, indicates that Apple's pricing strategy captures higher-value customers. Third place went to Xiaomi, capturing $9.2 million in revenue from 161,113 events at an average price of $151, appealing to budget-conscious consumers.

**Hourly Shopping Patterns**

Cyclical behavior was then represented in the e-commerce activity: The hourly pattern of e-commerce activities peaked within the early morning hours, starting at 0:00-6:00 UTC, with a peak at 5:00 A.M. with 66,024 cart additions versus 51,234 purchases. A global customer base shopping in their local evening therefore seems to be indicated by this pattern. The activity gradually decreased throughout the day to the lowest level recorded at 18:00 UTC: only 4,333 added to cart and 3,592 purchased. It is intriguing to note from the data an inverse relationship between shopping volume and average transaction value; lower-traffic hours, ranging from 14:00 to 18:00, set higher averages, from $371 to $412, while customers are willing to spend more money per transaction when they shop during off-peak hours.

**Price Segmentation**

Mid-range products, falling between $50 and $200, were the largest customer segment, with 624,400 total events. These products would have created a lot of revenue but remained accessible to the customers. The luxury category, above $500, was very strong, with 315,527 events at an average price of $935, which is a high demand for premium products. Budget items, below $50, had 189,809 events but had a lower total revenue since the average price was $30. Premium products, from $200 to $500, struck a great balance of volume and value at 462,609 events and an average price of $314.

**Conversion Funnel Analysis**

Analysis by category of cart-to-purchase conversion indicated very different patterns. Budget items seem to have a slightly higher purchase than cart additions, which may indicating impulse buying behavior. For mid-range products, there were 264,104 purchases against 360,296 cart additions, which indicated good conversion with a little consideration time for purchase. Luxury products had 134,378 purchases out of 181,149 cart additions, again signifying that higher-value purchases take greater time and consideration. This is in keeping with expectations since the overall dataset size was reduced from 42.4 million total events to 1.6 million cart and purchase events. This proved that the majority of e-commerce activity consists of browsing behavior, as only 3.8 percent of events resulted in cart additions or purchases. 

**Brand Competition**

Price category analysis showed competitive dynamics across market segments. The luxury category had only 111 unique brands competing for cart additions but 322 brands achieving actual purchases, which could indicate a difficult time breaking into the high-end market. Mid-range products saw fierce competition, with 370 brands in cart additions and 1,130 brands achieving purchases, indicating a crowded marketplace. For budget products, there were 338 brands for cart additions and 1,145 brands with purchases, which proved to be the most accessible entry point for new brands. This would suggest that brand loyalty and market positioning become increasingly important as price points rise.

---

## Technical Implementation

### Data Processing Statistics

- **Original Dataset:** 42,448,764 rows
- **After Quality Filtering:** 36,335,756 rows (14.4% reduction)
- **After Event Filtering:** 1,592,345 rows (96.2% total reduction)
- **Unique Brands:** 2,013 brands analyzed
- **Unique Users:** Over 500,000 unique customers
- **Processing Time:** 6.8 seconds for initial data load
- **Output Files:** 2,013 brand records in Parquet format, 48 hourly pattern records in CSV

### Optimization Strategies Applied

- Filter Pushdown: All filter operations were applied early, allowing Spark to minimize data movement and I/O operations. The `.explain()` plan confirmed that Spark pushed predicates to the file scan level. 
- Column Pruning: Only necessary columns were selected for each stage, reducing shuffle size and memory use.
- Caching Strategy: Cached the `brand_stats` DataFrame since it was reused multiple times, achieving over 3× faster repeated access
- Aggregation Optimization: Used Spark’s partial and final aggregation stages to reduce shuffle data volume, verified in the physical plan.  
- Partition Management: Maintained ~43 partitions for balanced parallelism without excessive overhead.  

> **Note:** The notebook was executed locally (Spark 4.0.1, 8 GB driver memory, `local[*]` mode) because the dataset (~5.7 GB) was too large to upload to Databricks.

---

## Screenshots

### Execution Plan
![Execution Plan](screenshot/execution_plan1.JPG)
![Execution Plan](screenshot/execution_plan2.JPG)

### Successful Pipeline
![Successful Pipeline](screenshot/successful_pipeline.JPG)

## Query Details
![Query Details](screenshot/query_details.JPG)
