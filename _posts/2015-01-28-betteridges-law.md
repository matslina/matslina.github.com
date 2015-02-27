---
layout: post
title: is betteridge's law of headlines correct?
location: New York
draft: true
---

Betteridge's law of headlines states that *any headline which ends in
a question mark can be answered by the word no*. This "law" is of
course no law -- it is trivial to create a counterexample -- but
should rather be viewed as a tongue-in-cheek remark on how poor
journalism sometimes hide behind dubious headlines.

In this post we'll have a look at just how incorrect Betteridge's law
is. Using the scripts in [this
repository](https://github.com/matslina/betteridgeslaw), we have
crawled 13 news sites for a total of 26000 headlines (2000 per site)
and have tried to manually answer the 766 of them that end in a
question mark.

Caveat lector: the data presented in this post is the product of a
single person trying to answer a large number of questions, many of
which are subjective in their nature, so don't confuse this for
science.

Polar questions
---------------

Some questions just can't meaningfully be answered with "yes" or
"no". The BBC asks [why we value
gold](http://www.bbc.com/news/magazine-25255957) and USA Today asks
[what Facebook could have bought instead of acquiring WhatsApp for $16
billion](http://www.usatoday.com/story/news/nation-now/2014/02/19/facebook-whatsapp-16-billion/5621721/).
We can with confidence rule out both "yes" and "no" as valid answers
for these questions, since the questions are not yes-no questions. A
*polar question*, on the other hand, is a question for which the
answer *is* expected to be "yes" or "no".

<!-- explain the pie chart more clearly. should be clear that it's the 397 sample. -->

![polarity image](/img/betteridge_polarity_pie.png)

Of the 766 headlines ending in question mark, roughly 46% are
non-polar and therefore violate Betteridge's law. They are not yes-no
questions, so the answer is not "no". We conclude that Betteridge's
law is incorrect.

But, we're not done yet. It could be that the remaining 54% all have
the answer "no", which would make the law "mostly true". Is
Betteridge's law mostly true?

Yes / No / Maybe
----------------

While the expected answer to a polar headline is either "yes" or "no",
providing such an answer is not always a simple task.

The BBC asks if [India is the next university
superpower](http://www.bbc.com/news/business-12597815) but doesn't
define what "university superpower" means. It doesn't seem impossible
that India could become that, whatever it is, but we can't really say
if it's more likely to happen than not. Hence, the answer will have to
be "maybe".

The Daily Mail asks if [Cathy Freeman's missing bodysuit has been
found](http://www.dailymail.co.uk/sport/othersports/article-2885773/Cathy-Freeman-s-missing-bodysuit-Sydney-2000-Olympic-Games-anonymously-handed-original.html). The
article suggests that this question should be answered in the near
future, but as of this writing we can't find any clear information on
whether or not it has been. The answer remains "maybe".

For the purpose of determining whether or not Betteridge's law is
*mostly* true, we don't need answers for all headlines, just for the
majority of them. We waded through the polar 56% of our total 766
headlines and answered as many of them as we could.

![polarity image](/img/betteridge_answer_pie.png)

With 46% non-polar and 20% answered "yes", at least two thirds of our
headline sample violates Betteridge's law. It cannot be "mostly
correct" either.

Conclusion
==========

We already knew that Betteridge's law was incorrect. Now we know it
even more. Additionally, we're going to go out on a limb here and
guesstimate that the remaining "maybe" answers can, given enough time
and effort, be turned into "yes" or "no" answers, and that these will
be distributed similarly to the 20:17 ratio of the fully
answered. That would put the total ratio of "no" at 25%.

It appears that *roughly a quarter of all headlines which ends in
question mark can be answered by the word no*.
