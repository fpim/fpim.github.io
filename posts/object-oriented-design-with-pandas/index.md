<!--
.. title: Object-Oriented Design architecture with Pandas
.. slug: object-oriented-design-with-pandas
.. date: 2019-09-06 15:29:14 UTC+08:00
.. tags: 
.. category: 
.. link: 
.. description: 
.. type: text
-->

Hi guys. Today I would like to talk about implementing Object-oriented (OO) Design to build a Business Intelligence module with Python and Pandas. 

Pandas is widely used for data analytics, it is easy to use, one can easily build a script of hundreds of lines of codes
using pandas to handle complex data. However, having worked with many data analysts, I found that most of the scripts produced by data analysts should fall into the category of "procedural code", which is not reusable.
Changing the context a bit would often result in a complete re-write of the code. It is fine if you are in the "research" area where your code
are used for "exploration" or "experimental" purposes, but if you are to write code that contributes
to a larger system or code that will be run repeatedly, you need to implement some architecture.

## My Background

I work in finance where I need to handle data with complex relationship daily.
The raw data that I received is usually too "raw" such that extracting any useful information from it often requires a lot of steps. Code could easily become impossible to maintain due to the mismatch of raw data structure and business logic.


#### The solution
 
Soon I realized that I needed a OO layer on top of the raw data to handle all the complexity. 
The reason I use the term **OO layer** here is that OO is essentially an abstraction layer. 
**It allows you to focus on business logic only, instead of having to worry about how to implement business from raw data.**
Architecture is all about reducing your cognitive burden when you try to edit a codebase.


# The Data

We will try to build a OO model for some trade data:

|Account,|Stock_Code,|BuySell,|Date,|Quantity,|Unit_Price|
|---|---|---|---|---|---|
|1001|001|B|20190105|8|2|
|1001|002|B|20190105|4|3|
|1001|002|S|20190106|3|1|
|1001|003|B|20190106|6|5|
|1001|003|S|20190106|4|6|
|1001|001|S|20190107|6|3|
|1002|001|B|20190105|6|2|
|1002|002|B|20190105|8|3|
|1002|002|S|20190106|3|1|
|1002|003|B|20190106|5|5|
|1002|003|S|20190106|4|6|
|1002|001|S|20190107|6|3|

<br/><br/>
some account information:

|Account,|Name|
|---|---|
|1001|David|
|1002|Tom|

<br/><br/>
and some closing price data:

|Stock_Code,|Closing_Price,|Data|
|---|---|---|
|001|2|20190105|
|001|3|20190106|
|001|2|20190107|
|002|2|20190105|
|002|3|20190106|
|002|5|20190107|
|003|5|20190105|
|003|6|20190106|
|003|7|20190107|


First, we will have a "Data Layer" to handle the i/o of source data. since we don't have the actual data source, we will 
mock it up:

```python
import pandas as pd
from io import StringIO

class Data():

    def trade_df(self):
        # This method should handle the import of source data
        data = StringIO()

        s = """
        Account|Stock_Code|BuySell|Date|Quantity|Unit_Price
        1001|001|B|20190105|8|2
        1001|002|B|20190105|4|3
        1001|002|S|20190106|3|1
        1001|003|B|20190106|6|5
        1001|003|S|20190106|4|6
        1001|001|S|20190107|6|3
        1002|001|B|20190105|6|2
        1002|002|B|20190105|8|3
        1002|002|S|20190106|3|1
        1002|003|B|20190106|5|5
        1002|003|S|20190106|4|6
        1002|001|S|20190107|6|3
        """

        data.write(s.replace(' ',''))
        data.seek(0)
        df = pd.read_csv(data,sep='|',dtype={'Account':str,
                                                'Date':str,
                                                'Stock_Code':str,
                                                "Date":str})
        df.Date = pd.to_datetime(df.Date,format='%Y%m%d')
        return df

    def account_df(self):
        data = StringIO()

        s = """
        Account|Name
        1001|David
        1002|Tom
        """

        data.write(s.replace(' ',''))
        data.seek(0)
        df = pd.read_csv(data, sep='|', dtype={'Account': str,
                                                  'Name': str})
        return df

    def stock_prices(self):

        data = StringIO()

        s = """
        Stock_Code|Closing_Price|Date
        001|2|20190105
        001|3|20190106
        001|2|20190107
        002|2|20190105
        002|3|20190106
        002|5|20190107
        003|5|20190105
        003|6|20190106
        003|7|20190107
        """

        data.write(s.replace(' ',''))
        data.seek(0)
        df = pd.read_csv(data, sep='|', dtype={'Stock_Code': str,
                                                  'Date': str})
        df.Date = pd.to_datetime(df.Date,format='%Y%m%d')
        return df
``` 
The `Data` object should handle the import and validation of the data only, and it should not implement and business logic.

Then on top of the `Data` object, we will build our first OO layer:

```python
class Book():
    def __init__(self):
        self._data = Data()

    @property
    def trade_book_df(self):
        if not hasattr(self,'_trade_book_df'):
            df = self._data.trade_df().join(self._data.account_df().set_index('Account'),on='Account')
            df['Trade_Amount'] = df.Quantity * df.Unit_Price
            self._trade_book_df = df
        return self._trade_book_df

    def stock_prices(self,date=None):
        df = self._data.stock_prices()
        if date:
            df = df[df.Date == date]
        return df

    @property
    def holdings(self):
        return Holdings(self)

    @property
    def accounts(self):
        return Accounts(self.holdings)

Book().trade_book_df

|    |   Account |   Stock_Code | BuySell   | Date                |   Quantity |   Unit_Price | Name   |   Trade_Amount |
|---:|----------:|-------------:|:----------|:--------------------|-----------:|-------------:|:-------|---------------:|
|  0 |      1001 |          001 | B         | 2019-01-05 00:00:00 |          8 |            2 | David  |             16 |
|  1 |      1001 |          002 | B         | 2019-01-05 00:00:00 |          4 |            3 | David  |             12 |
|  2 |      1001 |          002 | S         | 2019-01-06 00:00:00 |          3 |            1 | David  |              3 |
|  3 |      1001 |          003 | B         | 2019-01-06 00:00:00 |          6 |            5 | David  |             30 |
|  4 |      1001 |          003 | S         | 2019-01-06 00:00:00 |          4 |            6 | David  |             24 |
|  5 |      1001 |          001 | S         | 2019-01-07 00:00:00 |          6 |            3 | David  |             18 |
|  6 |      1002 |          001 | B         | 2019-01-05 00:00:00 |          6 |            2 | Tom    |             12 |
|  7 |      1002 |          002 | B         | 2019-01-05 00:00:00 |          8 |            3 | Tom    |             24 |
|  8 |      1002 |          002 | S         | 2019-01-06 00:00:00 |          3 |            1 | Tom    |              3 |
|  9 |      1002 |          003 | B         | 2019-01-06 00:00:00 |          5 |            5 | Tom    |             25 |
| 10 |      1002 |          003 | S         | 2019-01-06 00:00:00 |          4 |            6 | Tom    |             24 |
| 11 |      1002 |          001 | S         | 2019-01-07 00:00:00 |          6 |            3 | Tom    |             18 |

```

1. The `Book` object has an `_data` attribute which owns the `Data` object.
2. The `trade_book_df` property implemented two business logic:
    - The joining of `trade_df` and `account_df`.
    - The definition of `Trade_Amount` -> `Quantity` * `Unit_Price`.
3. The `stock_prices` method extracts stock prices of a specific date.
4. It provides the gateways to the `Holdings` and `Accounts` context which we are going to talk about.

## Contexts

"Context" is the dimension you want to view the data, all the views with the same dimension should be encapsulated 
in one object.

For example, to view the data in the "Holding" context, we need to:
1. aggregate the trade data up to a certain point of time.
2. multiply holding quantity and stock's closing price to obtain the market value.


```python
class Holdings():
    def __init__(self,book):
        self._book = book

    def holdings_of(self,date):
        trades_df = self._book.trade_book_df
        date_hld_df = trades_df[trades_df.Date <= date]
        date_hld_df['qnt_change'] = date_hld_df['BuySell'].map({'B':1,'S':-1}) * date_hld_df.Quantity
        hld_df = date_hld_df.groupby(['Account','Stock_Code'],as_index=False)\
            .agg({'qnt_change':sum})\
            .rename(columns={'qnt_change':'Holdings'})
        hld_df = hld_df.join(self._book.stock_prices(date).set_index('Stock_Code'),on='Stock_Code')
        hld_df['Market_Value'] = hld_df.Closing_Price * hld_df.Holdings
        return hld_df
        
from datetime import date
Book().holdings.holdings_of(date(2019,1,6))

|    |   Account |   Stock_Code |   Holdings |   Closing_Price | Date                |   Market_Value |
|---:|----------:|-------------:|-----------:|----------------:|:--------------------|---------------:|
|  0 |      1001 |          001 |          8 |               3 | 2019-01-06 00:00:00 |             24 |
|  1 |      1001 |          002 |          1 |               3 | 2019-01-06 00:00:00 |              3 |
|  2 |      1001 |          003 |          2 |               6 | 2019-01-06 00:00:00 |             12 |
|  3 |      1002 |          001 |          6 |               3 | 2019-01-06 00:00:00 |             18 |
|  4 |      1002 |          002 |          5 |               3 | 2019-01-06 00:00:00 |             15 |
|  5 |      1002 |          003 |          1 |               6 | 2019-01-06 00:00:00 |              6 |
       
```

You can then build context on top of context. For example you can have an `Accounts` context built on top of the 
`Holdings` context:

```python
class Accounts():
    def __init__(self,holdings):
        self._holdings = holdings

    def account_value(self,date):
        df = self._holdings.holdings_of(date)
        return df.groupby('Account',as_index=False).agg({'Market_Value':sum,
                                                         'Date':'first'})
                                                         
Book().accounts.account_value(date(2019,1,6))

|    |   Account |   Market_Value | Date                |
|---:|----------:|---------------:|:--------------------|
|  0 |      1001 |             39 | 2019-01-06 00:00:00 |
|  1 |      1002 |             39 | 2019-01-06 00:00:00 |

```

# Summary
The above is a simple example of how OO Design can be applied to your pandas script. The architecture allows you to 
scale up easily as the number of data source increase. This has been just a very brief introduction, there are many more 
modeling techniques you can use to manage complex data. Finally, I want to stress that these techniques are becoming more and more important. 
Traditionally, this kind of problem can be handled by the database and relational model, but given the growing complexity and scatteredness of the data, you are
more likely to encounter situations where your database cannot handle everything and you will need some supplement from your code,
especially when you need data from multiple data source.
