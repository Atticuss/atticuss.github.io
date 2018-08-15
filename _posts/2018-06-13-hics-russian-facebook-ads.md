---
layout: post
title: "Exploring Russian-bought Facebook Ads"
date: 2018-06-13 17:23:55 -0700
tags: 
- Russia
- House Intelligence Committee
- Facebook ads
- 2016 election
- Internet Research Agency
- Sentiment analysis
published: false
description: Back in May, the Democrat HIC posted a dump of Facebook ads purchased by the Internet Research Agency -- and where there are data sets, there may also be data trends. 
---

## Intro

Back in May, the Democrat House Intelligence Committee posted a dump of [Facebook ads](https://democrats-intelligence.house.gov/facebook-ads/social-media-advertisements.htm) purchased by Russian's between 2015 and 2017. Specifically, the HIC claims these ads were purchased by the infamous Internet Research Agency. So naturally, I pulled down the several GB worth of zipped data and started poking around. 

Each individual ad was given it's own two page PDF report with a chunk of metadata related to it, such as the date it ran, the groups it targeted and excluded, and the price paid on the first page:

![Page 1](/assets/images/blog/2018-06-13-hics-russian-facebook-ads/example-1.jpeg)

The second page then had a shot of the ad itself:

![Page 2](/assets/images/blog/2018-06-13-hics-russian-facebook-ads/example-2.jpeg)

So, if you're like me, your very first thought is: "how much do you have to hate yourself and everyone else to pick PDFs as the export format?!" Well, it's what we have to work with so lets dig into methodology before looking at the results. If you don't care about Python of the evils of PDFs, feel free to [skip ahead](https://veraciousdata.io/blog/2018/06/13/hics-russian-facebook-ads/#results).

## Methodology

So for those of you who aren't aware, PDF is the devil's format. It is an incredible pain to reliably parse, but I decided to take a first stab using the [pdfminer.six](https://github.com/pdfminer/pdfminer.six) module for Python. Using one of the bundled scripts as a template, I built my first implementation relatively quickly:

```
out = io.BytesIO()
with open(path, 'rb') as f:
  pdfminer.high_level.extract_text_to_fp(f, out)

out.seek(0)
raw_data = ' '.join([l.strip() for l in out.read().decode('utf-8').split('\n') if len(l) > 0])
raw_data = raw_data.split(fname)[0]
```

This actually ended up working and I was able to pull all the text out of the first page. The final line was a super hacky way to truncate data after the final line of meta-data was displayed (e.g. after the Ad Creation Date line). This was because, for whatever reason, parsing my initial test file had the filename (P(1)000003 in the above example) precede the "Redaction Completed at..." line. With this in place, just needed to setup a bit of parsing to turn a long string into a dictionary.

However, I hit a snag. The order in which the text was parsed was not deterministic. In most cases, text was parsed line-by-line, so the example ad would be parsed to:

```
Ad ID 650 Ad Text CONGRATS TO ALL THE GAY COUPLES!! After a long fight, you have your much deserved rights! Same-sex married couples... Ad Landing Page https://facebook.com...
```

But every so often, text would get parsed as two individual columns, resulting in the string:

```
Ad ID Ad Text Ad Landing Page Ad Targeting Ad Impression Ad Clicks Ad Spend Ad Creation Date 650 CONGRATS TO ALL THE GAY COUPLES!! After a long fight, you have your much deserved rights! Same-sex married couples... https://facebook.com...
```

This immediately broke the initial parsing logic I had written. Turns out that `pdfminer` has a few settings you can tweak to help with this. Specifically, you can alter the distance between two words in order for them to be considered in the same line, or the distance between letters for them to be considered part of the same word. However, even after several hours tinkering with these values, I was unable to find a set that would result in all PDFs getting parsed the same. It was at this moment that I realized the extent of the evil of PDFs: it might actually be easier to just convert to an image and run some good old OCR.

## OCR

And lo and behold, it was. Because PDFs suck. I more-or-less ripped the code right out of [this blog post](https://pythontips.com/2016/02/25/ocr-on-pdf-files-using-python/) using [wand](http://docs.wand-py.org/en/0.4.4/) (a Python wrapper for ImageMagick), [PIL](http://www.pythonware.com/products/pil/), and [pyocr](https://gitlab.gnome.org/World/OpenPaperwork/pyocr).

```
import io
import pyocr
import pyocr.builders

from wand.image import Image
from PIL import Image as PI

tool = pyocr.get_available_tools()[0]
lang = tool.get_available_languages()[1]

with Image(filename=path, resolution=300) as image_pdf:
  image_jpeg = image_pdf.convert('jpeg')

req_image = []
for img in image_jpeg.sequence:
  img_page = Image(image=img)
  req_image.append(img_page.make_blob('jpeg'))

raw_data = []
for img in req_image: 
  txt = tool.image_to_string(
      PI.open(io.BytesIO(img)),
      lang=lang,
      builder=pyocr.builders.TextBuilder()
  )
  raw_data.append(txt)

raw_data = ''.join([' '.join([k.strip() for k in j.split('\n')]) for j in raw_data])
```

Don't you love that last one-liner?? And by love, I obviously mean loathe with every square inch of sensibility within you. It's quite terrible. I've never been more proud. 

This worked perfectly, but one more snag was to be had: after every 9 PDFs, a `CacheError` exception would get thrown complaining about a lack of resources. Obviously, my first though was that one PDF in particular was causing issues, but no matter which files I threw at it, an exception would always get raised processing number nine.

After fighting with `pdfminer` for several hours, I had lost all patience in attempting to figure out what resource wasn't being properly closed and GCed. So I took the quick way out: I wrapped the PDF-to-JPEG and OCR functionality into its own function and called it via the `multiprocessing` module. I had hoped that spinning up a whole separate proc for this piece would force a resource de-allocation on whatever was hanging around once the proc was terminated. It would seem my assumptions were correct, as I managed to parse all 3500 or so ads after I refactored the code.

## Sentiment Analysis

My initial inspiration for doing this in the first place was an excuse to play with [VADER](https://www.aaai.org/ocs/index.php/ICWSM/ICWSM14/paper/view/8109): a sentiment analyzer that [ships with](http://www.nltk.org/howto/sentiment.html) `nltk`. Sweet, naive me thought I'd spend most of my time playing with this piece. Turns out, using VADER with `nltk` is _super_ simple:

```
from nltk import tokenize
from nltk.sentiment.vader import SentimentIntensityAnalyzer

sid = SentimentIntensityAnalyzer()

scores = []
lines_list = tokenize.sent_tokenize(data['Ad Text'])
for line in lines_list:
  ss = sid.polarity_scores(line)
  scores.append(ss)
```

In the snippet above, the `data` object is the dictionary holding the results of post-processed `raw_data` string. The above example would then have the following within `scores` (one for each sentence):

```
[{'neg': 0.0, 'neu': 0.557, 'pos': 0.443, 'compound': 0.6103},
{'neg': 0.266, 'neu': 0.734, 'pos': 0.0, 'compound': -0.4389},
{'neg': 0.0, 'neu': 0.811, 'pos': 0.189, 'compound': 0.8225},
{'neg': 0.0, 'neu': 0.367, 'pos': 0.633, 'compound': 0.784},
{'neg': 0.127, 'neu': 0.873, 'pos': 0.0, 'compound': -0.2584},
{'neg': 0.0, 'neu': 1.0, 'pos': 0.0, 'compound': 0.0},
{'neg': 0.0, 'neu': 0.629, 'pos': 0.371, 'compound': 0.908}]
```

It's interesting to note that despite the name, the compound score is actually calculated independently from the negative, neutral, and positive scores. This [StackOverflow](https://stackoverflow.com/a/40337646) answer gives a solid breakdown, but the tl;dr is that the compound score is computed using the raw valence score and a hyper-parameter `alpha`. 

As these scores are already normalized to `[-1,1]`, my approach was to average all the scores together for each line to create an aggregate score for the entire ad. There's an argument to be made for scaling the weight of an individual score according to sentence length or possibly other metrics; however, I decided against this.

## Post-SA

That results are, sadly, not as interesting as I would of hoped. For example, when performing a sentiment analysis against all the text, I ranked the ads by the most "negative." Doing this, I found a few with a perfect 1.0 score, such as these ads:

![A 1.0 negative VADER score](/assets/images/blog/2018-06-13-hics-russian-facebook-ads/high-neg-1.jpeg)

![A 1.0 negative VADER score](/assets/images/blog/2018-06-13-hics-russian-facebook-ads/high-neg-2.jpeg)

These are obviously not political ads at all. I'm not entirely sure what to make of this given what's been actively reported on the motivations of the IRA. Perhaps the HIC did a poor job properly discovering and attributing ads or maybe the IRA intentionally published completely neutral ads in an attempt to mask their activities. Perhaps I'm just not seeing the political implications of either of these. Regardless, I decided to take a stab at filtering these out via the landing page. The idea is that pages like Memopolis aren't going to have any political affiliation, but the blackmatters.us domain will always host left-leaning content. 

The vast majority of landing pages pointed to a Facebook link. But interestingly, there were almost no links to any news or publication sites. 

| Domain | Count |
| --- | --- |
| facebook.com | 2952 |
| blackmattersus.com | 254 |
| instagram.com | 120 |
| musicfb.info | 107 |
| meetup.com | 15 |
| dudeers.com | 9 |
| represent.com | 9 |
| youtube.com | 5 |
| black4black.info | 4 |
| petitions.whitehouse.gov | 4 |
| hilltendo.com | 3 |
| donotshoot.us | 2 |
| eventbrite.com | 1 |
| bonfirefunds.com | 1 |
| change.org | 1 |
| aljazeera.com | 1 |
| theguardian.com | 1 |
| edition.cnn.com | 1 |

For all the Facebook landing pages, there was a good amount of repeated pages. Here's a partial list ordered sequentially:

| Domain | Count |
| --- | --- |
| /Black-Matters-1579673598947501/ | 245 |
| /blackmattersus.mvmnt/ | 229 |
| /brownunitedfront/ | 204 |
| /WilliamsandKalvin/ | 159 |
| /blacktivists/ | 159 |
| /Memopolis-450474615151098/ | 154 |
| /Blacktivist-128371547505950/ | 132 |
| /Dont-Shoot-1157233400960126/ | 109 |
| /LGBT-United-839497472793277/ | 95 |
| /Woke-Blacks-294234600956431/ | 82 |
| /savethe2A/ | 68 |
| /WilliamsKalvin-788980617892144/ | 63 |
| /Stop-Al-896610653786585/ | 63 |
| /PanAfrootsmove/ | 60 |
| /patriototus | 58 |
| /MuslimAmerica/ | 54 |
| /stoptherefugees/ | 50 |
| /blackmattersus/ | 47 |

Just a quick eyeball of these pages shows that there was a much higher amount of left-leaning pages than right-leaning ones. In fact, there are only three right-leaning pages in that list: "savethe2A", "patriototus", and "stoptherefugees." I went through and found the pages and domains that reliably leaned one direction and tagged them appropriately. 

For those curious: I did the manual classification by writing up a quick Python script that would render an ad on screen and then listen for an arrow key stroke. Left for left, right for right (duh), and up for neutral or non-political. 

I made it about 50 ads into this process before giving up. The vast majority of ads I was coming across didn't neatly fit into either left or right, namely because the ads themselves weren't making political statements. For example, this ad is nothing more than a proclaiming a love for being Mexican:

![I love being Mexican](/assets/images/blog/2018-06-13-hics-russian-facebook-ads/mexican.jpeg)

And another from a black-specific page that has zero political leaning in the post itself:



How do you classify this? One could argue that since racial identity is such a relevant ideology on the left, it should be classified as such, but it doesn't espouse any kind of rhetoric, let alone inflammatory rhetoric that would rile up conservatives and drive the political wedge that much deeper. That said, there were certainly the kind of inflammatory ads I expected, they just made up a small portion of the dump.

**inflammatory ads**

Because of this, I decided to change tactics a bit. Through a combination of keyword searching and manual evaluation, I went through and tagged each ad as either inflammatory or not. I also tagged each ad for the group identity it appealed too -- after all, not every black person is on the left and not every Christian is on the right.

## Results

So for the final numbers, we end up with:

| Political Affiliation | Count |
| --- | --- |
| Left | 0 | 
| Right | 0 |
| None | 0 |

number of ads published by political affiliation on the y-axis and fiscal quarter on the x-axis
sentiment analysis of ads by political affiliation
overall sentiment of ads over each fiscal quarter