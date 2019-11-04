---
layout: post
title:  "Extracting TV shows and movie names from SEC filings of Netflix"
date:   2019-11-01 16:32:39 +0200
categories: nlp, python, html parsing
---

Like all publically listed companies on the US Nasdaq stock exchange, Netflix files quarterly returns with the U.S. Securities and Exchange Commission (SEC). This includes a [letter to the shareholders](https://www.sec.gov/Archives/edgar/data/1065280/000106528019000366/ex991q319.htm) that details financial performance of Netflix such as cashflow, subscriber growth, revenues etc. It also contains a section called "Content" where Netflix lists their recent and upcoming shows. Here's a snippet from their latest filing:

> We’re expanding our non-English language original offerings because they continue to help grow our penetration in international markets. In Q3, Season 3 of La Casa de Papel (aka Money Heist) became the most watched show on Netflix across our non-English language territories with 44m households watching the new season in the first four weeks of release. Sintonia, our latest Brazilian original, was the second most watched inaugural season in Brazil. The Naked Director broke out as the biggest title launch for us in Japan and was also highly successful throughout Asia. Similarly, in India, we debuted the second season of Sacred Games, our most watched show in India.

While browsing through their past filings, I realised that Netflix now creates a lot of new content in English as well as in other languages[^1]. I also seem to be stuck in some sort of recommender system local maxima bubble, so very little of this content ever gets recommended to me. I wondered if programmatically extracting names of TV shows and movies that get a mention in the Content section of Netflix's letter to the shareholders could be one way to discover new content.

## Extracting the "Content" section
While references to shows are scattered throughout Netflix's letter to the shareholders, most of them are concentrated in the Content section. I wanted to parse the HTML and extract the Content section for further processing. Unfortunately, the markup of these SEC filings tends to be quite poor with no hierarchy or semantic considerations. It's just full of a bunch of divs nested inside divs and font tags. For example:

```html
<div style="line-height:120%;font-size:11pt;">
  <font style="font-family:Calibri,sans-serif;font-size:11pt;"><br/></font>
</div>
```

(formatted for clarity - there are hardly any newlines in the [original](https://www.sec.gov/Archives/edgar/data/1065280/000106528019000366/ex991q319.htm))

Fortunately, the `requests_html` library was quite up to the task and with a little tinkering I was able to get the job done:

```python
def extract_content_section(filepath):
    content_html = []
    content_txt = []
    with open(filepath, 'r') as fp:
        doc = fp.read()
        html = HTML(html=doc)
        is_content_div = False
        for elem in html.find('div'):
            if elem.text == 'Content':
                is_content_div = True
                continue

            if is_content_div:
                style = elem.attrs.get('style', '')
                if 'font-size:18pt' in style:
                    is_content_div = False
                    break

                if 'line-height:120%' and 'font-size:11pt' in style\
                   and 'padding-bottom:6px' not in style:
                    content_html.append(elem.html)
                    content_txt.append(elem.text.strip())

    return ("\n".join(content_html),
            "\n".join(content_txt).replace("\n"," "))
```

The `extract_content_section` function returns a tuple of two strings - `content_html` with all the html tags intact and `content_txt` which is plain text. Why both? It should become clear in a moment.

## The NLP approach

Natural Language Processing (NLP), like many fields, has been seeing a lot of breakthroughs driven by deep learning. Extracting TV shows and movie names seemed like a good problem for Named-entity Recognition using spaCy. I installed spaCy along with the pre-trained `en_core_web_lg` model (a whopping 789 MB as of v2.2.0) and wrote a quick function similar to one in spaCy [documentation](https://spacy.io/usage/linguistic-features#named-entities).

```python
def get_movies_nlp(doc):
    import spacy

    nlp = spacy.load("en_core_web_lg")
    doc = nlp(doc)

    content = set()
    for ent in doc.ents:
        print(ent.text, ent.start_char, ent.end_char, ent.label_)
        content.add(ent.text)

    return content
```

Calling this function with the following text:

> Sintonia, our latest Brazilian original, was the second most watched inaugural season in Brazil. The Naked Director broke out as the biggest title launch for us in Japan and was also highly successful throughout Asia. Similarly, in India, we debuted the second season of Sacred Games, our most watched show in India.

Gives:

```
Sintonia 0 8 ORG
Brazilian 21 30 NORP
second 49 55 ORDINAL
Brazil 89 95 GPE
The Naked Director 97 115 PERSON
Japan 164 169 GPE
Asia 212 216 LOC
India 232 237 GPE
the second season 250 267 DATE
India 310 315 GPE
```

### Challenges

There are several challenges with the named-entity approach.

1. It completely missed Sacred Games.
2. It's difficult to decide which entities to consider as title of shows. For example, if we are to pick only ORG (Sintonai) and PERSON (The Naked Director) entities, what happens when a show named 'Brazil' comes along?

### Other considerations

The corpus used for training the `en_core_web_lg` model is probably not a good fit for this problem space. Retraining on a new, more relevant corpus could be a way forward. Not only would this require annotated data, I am not even sure *how much* data would be required to achieve reasonable accuracy.

Another way forward could be to use [syntactic dependency parsing](https://spacy.io/usage/linguistic-features#dependency-parse) and then try and infer the names of shows from the structure of the sentence.

Both approaches sound time consuming and good outcomes seem far from certain. I must qualify this by saying that I am not a practicing NLP professional, so if I am wrong or missing another approach, I'd be happy to be corrected (my email is at the bottom of this page)!

## The HTML parsing approach

While looking at the HTML source of the Content section, I realised that all show names were always wrapped in a `<font>` and carried a `font-style:italic` attribute:

```html
<font style="font-family:Calibri,sans-serif;font-size:11pt;color:#1155cc;font-style:italic;text-decoration:underline;">
  Stranger Things
</font>
```

The use of `color:#1155cc` and `text-decoration:underline` attributes to make the text look like a link (cringeworthy, isn't it) is inconsistent[^2]. Fortunately, the use of `font-style:italic` seems very consistent across several past quarters. I exploited this to extract show names from `content_html` that the `extract_content_section` function above returns:

```python
def get_movies_html(doc):
    html = HTML(html=doc)

    # deduplicate movie names as they could be repeated across the text
    content = set()

    for elem in html.find('font[style*="italic"]'):
        title = elem.text.strip()
        if title and title != ".":
            if ',' in title:
                names = title.split(',')
                for name in names:
                    content.add(name.strip())
            else:
                content.add(title.strip())

    return content
```

Calling this function with the following HTML snippet:

```html
<font style="font-family:Calibri,sans-serif;font-size:11pt;color:#1155cc;font-style:italic;text-decoration:underline;">Sintonia</font><font style="font-family:Calibri,sans-serif;font-size:11pt;"><sup style="vertical-align:top;line-height:120%;font-size:pt">4</sup></font><font style="font-family:Calibri,sans-serif;font-size:11pt;">, our latest Brazilian original, was the second most watched inaugural season in Brazil. </font>
<font style="font-family:Calibri,sans-serif;font-size:11pt;color:#1155cc;font-style:italic;text-decoration:underline;">The Naked Director</font><font style="font-family:Calibri,sans-serif;font-size:11pt;"><sup style="vertical-align:top;line-height:120%;font-size:pt">5</sup></font><font style="font-family:Calibri,sans-serif;font-size:11pt;"> broke out as the biggest title launch for us in Japan and was also highly successful throughout Asia.
Similarly, in India, we debuted the second season of </font><font style="font-family:Calibri,sans-serif;font-size:11pt;color:#1155cc;font-style:italic;text-decoration:underline;">Sacred Games</font><font style="font-family:Calibri,sans-serif;font-size:11pt;"><sup style="vertical-align:top;line-height:120%;font-size:pt">6</sup></font><font style="font-family:Calibri,sans-serif;font-size:11pt;">, our most watched show in India.
To date, we have globally released 100 seasons of local language, original scripted series from 17 countries and have plans for over 130 more in 2020. We also plan to expand our investment in local language original films and unscripted series.</font>
```

Gives:

```
{'Sacred Games', 'The Naked Director', 'Sintonia'}
```

Which indeed looks more realistic. I ran this on all the letters to investors filed this year by Netflix and generated a [webpage](https://www.deepakg.com/_assets/netflix_shows.html)[^3]. If you are browsing on a Desktop browser, you can hover over an individual show to see a tooltip with some additional context about it.

### Challenges

While currently the HTML parsing approach works reasonably well, I am afraid it is too tightly coupled with the current formatting conventions of the page. If they were to change the attribute or tags for highlighting shows, or worse, stop highlighting shows altogether, this approach will stop working and might need something NLP based.

## Conclusion

Though extremely narrow in scope, this exercise has given me a new-found appreciation for the progress NLP has made and the work that still remains to be done.

[Source Code](https://github.com/deepakg/netflix_shows)


---
[^1]: As of this writing, their international subscriber count is roughly 1.6x more than their US subscriber count. And while their international margins are lower, thanks to the bigger user base, they've been earning more revenue internationally since Q4 2018. So while creation of non-English content might be motivated by revenue, it might also be driven by regulation (for example, see EU's [recent 30% content quota legislation](https://www.europarl.europa.eu/news/en/press-room/20180925IPR14307/new-rules-for-audiovisual-media-services-approved-by-parliament)).

[^2]: This style is used when they want to link a show name to other content, e.g. a trailer on YouTube. In that sense this is a link but the link URL ends up as a footnote. It's probably done to make it easier to read these reports in print but a YouTube link there is pointless anyway.

[^3]: I intend to go as far back as possible and update this page. Hopefully it lets you discover content you might not otherwise have come across!