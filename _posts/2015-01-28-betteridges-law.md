---
layout: post
title: is betteridge's law of headlines correct?
location: New York
draft: true
---

Betteridge's law of headlines states that "*[a]ny headline which ends
in a question mark can be answered by the word no*". This "law" is of
course no law - it is trivial to create a counterexample - but rather
a tongue-in-cheek comment on poor journalism hiding behind dubious
headlines.

<!-- counterexample feels iffy. do em instead of hyphen. or parenthesis? -->

In this post we'll have a look at just how incorrect Betteridge's law
is. Using the scripts in [this
repository](https://github.com/matslina/betteridgeslaw), we have
crawled 13 news sites for a total of 26000 headlines (2000 per site)
and have tried to manually answer the 397 of them that ends in a
question mark. Note that the data is the product of a single person
doing their best to answer a ton of questions, so caveat lector and
don't confuse this for science.

Polar questions
---------------

First of all, some questions just can't meaningfully be answered with
"yes" or "no.  For instance, the answer to "what is brainfuck?" could
be "an interesting programming language", but certainly not "yes" and
certainly not "no". A *polar question*, on the other hand, is a
question for which the answer *is* expected to be "yes" or "no".

<!-- perhaps give an example of a real non-polarheadline? -->
<!-- explain the pie chart more clearly. should be clear that it's the 397 sample. -->

![polarity image](/img/betteridge_polarity_pie.png)

By virtue of their non-polarity, 45% of our dubious headlines violate
Betteridge's law: they are not yes-no questions, so the answer cannot
be "no". The remaining 55% could of course all have the answer "no",
which would make the law "mostly true", but more on that later. Let's
have a look at a breakdown of polarity by source.

![polarity image](/img/betteridge_polarity_stack.png)

No news site really appears to stand out when it comes to the ratio of
polar to non-polar questions but there are clear differences in the
total number of headlines ending in question mark. Then again, our
crawler is not all that polished, so there may be some bias introduced
by how the headlines were collected.

<!-- this paragraph should be split. first cover polarity ratio. then cover ratio of qmark hlines per source. -->

Yes / No / Maybe
================

While the expected answer to a polar question is "yes" or "no", a
large chunk of those in our sample are pretty hard to answer. For
instance, the BBC asks if [India is the next university
superpower](http://www.bbc.com/news/business-12597815) but doesn't
really define what "university superpower" means. It doesn't seem
impossible that India would be that, whatever it is, but we can't
really say if it's more likely than not. Hence, the answer will have
to be "maybe".

Similarly, on the 23rd of December, 2014, the Daily Mail asked if
[Cathy Freeman's missing bodysuit has been
found](http://www.dailymail.co.uk/sport/othersports/article-2885773/Cathy-Freeman-s-missing-bodysuit-Sydney-2000-Olympic-Games-anonymously-handed-original.html).
While we will probably have an answer in the near future, a 4 minute
session of web searching suggests that this has not been verified as
of this writing. For now, our answer will have to be "maybe".

![polarity image](/img/betteridge_answer_pie.png)

So with the bulk of the polar questions answered, we're looking at
roughly a 50-50 split between "yes" and "no". For completness sake,
we'll also have a look at the breakdown of answers per source. Do keep
in mind that the sample size is fairly small for some sources,
especially Reuters and NY Daily News, so let's not jump to too many
conclusions based on this data.


<!-- this part feels a bit thin. perhaps mention that iterating on and researching maybes seem to maintain 50% split. guesstimate that it approaches 50-50. -->


![polarity image](/img/betteridge_answer_stack.png)

<!-- relevant here is that most are still close to 50-50, in spite of publications presumably being biased to/from my frame of mind. does that suggest that 50-50 split is correct? -->

<!-- perhaps high ratio of 'maybe' is an indication of weaslyness? -->


Conclusion
==========

<!-- possibly chop this up as per above comment -->

We already knew that Betteridge's law was incorrect. Now we know it
even more. Additionally, we're going to go out on a limb here and
guesstimate that most of the "maybe" answers can, given enough time
and effort, be turned into "yes" or "no" answers, and that these will
be distributed roughly 50-50 as well. We conclude that "*roughly a
quarter of all headlines which ends in a question mark can be answered
by the word no.*"

<!-- at some point I need to mention that I'm not publishing the raw headlines and their answers coz copyright. but encourage others to redo the experiment. -->