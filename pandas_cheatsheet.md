# Pandas Cheatsheet

---

## DataFrame Creation

| Method | Example |
|--------|---------|
| From list of dicts | `pd.DataFrame([{'a': 1, 'b': 2}, {'a': 3, 'b': 4}])` |
| From list of lists | `pd.DataFrame([[1,2],[3,4]], columns=['a','b'])` |
| From CSV | `pd.read_csv('file.csv')` |
| CSV with separator | `pd.read_csv('file.csv', sep=';')` |
| CSV without header | `pd.read_csv('file.csv', header=None, names=['a','b','c'])` |

---

## Inspection

| Method | What it does |
|--------|-------------|
| `df.shape` | (rows, columns) |
| `df.columns` | Column names |
| `df.index` | Row index |
| `df.dtypes` | Data types per column |
| `df.info()` | dtypes + non-null counts |
| `df.describe()` | Stats summary (numeric cols) |
| `df.head(n)` | First n rows |
| `df.tail(n)` | Last n rows |
| `df.sample(n)` | n random rows |
| `df.count()` | Non-null count per column |

---

## Selecting Data

```python
df['col']                        # Single column (Series)
df[['col1', 'col2']]             # Multiple columns (DataFrame)
df[:3]                           # First 3 rows (slicing)
```

---

## loc vs iloc

| | `loc` (label-based) | `iloc` (integer-based) |
|-|---------------------|------------------------|
| Single cell | `df.loc[0, 'col']` | `df.iloc[0, 0]` |
| Row range | `df.loc[0:10, 'col']` | `df.iloc[0:10, 0]` |
| Multi-col | `df.loc[10:15, ['a','b']]` | `df.iloc[10:15, [0,4]]` |

> `loc` is **inclusive** on both ends; `iloc` **excludes** the stop index.

---

## Filtering

```python
df[df['length'] <= 0.4]                                      # Boolean mask
df[(df['length'] <= 0.4) & (df['sex'] == 'F')]               # AND condition
df[(df['length'] <= 0.4) | (df['sex'] == 'F')]               # OR condition
df[~(df['sex'] == 'F')]                                       # NOT — rows where sex is NOT 'F'
df.query('length <= 0.4 and sex == "F"')                     # query() — cleaner syntax for complex filters
df.query('length > 0.35 and weight_whole > 0.25')            # multiple conditions, no & needed

# isin — match against a list of values
df[df['sex'].isin(['F', 'M'])]
df[~df['sex'].isin(['I'])]                                    # exclude

# between — inclusive range filter
df[df['length'].between(0.3, 0.5)]
```

---

## Series Methods

| Method | What it does |
|--------|-------------|
| `s.unique()` | Array of unique values |
| `s.nunique()` | Count of unique values |
| `s.value_counts()` | Frequency of each value |
| `s.value_counts(normalize=True)` | Relative frequency (proportions) |
| `s.count()` | Non-null count |
| `s.sum()` | Sum |
| `s.mean()` | Mean |
| `s.median()` | Median |
| `s.mode()[0]` | Most common value |
| `s.min()` | Minimum |
| `s.max()` | Maximum |
| `s.std()` | Standard deviation |
| `s.var()` | Variance |
| `s.agg(['min','max','mean'])` | Multiple aggregations at once |
| `s.isna()` | Boolean mask of NaN values |
| `s.notna()` | Boolean mask of non-NaN values |

---

## GroupBy

```python
df.groupby('sex').mean()                                        # Mean of all numeric cols by group
df.groupby('sex')['length'].mean()                              # Mean of one specific col by group
df.groupby(['sex', 'rings']).max()                              # Group by multiple cols
df.groupby('sex').size()                                        # Count all rows per group (includes NaN)
df.groupby('sex').count()                                       # Count non-null values per group per col
df.groupby('sex').agg({'length': 'mean', 'weight_whole': 'max'})  # Different agg per column
```

---

## Sorting

```python
df.sort_values('col')                              # Ascending (default)
df.sort_values('col', ascending=False)             # Descending
df.sort_values(['col1', 'col2'], ascending=False)  # Both cols sorted descending
df.sort_values(['col1', 'col2'], ascending=[True, False])  # col1 asc, col2 desc
df.nlargest(5, 'col')                              # Top 5 rows by col value
df.nsmallest(5, 'col')                             # Bottom 5 rows by col value
```

---

## Column Operations

```python
df.rename(columns={'old': 'new'}, inplace=True)
df.columns = ['a', 'b', 'c']                    # Rename all columns at once
df.drop('col', axis=1, inplace=True)            # Drop column
df.drop(['col1','col2'], axis=1, inplace=True)  # Drop multiple
df['new_col'] = df['col1'] * df['col2']         # Arithmetic

# Type casting
df['col'].astype(int)
df['col'].astype(float)
df['col'].astype(str)

# Index operations
df.set_index('col', inplace=True)               # Set column as index (col removed from columns)
df.reset_index(drop=True, inplace=True)         # Reset to 0,1,2… index; drop=True discards old index instead of making it a column
```

---

## eval()

```python
df.eval('age = nr_rings + 1.5', inplace=True)   # New column from expression; inplace modifies df directly
df.eval('ratio = col1 / col2', inplace=True)    # Division between two columns
df.eval('total = a + b + c', inplace=True)      # Sum across multiple columns
```

---

## Handling Missing Data

```python
df.isna().sum()                        # Count NaNs per column
df.isna().any()                        # True for any col that has NaN
df.fillna(0, inplace=True)             # Fill NaN with a value
df.fillna(df.mean(numeric_only=True))  # Fill numeric cols with their mean
df.ffill()                             # Forward fill — copies last valid value downward
df.bfill()                             # Backward fill — copies next valid value upward
df.dropna(inplace=True)                # Drop rows with any NaN
df.dropna(subset=['col'])              # Drop only where specific col is NaN
df.dropna(how='all')                   # Drop rows where ALL values are NaN
```

---

## Datetime — Python stdlib

```python
from datetime import date, time, datetime, timedelta

date(2024, 1, 15)                              # date object
date.today()                                   # today's date
datetime(2024, 1, 15, 10, 30, 0)              # datetime object
datetime.now()                                 # current datetime

dt = datetime(2024, 1, 15, 10, 30, 0)
dt.strftime('%d/%m/%Y')                        # datetime → string: '15/01/2024'
dt.weekday()                                   # 0=Mon … 6=Sun
dt.isoweekday()                                # 1=Mon … 7=Sun
dt.isocalendar()                               # (year, week_number, weekday)

datetime.strptime('2024-01-15', '%Y-%m-%d')   # string → datetime
```

---

## Datetime — timedelta

```python
timedelta(days=7)                             # 1 week offset
timedelta(hours=3, minutes=30)                # 3.5 hour offset

d1 = datetime(2024, 1, 1, 8, 0, 0)
d2 = datetime(2024, 1, 2, 11, 30, 0)
duration = d2 - d1                            # timedelta of 27.5 hours

# ⚠️  Three different attributes — easy to confuse:
duration.days                                 # 1          ← full days only
duration.seconds                              # 12600      ← leftover seconds after full days (max 86399); NOT total
duration.total_seconds()                      # 99000.0    ← days*86400 + seconds; use this for real arithmetic

# Convert to hours
duration / timedelta(hours=1)                 # 27.5  ← preferred readable form
duration.total_seconds() / 3600              # 27.5  ← equivalent
```

---

## Datetime — Pandas

```python
pd.to_datetime('2024-01-15')                              # string → Timestamp
pd.to_datetime(df['date'], format='%Y/%m/%d')             # convert whole column to datetime dtype
pd.to_timedelta(1, unit='D')                              # create timedelta of 1 day

# date_range — generates a sequence of dates
pd.date_range(start='2024-01-01', end='2024-01-31', freq='D')    # daily, Jan 2024 (31 dates)
pd.date_range(start='2024-01-01', periods=10, freq='h')          # 10 hourly timestamps
pd.date_range(start='2024-01-01', periods=10, freq='min')        # 10 minute-by-minute timestamps
# freq options: 'D'=day, 'h'=hour, 'min'=minute, 'W'=week, 'ME'=month-end
```

---

## dt Accessor

```python
df['date'] = pd.to_datetime(df['date'], format='%Y/%m/%d')

df['date'].dt.date             # date part only (no time)
df['date'].dt.day
df['date'].dt.month
df['date'].dt.year
df['date'].dt.hour
df['date'].dt.minute
df['date'].dt.second
df['date'].dt.weekday          # 0=Mon … 6=Sun
df['date'].dt.day_name()       # 'Monday', 'Tuesday', …
df['date'].dt.quarter          # 1–4
df['date'].dt.is_month_end     # True/False
```

---

## String Accessor (.str)

```python
df['col'].str.lower()                  # lowercase all
df['col'].str.upper()                  # uppercase all
df['col'].str.strip()                  # remove leading/trailing whitespace
df['col'].str.replace('old', 'new')    # replace substring
df['col'].str.contains('pattern')      # boolean mask — rows that match (supports regex)
df['col'].str.contains('pat', case=False)  # case-insensitive match
df['col'].str.startswith('A')          # True for values beginning with 'A'
df['col'].str.endswith('s')            # True for values ending with 's'
df['col'].str.len()                    # length of each string
df['col'].str.split(',')               # split into lists
df['col'].str.split(',', expand=True)  # split into separate columns
```

> `.str` methods are vectorised — no loop needed. Works on object-dtype columns.

---

## Plotting

```python
df['col'].plot(kind='hist', bins=20, title='Title', xlabel='X')  # distribution of one column
df['col'].plot(kind='bar')                                        # frequency bar chart
df['col'].plot(kind='line')                                       # trend over index
df.plot(kind='scatter', x='col1', y='col2')                      # relationship between two cols
df['col'].value_counts().plot(kind='pie')                         # proportion of each category
df.plot(kind='box')                                               # spread & outliers for all numeric cols

from pandas.plotting import scatter_matrix
scatter_matrix(df[['col1', 'col2', 'col3']])   # pairwise scatter plots + histograms on diagonal

import matplotlib.pyplot as plt
plt.show()                                      # display plot (needed outside Jupyter)
```

---

## Common Format Codes (strftime / strptime)

| Code | Meaning | Example |
|------|---------|---------|
| `%Y` | 4-digit year | 2024 |
| `%m` | Month (01–12) | 01 |
| `%d` | Day (01–31) | 15 |
| `%H` | Hour 24h (00–23) | 14 |
| `%M` | Minute (00–59) | 30 |
| `%S` | Second (00–59) | 05 |
