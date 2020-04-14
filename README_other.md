# ------------------ Choose a type of test ------------------

1. focus
2. nonfocus

# ------------------ Read data ------------------

1. orders = complete order data
2. product_master = test information
3. outlets = client information

# ------------------ Find focus test ------------------

1. DB Table: product_master
2. focus_test = "SKUCode", where (product_master['attr1'] == '') & (product_master['sby'] == '')

# ------------------ Filter data ------------------

## Recent Data:

filter last 6 month data (which has complete month data)
['2019-09', '2019-10', '2019-11', '2019-12', '2020-01', '2020-02']

## Focus test:

1. DB Table: orders
2. take records where "code" is within focus_test

# ------------------ recommendation using collaborative filtering ------------------

## User Matrix:

outlets avg monthly order "qty" contribution for each test:
    - order for each, test in every month, for each outlet
    - total order in every month, for each outlet
    - ratio of these two

## Item Matrix:

find test ranking:
    - for each test, in every month, no of client order
    - in every month, no of client order
    - ratio of these two

## Utility Matrix (user-item):

1. use autoencoder, to reduce above 2 dimention to 1 dimentional latent factor
2. convert it to pivot table:
    - index: client_code
    - columns: test_code
    - value: 1 dimentional latent factor

## Find similar clients

1. use autoencoder, to reduct this sparse matrix (all test_code in column):
    - reduce 258 column into, 100 dimentional latent factors
    - now this is Utility Matrix Encoded

2. use similarity score to find top 10 (including itself) similar outlets from "Utility Matrix Encoded"

## Market Basket Analysis within similar client group

1. Preprocessing (in original "utility_matrix"):
    - values > 0 to 1
    - values <= 0 to 0
2. use this new binary value utility_matrix to find correlated tests
3. filter min top 5 correlated tests by sorting in ['confidence', 'lift', 'support']

# ------------------ recommendation using de-growing trend ------------------

## create rolling window:

1. set window = 2
2. input = ['2019-09', '2019-10', '2019-11', '2019-12', '2020-01', '2020-02']

3. rolling_month = [['2019-09', '2019-10'],
                    ['2019-10', '2019-11'],
                    ['2019-11', '2019-12'],
                    ['2019-12', '2020-01'],
                    ['2020-01', '2020-02']]

## do for each rolling_month, each client, each test

### create features:

1. recency (time since last order in days):
    - get max order date for all client
    - days differance within, max order and last order (specific client)
2. frequency (avg days between two order): avg days gap between two order
3. monetary (avg order value): avg order "qty"

### scoring:

1. create percentiles range = ['0% - 20%', '21% - 40%', '41% - 60%', '61% - 80%', '81% - max']

2. ranking for each feature: 
        rfm_score = {
            'recency': {
                '0% - 20%': 5,
                '21% - 40%': 4,
                '41% - 60%': 3,
                '61% - 80%': 2,
                '81% - max': 1
            },
            'frequency': {
                '0% - 20%': 1,
                '21% - 40%': 2,
                '41% - 60%': 3,
                '61% - 80%': 4,
                '81% - max': 5
            },
            'monetary': {
                '0% - 20%': 1,
                '21% - 40%': 2,
                '41% - 60%': 3,
                '61% - 80%': 4,
                '81% - max': 5
            }
        }
3. rfm_score = (rfm['recency_score'] * 0.3) + (rfm['frequency_score'] * 0.3) + (rfm['monetary_score'] * 0.3)

## find down-growing test:

1. label encoding rolling_month column by:
    {"['2019-09', '2019-10']": 0,
    "['2019-10', '2019-11']": 1,
    "['2019-11', '2019-12']": 2,
    "['2019-12', '2020-01']": 3,
    "['2020-01', '2020-02']": 4}

2. do for each each client, each test:
    - draw a linear regression line on rfm_score for each rolling_month
    - trend = coefficient of that line
    - if trend < 0 then, de-growing
      else, growing
    - list de-growing tests for each client

# ------------------ Final recommendation ------------------

## 3 types of recommendation:
    - HIGH
    - MID
    - LOW
## Recommendation priority:

    1. HIGH recommendation:
        - test identified in collaborative_filtering and
        - test identified in de-growing
    2. MID recommendation:
        - other test identified in collaborative_filtering excluding "HIGH recommendation" tests
    3. LOW recommendation:
            1. Step1:
                - degrowing_product_remains = that client's degrowing tests, excluding "HIGH" & "MID" recommended tests
                - tests belongs to:
                    - degrowing_product_remains and
                    - other similar client's collaborative_filtering tests
            2. Step2:
                - that client's remaining degrowing tests, excluding "HIGH" & "MID" & "LOW (Step1)" recommended tests

    4. Maintain order of tests ("HIGH" --> "MID" --> "LOW")

## Top recommendation choosing:

1. take only top 3 tests, if only "LOW" priority recommendation exists
2. take top 5 tests, if other priority recommendation also exists
3. but in both case there can be maximum 3 "LOW" priority test exist