---
published: true
layout: post
categories: blog
title: Scraping Reddit Sentiment Data
author: Jonn Callahan
tags:
  - kubernetes
  - reddit
  - machine learning
  - sentiment analysis
  - bitcoin
description: "I started scraping sentiment data from /r/bitcoin and have made it publicly available. Possibly other sources to be added later. Each days worth of data is written via HDF5 and backed up to a publicly readable S3 bucket. Don't make me regret it."
---

In my continued attempt to collect massive amounts of useless data, I've started scraping [/r/bitcoin](https://reddit.com/r/bitcoin) posts and running them through the VADER sentiment analyzer. Filenames represent a days worth of data in the format of `year-month-day.hdf`. The bucket can be found here:

[https://cryptoexchanges.veraciousdata.io/](https://cryptoexchanges.veraciousdata.io/)

## How to Use

The bucket itself is public, so any set of creds can list and pull objects. Object keys are prefixed with the source I'm pulling data from. For this dataset, use `reddit` as the key prefix.

```python
>>> import boto3
>>> client = boto3.client("s3")
>>> resp = client.list_objects_v2(Bucket="cryptoexchanges.veraciousdata.io", Prefix="reddit/")
>>> [c["Key"] for c in resp["Contents"]]
['reddit/20-03-31.hdf']
```

The data is written via [HDF5](https://en.wikipedia.org/wiki/HDF) with a single file dedicated to each day. I used the `h5py` Python module, but any HDF5 reader would suffice:

```python
>>> import h5py
>>> f = h5py.File("20-03-31.hdf", "r")
```

The HDF itself is split into groups, one for each set of queries, named by the UTC timestamp the scrape was started:

```python
>>> [k for k in f.keys()]
['1348:15', '1418:15', '1448:15', '1518:15']
```

Within each group, there's a `comments` and `submissions` subgroup. As of now, the script is pulling down 100 submissions at a time using the "hot" reddit filter. Several pieces of metadata are stored, including the sentiment scores, publish data, vote scores, etc. Raw text itself is not stored, in an effort to reduce the size of the dataset. The exact data structure can be dynamically referenced via the `dtype` attribute:

```python
>>> sub = f["1348:15"]["submissions"][0]
>>> sub.dtype
dtype([('submission_id', 'S10'), ('author', 'S50'), ('created', '<f16'), ('score', [('total', '<i2'), ('upvotes', '<i2'), ('downvotes', '<i2')]), ('title_sentiment', [('pos', '<f16'), ('neu', '<f16'), ('neg', '<f16'), ('compound', '<f16')]), ('text_sentiment', [('pos', '<f16'), ('neu', '<f16'), ('neg', '<f16'), ('compound', '<f16')])])
>>> sub["title_sentiment"]["neu"]
1.0
>>> sub
(b'fbka5j', b'neonzzzzz', 1.58304418e+09, (109, 109, 0), (0., 1., 0., 0.), (0., 0., 0., 0.))
```

It should be noted that sentiment scores are always recorded in the data set, even if a submission is a link and not a self-post. If a sentiment score is a four member tuple with all values of 0.0, that means no sentiment analysis was run.

All comments are scraped and stored within the `comments` subgroup. While the storage design does not preserve the comment forest nor the parent submission, it can be rebuilt by using the `comment_id`, `parent_comment_id`, and `submission_id` fields.

```python
>>> comment = f["1348:15"]["comments"][0]
>>> comment.dtype
dtype([('comment_id', 'S10'), ('parent_comment_id', 'S10'), ('submission_id', 'S10'), ('author', 'S50'), ('created', '<f16'), ('score', [('total', '<i2'), ('upvotes', '<i2'), ('downvotes', '<i2')]), ('text_sentiment', [('pos', '<f16'), ('neu', '<f16'), ('neg', '<f16'), ('compound', '<f16')])])
>>> comment["score"]["total"]
21
>>> comment
(b'fj5xli7', b't3_fbka5j', b'fbka5j', b'Bitcoin_to_da_Moon', 1.58307588e+09, (21, 21, 0), (0.097, 0.893, 0.011, 0.8762))
```

All pieces of metadata are recalculated at time-of-query. If the same comment has a different sentiment score in a later dataset, it's because the comment was edited. There currently is not field which explicitly stores edit status.

## On the Flat Data Structure

Originally, I had planned to retain the comment forest structure within the data type itself. However, since a post has a variable number of comments, I found it difficult to design a `dtype` that allowed for this. The closest I came was to using the `h5py.vlen_dtype()` wrapper. Unfortunately, while I was able to write data just fine, it removed the ability to reference data by column names.

```python
>>> comment_dtype = np.dtype([
...     ("comment_id", "S10"),
...     ("parent_comment_id", "S10"),
... ])
>>> submission_dtype = np.dtype([
...     ("submission_id", "S10"),
...     ("comments", h5py.vlen_dtype(comment_dtype))
])
>>> data = np.asarray([("123", [("234", "345"), ("456", "567")])], dtype=submission_dtype)
>>> data[0]["submission_id"]
b'123'
>>> data[0]["comments"]
[('234', '345'), ('456', '567')]
>>> data[0]["comments"][0]
('234', '345')
>>> data[0]["comments"][0]["comment_id"]
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: tuple indices must be integers or slices, not str
```

Alternatively, I had considered not using ragged columns and just setting the `comment_dtype` to have an expected number of rows set to the maximum number of comments discovered in a given post. However, going down this route would end up bloating the size of the dataset due to the large number of functionally null rows that would need to be included. For example, if post A had 100 comments, the most of all posts seen, and post B only had 50 comments, the final data set for post B would have 50 rows of descriptive metadata and 50 rows of blank filler values. I decided that since I'm gathering all the info required to rebuild the comment forests, it's easiest to just store things entirely flat.

If you'd like to rebuild the forest, the easiest way is to use the `np.where()` and `np.take()` function. The following is a quick example using a toy dataset:

```python
>>> dtype = np.dtype([
...     ("comment_id", "S10"),
...     ("parent_comment_id", "S10"),
... ])
>>> data = np.asarray([("234","123"), ("345","123"), ("456","345")], dtype=dtype)
>>> hdf = h5py.File("test.hdf", "w")
>>> group = hdf.create_group("test")
>>> dataset = group.create_dataset("comments", data=data)
>>> hdf.close()
>>>
>>> hdf = h5py.File("test.hdf", "r")
>>> indices = np.where(hdf["test"]["comments"]["comment_id"] == b"123")
>>> indices
(array([0, 1]),)
>>> np.take(hdf["test"]["comments"], indices)
array([[(b'234', b'123'), (b'345', b'123')]],
      dtype=[('comment_id', 'S10'), ('parent_comment_id', 'S10')])
```

## Data Type Reference

The following tables list the data available and the key name to reference it by:

#### Submissions

| Key | Description |
| submission_id | The ID of the submission, as reported by `praw`. |
| author | The username who submitted the post. Blank if not available, such as due to deletion. |
| created | Unix timestamp of when the post was submitted. |
| score | A three-item tuple housing the `total`, `upvotes`, and `downvotes` score for a post as sub-keys. |
| title_sentiment | The VADER sentiment scores for the title. |
| text_sentiment | The VADER sentiment score for the post text, if it's a self-post and not a link. |

#### Comments

| Key | Description |
| comment_id | The ID of the comment, as reported by `praw`. |
| parent_comment_id | The ID of the parent comment, as reported by `praw`. |
| submission_ID | The ID of the parent submission, as reported by `praw`. |
| author | The username who submitted the comment. Blank if not available, such as due to deletion. |
| created | Unix timestamp of when the comment was submitted. |
| score | A three-item tuple housing the `total`, `upvotes`, and `downvotes` score for a post as sub-keys. |
| text_sentiment |  The VADER sentiment score for the comment text. |

## Deployment

The data is harvested via a Python3 script using `h5py` (obviously), `twisted` for scheduling, and `praw` for querying the reddit API. The `vaderSentiment` module was used for performing the sentiment analysis itself:

https://github.com/cjhutto/vaderSentiment

The (ugly-as-sin) code can be found [here](https://github.com/Atticuss/little-black-box/tree/master/images/sentiment_scrape/reddit), as well as the Dockerfile.
