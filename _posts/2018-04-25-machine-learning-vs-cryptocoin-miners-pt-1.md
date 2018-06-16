---
published: true
layout: post
categories: blog
title: Machine Learning vs Cryptocoin Miners -- Part 1
author: Jonn Callahan
tags:
  - AWS
  - VPC Flow Logs
  - Cryptocoin mining
  - Machine Learning
  - Intrusion Detection System
description: One of the most common attack-patterns for black hats these days is to pivot from server control to cryptocoin mining. In this post, I'm going to dig into my attempts at a poor man's IDS leveraging ML and AWS VPC Flow Logs.
---

## Intro

With the meteoric rise of cryptocurrencies as an economic entity, morally corrupt folk have begun leveraging them as an easy way to convert popped boxes into money. Specifically, and obviously, this is done by turning compromised machines into cryptocoin miners. Many coins require the use of highly specialized hardware, such as Field-Programmable Gate Arrays (FPGAs) or Application-Specific Integrated Circuits (ASICs), in order for mining to see any kind of realistic return; however, many can be successfully mined using GPUs and even CPUs. Monitoring your AWS environment for malicious insiders or external attackers mining GPU-based coins is fairly straight forward for the vast majority of organizations: set up a CloudWatch event to trigger any time a GPU-enabled EC2 instance is spawned. Most organizations have no legitimate use for GPU-enabled servers, so if one is spun up, it's almost certainly an event that warrants review. This has lead to attackers preferring to leverage coins that can be mined via CPUs. The most popular choice is Monero (XMR), though Verium (VRM) is another possible candidate. 

## Overview

Given how prevalent this threat has become, I decided to look into the feasibility of identifying compromised machines mining cryptocurrencies by monitoring VPC Flow Logs as a sort-of poor man's IDS. I chose to target Virtual Private Cloud (VPC) Flow Logs because one of my main goals was to create a solution that was as non-intrusive as possible. You would likely see much better results leveraging data from a networking sniffer or via an agent installed on every EC2 instance, but this can quickly become time-prohibitive depending on how large and varied an environment is. Additionally, an agent would not help if an attacker managed to compromise the environment itself and spin up EC2 instances without the agent. 

## VPC Flow Logs

For those unfamiliar, VPC Flow Logs are a 5-tuple record of traffic seen by AWS and organized by the Elastic Network Interface (ENI) it was routed to/from. Every 10 minutes, AWS dumps a list of aggregate metrics of all unique connections seen with uniqueness determined by the source IP/port, destination IP/port, and the protocol. The result is a space-delimited list in the following format:

```
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status
```

Most of the fields are self-explanatory, but the [AWS documentation](https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/flow-logs.html#flow-log-records) can be referenced for those that are not.

Note that because uniqueness is sorted by source and destination, logs are stateless and there is a different entry for each direction of a bi-directional connection (e.g. TCP). This requires us to manually rebuild and associated entries that occurred from the same connection. Additionally, because AWS only reports data over a ten-minute window and not per-packet, there are significant challenges to applying statistical analysis techniques against our data.

## Massaging The Log Data

Before digging into statistical analysis of any kind, some work needs to be done to modify the VPC Flow Log entries to be workable. The first thing to do is simply strip out all logs with an `action` of `DENY`. While it may be possible to build a model using traffic generated from a mining client that is unable to reach a pool or wallet, this post is focused around identifying clients that are successfully communicating with a pool. The next step is to find all bidirectional traffic and strip out the rest. This was done by aggregating logs whose source IP:port matched the destination IP:port and vice versa (and if the protocol matched). For example, these two log entries would be part of the same TCP connection:

```
2 178069999999 eni-55aa9999 91.189.89.199 172.12.12.12 3333 45113 6 3 760 1520029322 1520029358 ACCEPT OK
2 178069999999 eni-55aa9999 172.12.12.12 91.189.89.199 45113 3333 6 157 20607 1520029355 1520029358 ACCEPT OK
```

These logs were tagged with a "traffic ID" in order to keep track of unique connections. This ID was in the form of `{src_ip}:{src_port}::{dst_ip}:{dst_port}::{proto}`. I had to ensure that when comparing data across all logs, the source IP was that of the ENI and not the remote machine the ENI was talking to. While it would be possible to leverage the AWS API to pull a list of the IP addresses associated with the ENI we're working with, I wanted to keep analysis as self-contained as possible. So instead, I built a list of every unique IP address seen and ranked them by the number of occurrences. When creating the traffic ID, the IP address with the greatest number of occurrences was assigned as the source. 

From there I pared it down a bit further by only looking at traffic IDs with a protocol of `6` which corresponds to TCP traffic. However, this left me with a possibly incomplete view of traffic streams. This is because while the server in a client-server communication will have a static port, the client typically uses ephemeral ports. If the client and server open up two discrete TCP connections during the same ten-minute window, it's likely the client will use a different port. For example, the following logs are all from the same client connecting to the same web service:

```
2 178069999999 eni-55aa9999 91.189.89.199 172.12.12.12 41535 443 6 12 640 1520029322 1520029358 ACCEPT OK
2 178069999999 eni-55aa9999 172.12.12.12 91.189.89.199 443 41535 6 2060 81989 1520029358 1520029358 ACCEPT OK
2 178069999999 eni-55aa9999 91.189.89.199 172.12.12.12 32416 443 6 13 760 1520029410 1520029473 ACCEPT OK
2 178069999999 eni-55aa9999 172.12.12.12 91.189.89.199 443 32416 6 2174 84253 1520029470 1520029358 ACCEPT OK
```

While these are two discrete TCP streams, the metrics derived from them should be aggregated as they represent the same client interacting with the same service. The important thing to note here is that the IP addresses and the server port (443) are identical while the timestamps do not overlap. Using all this information, we can make some pretty good guesses on which logs are from the same client service interacting with the same server service, even when it's done across multiple TCP streams.

The final step for massaging the traffic logs was to organize groups of logs by collection time frame. AWS stores log data as a JSON object with the log in the `message` attribute. There is also a `timestamp` attribute which indicates when the event was captured. This value was used to organize all logs for each traffic ID.

## And They Said Math Would Never Apply To My Life

So now we've got all our VPC logs more-or-less organized by service-to-service interactions, it's time to start digging into the data itself. My basic strategy going forward was to iteratively filter out logs which don't match against known-mining traffic by using a variety of metrics and statistical analysis methodologies. 

With the data available from Flow Logs, I identified the following metrics:

  * Number of bytes generated by the source and destination
  * Number of packets generated by the source and destination
  * Number of unique source and destination ports
  * Communication duration by source and destination

These metrics were calculated for each unique `timestamp` of a given traffic ID. This resulted in eight dimensions of attributes for each connection, which is far too much to work with. After generating all the metrics for a given traffic ID, the dimensionality was reduced by applying [Principle Component Analysis](https://en.wikipedia.org/wiki/Principal_component_analysis) (PCA). The math behind this (and most other things I'll discuss) is outside the scope of this post -- all you need to know is that it's a technique leveraging [eigenvalues and eigenvectors](https://en.wikipedia.org/wiki/Eigenvalues_and_eigenvectors) to reduce the dimensionality of a dataset while maintaining a significant amount of the variability and variance of the original data. Using this, I was able to reduce my 8-dimensions of metrics down to two. The following is a graph of the normalized, PCA-reduced metrics of some random network traffic:

![Non-Mining Network Traffic PCA](/assets/images/blog/2018-04-25-machine-learning-vs-cryptocoin-miners-pt-1/non_mining_pca.png)

And this graph shows the PCA-reduced metrics of three sets of mining traffic:

![Mining Network Traffic PCA](/assets/images/blog/2018-04-25-machine-learning-vs-cryptocoin-miners-pt-1/mining_pca.png)

Interestingly, the mining traffic varied between EC2 instances far more than I expected, but only in the direction the points seem to follow. The clusters themselves look to have very similar clustering characters in terms of the "evenness" of the distribution, the number of points, where and how many outliers there are, etc. This is especially apparent when comparing against the first graph of some random network traffic. 

## Cluster Analysis Using DBSCAN

I'll touch on the "flow" of the mining data points shortly, but for now, the question becomes: how do we quantify and differentiate between two sets of points? The answer is quite a broad one: cluster analysis. There are several techniques out there for identifying and quantifying a cluster, such as [k-means](https://en.wikipedia.org/wiki/K-means_clustering) and [GMM](https://en.wikipedia.org/wiki/Mixture_model#Gaussian_mixture_model), but I ended up deciding on [DBSCAN](https://en.wikipedia.org/wiki/DBSCAN) for its unique ability to accurately identify oddly shaped clusters and its ability to work without any preconception of the number of clusters. Again, the details of DBSCAN are outside the scope of this post, but the relevant things to know are that it functions by iteratively identifying clusters given two parameters: the minimum number of points (minpts) to consider a region "dense" and a radius (epsilon) for the maximum size of a "dense" region. The following is a graph of the same mining traffic, but with DBSCAN-identified clusters highlighted:

![DBSCAN of Mining Traffic](/assets/images/blog/2018-04-25-machine-learning-vs-cryptocoin-miners-pt-1/mining_dbscan.png)

Note how both graphs include large and small colored markers. The larger markers indicate "core" points, while the smaller ones are "density-reachable." The black dots indicate outliers or noise from any of the three datasets. I brute forced the minpts and epislon values to be as small as possible while still flagging 90% of the data as part of the cluster. Using these values, I iterated through all my non-mining datasets, PCA-reduced them, ran DBSCAN, and looked for datasets which had only a single cluster with greater than 90% of the points included within the cluster. The idea is that this would be a great quick-and-dirty method for finding datasets that clustered similarly to the mining traffic datasets. For example, here are the graphs of three non-mining datasets which could be ruled out via this methodology:

![Zero DBSCAN Clusters](/assets/images/blog/2018-04-25-machine-learning-vs-cryptocoin-miners-pt-1/zero_dbscan.png)

![Eight DBSCAN Clusters](/assets/images/blog/2018-04-25-machine-learning-vs-cryptocoin-miners-pt-1/eight_dbscan.png)

![One DBSCAN Cluster With High Noise](/assets/images/blog/2018-04-25-machine-learning-vs-cryptocoin-miners-pt-1/low_density_dbscan.png)

While the third graph is of a dataset which had a single cluster using our minpts and epsilon values, only 26% of the points were part of the cluster. While this was a fantastic first step, it's hardly the complete solution. Due to the wide spread of points in the pool mining data, there were still plenty of non-mining datasets which made it through this filter. For example, this non-mining data set has a single DBSCAN cluster with over 90% membership:

![DBSCAN False Positive](/assets/images/blog/2018-04-25-machine-learning-vs-cryptocoin-miners-pt-1/false_positive_dbscan.png)

## Quantifying Data "Flow" Using Regression Line Angles

So let's go back to the "direction" or "flow" our mining data takes. The most common way to quantify and describe this feature is via a [least squares regression line](https://en.wikipedia.org/wiki/Least_squares#Least_squares,_regression_analysis_and_statistics). This technique allows you derive a single line from a set of points which best models the values of those points. For a quick visualization, take another look at the same DBSCAN graph of mining data, but with a regression line derived from all the points identified as part of a cluster:

![DBSCAN of Mining Traffic with Regression Lines](/assets/images/blog/2018-04-25-machine-learning-vs-cryptocoin-miners-pt-1/mining_dbscan_regression.png)

What's interesting is that while the regression lines are very different, at a glance, they also look to have inverted slope values. A quick check of each regression line's angle relative to the x-axis seems to confirm this:

| Set | Angle |
| --- | --- |
| Red | 32.7° |
| Green | 27.0° |
| Blue | 34.0° |

If these values seem low (the graph makes them look closer to 45° and the lines themselves perpendicular), take a second look at the axis' -- mind your scales! 

I am not entirely sure what to make of this. Both green and blue mined XMR for a very similar duration against the same Stratum port of the same pool. I checked the IP addresses of the traffic ID and confirmed that both green and blue treat the ENI's public IP as the source for all metric generation. The only difference between the two sets appears to be the size of the EC2 instance used: c4.large vs c4.4xlarge. Perhaps faster hash rates change the traffic generated in a significant way? Regardless, it is a clear indication that I need a much wider and more varied set of mining traffic in order to flesh out actual patterns from the noise.

Even still, I decided to grab the angle of the regression line for each of the non-mining datasets. This let me filter out data such as this, which had a regression line angle of 0.5°:

![Regression Line Angle of Non-Mining Traffic](/assets/images/blog/2018-04-25-machine-learning-vs-cryptocoin-miners-pt-1/regression_line_angle.png)

Again, this strategy may not pan out if the similarity of the regression line angles turns out to be coincidental. That said, we can look at another metric regarding the regression lines that is also somewhat similar across all three mining datasets: the average point deviation from the data and the regression line. This was done by finding the distance between each point from the regression line and averaging.

| Set | Average Point Deviation |
| --- | --- |
| Red | 10.51 |
| Green | 15.62 |
| Blue | 8.11 |

The values appear to vary fairly significantly, but they can still be used to trim out extreme non-mining datasets:

![Regression Line Deviation of Non-Mining Traffic](/assets/images/blog/2018-04-25-machine-learning-vs-cryptocoin-miners-pt-1/regression_line_deviation.png)

This particular set of data had a single DBSCAN cluster, a 99% cluster membership, a regression line angle of 23.6°, but an average point deviation of 70.19. This technique was able to pare down my false positives further, but not completely. For example, the following non-mining dataset had a single DBSCAN cluster, a 99% cluster membership, a regression line angle of 25.7°, and an average point deviation of 13.39:

![Regression Line False Positive](/assets/images/blog/2018-04-25-machine-learning-vs-cryptocoin-miners-pt-1/false_positive.png)

## Mean And Median Of The Nearest-Neighbor

The previous graph of non-mining dataset passes through all our filters so far, but even at a glance, it is quite apparent that it doesn't "look" anything like the mining datasets. This particular dataset appears to have large amounts of sub-clustering, while the points in the mining datasets seem to be fairly evenly distributed across the cluster, albeit with some higher density near the centroid. With this in mind, I set out to find a way to quantify the "evenness" of the distribution of points. So the final filter that I've implemented is to calculate the mean and median nearest-neighbor for every point in a dataset. The thought process behind this was that if the mean and median value for the nearest-neighbor are relatively close, there's a good chance the distribution is much more "even" while a heavy delta between the mean and median implies a [skew](https://en.wikipedia.org/wiki/Skewness#Relationship_of_mean_and_median). Additionally, an exceedingly low median and mean also implies heavy amounts of sub-clustering.

Using the nearest-neighbor approach against the previous graph and one of the mining datasets, you get the following:

| Set | Mean | Median | Delta |
| --- | --- | --- | --- |
| Red | 0.1365 | 0.0940 | 0.0425 |
| Non-mining | 0.0105 | 0.0 | 0.0105 |

While the median and mean deltas are similar, the values themselves are very low. This is not inherently obvious from looking at the graph, but there are actually 212 points. This explains how the nearest-neighbor distance median could be 0 (multiple points laid directly on top of each other). This final filter was the last piece I needed to strip out all datasets that weren't gathered from mining hosts. However, I'll be the first to admit this final technique is not that great. One of the first things I plan to do next is to implement the [Ripley L](https://en.wikipedia.org/wiki/Spatial_descriptive_statistics#Ripley's_K_and_L_functions) function to get a much better sense of a dataset's spatial homogeneity.

## Final Thoughts

At this point, I feel as though I've done as much as I can with the data I have (beyond some improvements listed below). While I didn't have much data to work with, I feel confident I was able to tease out some patterns that hopefully hold true as I collect more data to test against. Part two of this blog will focus heavily on revising what I've talked about here while using these methodologies to build an actual classifier (see my note on SVMs in the section below). I also plan to take a final solution, deploy it within an AWS architecture, and begin training it against live data.

## Notable Areas Of Improvement

Further effort could be put into the kind of metrics being gathered from the log data. One way to improve this approach is take a look at each individual metric and strip out those which have minimal variation between mining and non-mining traffic -- including these metrics does nothing but introduce noise. Additionally, I may be able to reduce my signal-to-noise ratio by decomposing my data into 3-dimensions via PCA instead of two. I'll obviously still have information loss, but there should be less using three dimensions. Alternatively, perhaps I do not need to decompose my data at all. Almost every technique used relies on point distances and the equation for calculating the [Euclidian distance](https://en.wikipedia.org/wiki/Euclidean_distance#n_dimensions) of two points is independent of the number of dimensions the points reside in.

I made an assumption by requiring clusters to have 90%+ membership rate; I did not have a good reason for generating the minimal values for the mining datasets against a 90%+ membership rate. As much of the work done here is built on top of the results of DBSCAN-identified clusters, I should definitely spend more time revising this piece. Instead of an arbitrary 90%, perhaps I should manually identify which points "look" to be part of the cluster and not just noise, and then minimize the minpts and epsilon values so all of them are included. 

The least squares regression line is built by minimizing the vertical distance between all points and the derived line, but when I calculated the average point deviation, I used the perpendicular distance. A possible improvement would be to use the same measure of distance for both.

Another technique I plan to look at next is the applicability of a [Support Vector Machine](https://en.wikipedia.org/wiki/Support_vector_machine). I had initially skipped looking into this as I don't believe it's appropriate for classifying sets of data under a single label (though I admittedly only took a cursory look at it). However, there's a good chance this problem could be solved by feeding all my "meta-metrics" (regression line angle, DBSCAN clusters, spatial homogeneity, etc.) into an SVM and solving for a hyperplane which divides mining and non-mining data points.

Finally, I had fairly minimal mining traffic to work with. Unfortunately, this project did not occur to me when doing my previous research on [cloud mining and EC2 CPU degradation](https://nvisium.com/blog/2018/02/12/ec2-cpu-degradation.html) and so I do not have any logs from that. Instead, I stood up a few instances of varying sizes pool mining XMR and pool/solo mining VRM. There were significant differences in the metric patterns when looking at pool mining vs solo mining, but I spent the vast majority of my time (and the entirety of this post) looking at pool mining. One of the next steps I need to take is to generate far more mining traffic to compare against each other to look for commonalities while also revisiting solo mining traffic.

All in all, I believe this is a good first stab at possible techniques that could be used to build a cryptocoin mining IDS. There's still a lot of work to do, things to improve and implement, but I've had good enough results so far that this path seems promising.

## Tooling + Learning

All of my graphs were generated via [matplotlib](https://matplotlib.org/) and the vast majority of the math itself was handled via [numpy](http://www.numpy.org/) and [sklearn](http://scikit-learn.org/stable/). If you're interested in learning ML on your own time, I'd highly recommend [Algorithms of the Intelligent Web](https://www.manning.com/books/algorithms-of-the-intelligent-web-second-edition). Great intro to ML by pairing high-level concepts with Python code. I'll likely include code alongside part two when I refine and actually start piecing these techniques together to deploy within a live environment. 

## Cries For Help

While I was able to glean some patterns from the data I had, I am still woefully lacking in the test data department. I've had one acquaintance reach out and offer to share some flow logs with me so I could perform this analysis, but I only had around 7,000 separate bidirectional streams of data. If you would like to help enable further research into this topic, which will hopefully result in some open source tooling, please reach out to me at jonn.callahan@nvisium.com -- all logs are pulled and stored within unique, highly-controlled S3 buckets. Any publicly released write-ups will have sensitive info anonymized just like all examples in this post have been.

If you have any criticisms or critiques of the methodologies here, please reach out. Any feedback would be greatly appreciated.