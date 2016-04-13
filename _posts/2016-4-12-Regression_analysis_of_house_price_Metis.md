---
layout: post
title: Estimating House Prices in San Francisco
---

##Motivation
When we buy a house, we usually don't know exactly which house we are going to buy, but we know what kind of houses we want.
It is very important to know the price of houses with a specific set of features (size, bathrooms, bedrooms...).

This project is trying to estimate house prices based on the features, using publicly available data.

##Data source
House sales record: http://www.sfgate.com/webdb/homesales/
House features: Zillow API. Including Finished square foot, Lot square foot, Number of bedrooms, bathrooms, total rooms, Built year, House type (single family house, condo…)
and Neighborhood.
S&P/Case-Shiller house price index: https://research.stlouisfed.org/fred2/series/SFXRSA

###Web scraping of rendered webpage
The house sales record from sfgate is a rendered webpage. This means I can't get the data by using the method I usually use for web scraping.
One method to scrape the rendered webpage is to look at the Network tag in Inspect and find the right url of the data.
But in this case, I had to use PyQt4 package in Python to be able to do the scraping.
The code for scraping is shown below:

```python
import sys
from PyQt4.QtGui import *
from PyQt4.QtCore import *
from PyQt4.QtWebKit import *

import requests
import pprint
from bs4 import BeautifulSoup
import re
import pandas as pd
import time

def loadPage(url):

    page = QWebPage()
    loop = QEventLoop() # Create event loop
    page.mainFrame().loadFinished.connect(loop.quit) # Connect loadFinished to loop quit
    page.mainFrame().load(QUrl(url))
    loop.exec_() # Run event loop, it will end on loadFinished
    return page.mainFrame().toHtml()

app = QApplication(sys.argv)

for i in range(1,1000): 
    url = 'http://www.sfgate.com/webdb/homesales/?appSession=64329322129873111204574554973483280103329065656847042053181216317168570306162401356813312547725230111651362605283184354571771674&RecordID=&PageID=2&PrevPageID=1&cpipage='+str(i)+'&CPISortType=&CPIorderBy='

    html = loadPage(url)
    soup = BeautifulSoup(str(html.toAscii()))
    info = soup.find_all('font', {'face':"verdana"})

    print i, len(info)
    #c = 0
    df = pd.DataFrame()
    for e in info:
        time.sleep(1.0)
        #print c
        #c += 1
        data = {}
        house = e.text.split(',')
        #print house
        data['address'] = house[0]
        data['house_info'] = house[1]
        
        df = df.append(data, ignore_index=True)

    df.to_csv('sfdata.csv', mode='a+', header=False)

app.exit()
```

###Time series analysis
To analyze all the house sales price base on the features, I need to adjust the houses prices to the same month. Because it is obvious
the value of a house is different now as it was three years ago.
In this model, I assumed that in the past three years all the values of the houses had the same increase rate, which followed the same 
trend as the house price index.
Unfortunately, The house sales record I got was till Feb, 2016. but the house price index was delayed, only till Jan, 2016.
So a simple time series analysis using ARMA (Autoregression moving average) was made in order extend the house price index to Feb, 2016.
ARMA is a very basic time series method, it is not a good idea to use it to forecast the house price in the future (after a year or so).
But for just one step ahead prediction, it gives a very good result.

##Clustering the neighborhood
The problem of analyzing the house price is that there were not many house sales. San Francisco has 71 neighborhoods, but some neighborhoods
only had a few of transitions in a year. Clustering the neighborhoods became a very clear approach.

The clustering was based on the market behavior, which means the average number of sales per month and the average price per square feet.
The result of the clustering is shown as below by usint tableau public.

![_config.yml]({{ site.baseurl }}/images/plot21.png)

##Regression models
After all the prepare, different models were tested. Single family houses and condos were trained seperately.
The performance of the models were evaluated by cross-validation. (see table below)
Among all the models, randomforest had the best performance and it is interesting that all the models performed better on single family
houses than condos.

![_config.yml]({{ site.baseurl }}/images/plot22.png)

##Important features
In randomforest and tree regression, the relative rank of a feature used as a decision node in a tree can be used to assess the relative
importance of that feature. Features used at the top of the tree have a larger fraction contribution to the final prediction. The expected
fraction of the sample they contributed to can thus be used as an estimation of the relative importance of the features.
The important features for single family houses and condos are shown as below:

![_config.yml]({{ site.baseurl }}/images/plot23.png)

As we can see, beside the size of the houses neighborhood, built year and the interaction term between the size and room of the house 
are also very important for both single family houses and condos. Neighborhood plays a really important part in the single family houses
but less important for condos. 

##Future
It is obvious that some of the important features were not included in my model, such as kitchen, pool, style of the house.
With more features and more data, better prediction can be made.

Forecasting is very important for the real estate market. 
By combining economic data, it is possible to give a forecasting of the price in the future.



