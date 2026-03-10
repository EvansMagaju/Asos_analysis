# ASOS Product Data Analysis

This notebook performs an exploratory data analysis on ASOS product data, focusing on brand performance, stockout rates, and lost revenue.

## Project Overview
The goal of this analysis is to identify key insights regarding product availability and its financial implications for various brands on the ASOS platform. We analyze product descriptions to extract brand information, calculate stockout rates, and estimate lost revenue due to out-of-stock items.

## Data Loading and Preprocessing
The analysis begins by loading product data from `products_asos.csv` into a pandas DataFrame. The 'price' column is converted to a numeric type, and rows with missing price data are removed. Initial data shape and a preview of the DataFrame are provided.

```python
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns

df = pd.read_csv('products_asos.csv')
df['price'] = pd.to_numeric(df['price'], errors='coerce')
df = df.dropna(subset=['price'])
print(f"Data Loaded {len(df)} row")
df.head()
```

## Brand Extraction
A custom function `get_brand` is used to extract brand names from the 'description' column. This involves parsing the text to find the brand mentioned after 'by'. A `brand_raw` column is created, and then a `brand_map` is applied to standardize some brand names, resulting in a cleaned 'Brand' column.

```python
df['description'] = df['description'].astype(str)

def get_brand(text):
  if 'by ' in text:
    try:
      return text.split('by ')[1].strip(' ')
    except:
      return "Unknown"
  return "Unknown"

df['brand_raw'] = df['description'].apply(get_brand)

brand_map = {
            'New': 'New Look',
            'River': 'River Island',
            'Miss': 'Miss Selfridge',
            'TopshopWelcome': 'Topshop'
}

df['Brand'] = df['brand_raw'].map(brand_map).fillna(df['brand_raw'])

brand_counts = df['Brand'].value_counts()
valid_brands = brand_counts[brand_counts > 5].index
df_cleaned = df[df['Brand'].isin(valid_brands)].copy()

print(df_cleaned['Brand'].value_counts().head(5))
```

## Stockout Analysis and Lost Revenue Calculation
We define a function `calculate_phantom_revenue` to determine the number of sizes out of stock and the corresponding stockout rate for each product. This information is then used to calculate 'Lost_Revenue' by multiplying the product's price by the 'Stockout_Count'. The products with the highest lost revenue are identified.

```python
def calculate_phantom_revenue(size_str):
  if not isinstance(size_str, str):
    return 0, 0.0

  sizes = size_str.split(',')
  total_sizes = len(sizes)

  Out_of_stock = size_str.count('Out of stock')

  rate = Out_of_stock / total_sizes if total_sizes > 0 else 0.0

  return Out_of_stock, rate

metrics = df_cleaned['size'].apply(lambda x: calculate_phantom_revenue(x))
df_cleaned['Stockout_Count'] = [x[0] for x in metrics]
df_cleaned['Stockout_Rate'] = [x[1] for x in metrics]
df_cleaned['Lost_Revenue'] = df_cleaned['price'] * df_cleaned['Stockout_Count']

cols = ['Brand', 'name', 'price', 'Stockout_Count', 'Lost_Revenue']
print(df_cleaned.sort_values(by='Lost_Revenue', ascending=False).head(5)[cols])
```

## Brand Strategy Analysis (Scatter Plot)
A scatter plot is generated to visualize brand strategy, showing the relationship between average price, stockout rate, and total lost revenue for brands. Brands with an average price above £40 and a stockout rate greater than 0.4 are highlighted as potential "winners" – indicating high demand with significant lost revenue opportunities.

```python
brand_strategy = df_cleaned.groupby('Brand').agg({
    'price': 'mean',
    'Stockout_Rate': 'mean',
    'Lost_Revenue': 'sum',
    'name': 'count'
}).reset_index()

brand_strategy = brand_strategy[brand_strategy['name'] > 10]

plt.figure(figsize=(12,10))
sns.scatterplot(
    data=brand_strategy,
    x='price',
    y='Stockout_Rate',
    size='Lost_Revenue',
    sizes=(50,500),
    alpha=0.7,
    palette='viridis'
)

winners = brand_strategy[
    (brand_strategy['price']>40)&
    (brand_strategy['Stockout_Rate']>0.4)

]

for i in range(len(winners)):
  plt.text(
      winners.iloc[i]['price'],
      winners.iloc[i]['Stockout_Rate'],
      winners.iloc[i]['Brand'],

