---
published: true
layout: post
categories: blog
title: Scraping Cryptocoin Exchange Data
author: Jonn Callahan
tags:
  - kubernetes
  - coinbase
  - machine learning
description: "I started scraping data from Coinbase and have made it publicly available. Possibly more exchanges to be added later. Each days worth of data is written via HDF5 and backed up to a publicly readable S3 bucket. Don't make me regret it."
---

Coinbase Pro (previously Gdax) has a nice little API for pulling market data. So I decided to use it to scraping all open orders once per minute. I've made this data publicly available via an S3 bucket. The bucket can be found here:

[https://cryptoexchanges.veraciousdata.io/](https://cryptoexchanges.veraciousdata.io/)

## How to Use

The bucket itself is public, so any set of creds can list and pull objects. Object keys are prepended with the exchange the data was pulled from, though at the moment I'm only scraping Coinbase:

```python
>>> import boto3
>>> client = boto3.client("s3")
>>> resp = client.list_objects_v2(Bucket="cryptoexchanges.veraciousdata.io")
>>> [c["Key"] for c in resp["Contents"]]
['coinbase/26-06-19.hdf']
```

The data is written via [HDF5](https://en.wikipedia.org/wiki/HDF) with a single file dedicated to each day. I used the `h5py` Python module, but any HDF5 reader would suffice:

```python
>>> import h5py
>>> f = h5py.File("26-06-19.hdf", "r")
```

The HDF itself is split into groups, one for each set of queries, named by the UTC timestamp the scrape was started:

```python
>>> [k for k in f.keys()[0:5]]
1409:17
1410:17
1411:17
1412:17
1413:17
1414:17
```

Within each group, there's a `bids`, `asks`, and `price-points` subgroup. The `bids` and `asks` groups are the complete order books listed on Coinbase at that time. They were pulled via the [/products/BTC-USD/book?level-3](https://docs.pro.coinbase.com/#get-product-order-book) API call. Each item within `bids` and `asks` can be retrieved via the `price` and `size` keys.

```python
>>> f["1409:17"]["asks"][0:5]
array([(12850.26, 0.49446064), (12850.26, 0.5       ),
       (12850.27, 0.27      ), (12850.29, 0.42372307),
       (12850.29, 0.5       )], dtype=[('price', '<f4'), ('size', '<f4')])
>>> f["1409:17"]["asks"][0]["price"]
12850.26
```

You can also use standard `numpy` ndarray slicing syntax. For example, if you wanted to get just the prices of each open ask order:

```python
>>> f["1409:17"]["asks"][:]["price"]
array([1.285026e+04, 1.285026e+04, 1.285027e+04, ..., 1.000000e+09,
       1.000000e+09, 8.385506e+09], dtype=float32)
```

Can we just take a quick second to note that there are ask orders open for $8.38e+09? Those are _definitely_ going to get filled, guy. 

 The `price-points` group has the `buy`, `sell`, and `spot` prices for not only BTC, but ETH, XRP, and LTC as well. The trade volume for these coins is regularly pretty high, and a fair margin higher than the rest of the shitcoins out there, so figured it'd be worth collecting as well:

 ```python
 >>> [k for k in f["1409:17"]["price-points"].keys()]
BTC
ETH
LTC
XRP
>>> f["1409:17"]["price-points"]["ETH"]["buy"]
347.58
```

## Deployment

The data is harvested via a Python3 script using `h5py` (obviously) and `twisted`. Originally, I had written it as a "run once" job, executing once per minute as a k8s cronjob. Unfortunately, what I found was that the k8s scheduler didn't perform as expected. Take a look at the timestamps from the original HDF:

```python
>>> import h5py
>>> with h5py.File("26-06-19.hdf", "r") as f:
...   [k for k in f[0:9]]
<HDF5 group "/0131:15" (3 members)>
<HDF5 group "/0132:22" (3 members)>
<HDF5 group "/0133:29" (3 members)>
<HDF5 group "/0134:36" (3 members)>
<HDF5 group "/0135:46" (3 members)>
<HDF5 group "/0136:53" (3 members)>
<HDF5 group "/0138:00" (3 members)>
<HDF5 group "/0138:17" (3 members)>
<HDF5 group "/0139:24" (3 members)>
<HDF5 group "/0140:31" (3 members)>
```

The script takes several seconds to run, as I included some sleep statements to keep from tripping the rate limits. Accounting for that and container mount time, I can definitely see the total job execution time being around 7 seconds -- the exact amount of time each job is getting offset by. This makes me suspicious that the k8s `cronjob` type actually just sleeps, instead of schedules, jobs. Though for what it's worth, I haven't deep dived the code to see if that's actually what's happening.

Regardless, a k8s cronjob wasn't going to suit my needs. So I used `twisted` as a scheduler within a standard deployment. The (ugly-as-sin) code can be found [here](https://github.com/Atticuss/little-black-box/tree/master/images/exchange_scrape/coinbase), as well as the Dockerfile.