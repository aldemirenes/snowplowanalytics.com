---
layout: post
title: Towards high-fidelity web analytics - introducing Snowplow's innovative new event validation capabilities
title-short: Snowplow event validation
tags: [snowplow, data, event, validation]
author: Alex
category: Inside the Plow
permalink: /blog/2013/04/10/snowplow-event-validation
---

A key goal of the Snowplow project is enabling **high-fidelity analytics** for businesses running Snowplow.

What do we mean by high-fidelity analytics? Simply put, high-fidelity analytics means Snowplow faithfully recording _all_ customer events in a rich, granular, non-lossy and unopinionated way.

This data is incredibly valuable: it enables companies to better understand their customers and develop and tailor products and services to them. Ensuring that the data is high fidelity is essential to ensuring that any operational and strategic decision making that's made on the basis of that data is sound. Guaranteeing data fidelity is not a sexy topic. But it's an important one.

Surprisingly, ensuring your data is high fidelity is **not** something that is enforced by other analytics products.

![high-fidelity] [high-fidelity]

Why is Snowplow so unusual in aiming for high-fidelity analytics? Most often, analytics vendors sacrifice the goal of high-fidelity data at the altar of these three compromises:

1. **Premature aggregation** - when the data store gets too large, or the reports take too long to generate, it's tempting to perform the  aggregation and roll-up of the raw event data earlier, sometimes even at the point of collection. Of course this offers a huge potential performance boost to the tool, but at the cost of a huge degree of customer data fidelity
2. **Ignoring bad news** - the nature of event data means that often incomplete, corrupted or plain wrong data is sent in to the analytics tool by the event trackers. Handling bad event data is complicated (let's go shopping!). Instead of dealing with the complexity, most analytics packages just throw the bad data away silently; this is why tag audit companies like [ObservePoint] [observepoint] exist
3. **Being over-opinionated** - customer analytics is full of challenging questions which need answering before you can analyse the data: do I track users by their first-party cookie, third-party cookie, business ID and/or IP address? Do I use the server clock, or the user's clock to log the event time? When does a user session start and end? Because these questions can be difficult to answer, most analytics tools don't ask them: instead they take an opinionated view of the "right answer" and silently enforce that view through their event collection, storage and analysis. By the time users realize that the logic enforced is one that does not work for their business, they are already tied to that vendor and the imperfect data set they have created with that vendor to date.

To deliver on the goal of high-fidelity analytics, then, we're trying to steer Snowplow around these three common pitfalls as best we can.

We have talked in detail on our website and wiki about avoiding pitfall #1, Premature aggregation. In short: we do **no** aggregation - Snowplow users have access to granular, event level data, so that they can work out how best they should aggregate it for each type of analysis they wish to perform.

We will blog more about our ideas to combat #3, Being over-opinionated, in the future.

For the rest of this blog post, though, we will look at our solution to pitfall #2, Ignoring bad news: namely, **event validation**.

<!--more-->

Our new Scalding-based event enrichment process (introduced in [our last blog post] [snowplow-080]) introduces the concept of **event validation**.

Instead of "ignoring bad news", the Snowplow enrichment engine now validates that every logged event matches the format that we expect for Snowplow events - be they page views, ecommerce transactions, custom structured events or some other type of event. Events which do not match this format are stored in a new "Bad Rows" bucket in Amazon S3, along with the specific data validations which the event failed.

By way of example, here are a couple of [custom structured events] [struct-events] generated by a ecommerce site running Snowplow; both of these events failed the new validation step in our Scalding ETL process. You will note that the bad rows are logged to the S3 bucket in JSON format - we have "pretty printed" the rows to make them easier to read:

{% highlight json %}
{
  "line": "2012-11-14\t11:53:07\tDUB2\t3707\t92.237.59.86\tGET\td10wr4jwvp55f9.cloudfront.net\t\/ice.png\t200\thttps:\/\/www.psychicbazaar.com\/shop\/checkout\/?token=EC-6H7658847D893744L\tMozilla\/5.0%20(Windows%20NT%206.0)%20AppleWebKit\/537.11%20(KHTML,%20like%20Gecko)%20Chrome\/23.0.1271.64%20Safari\/537.11\tev_ca=ecomm&ev_ac=checkout&ev_la=id_city&ev_pr=SUCCESS&ev_va=Liverpool&tid=404245&uid=4434aa64ebbefad6&vid=1&lang=en-US&refr=https%253A%252F%252Fwww.paypal.com%252Fuk%252Fcgi-bin%252Fwebscr%253Fcmd%253D_flow%2526SESSION%345DiuJgdNO9t8v06miTqv5EHhhGukkGNH3dfRqrKhe0i-UM9FCbVNg26G10sRC%2526dispatch%253D50a222a57771920b6a3d7b606239e4d529b525e0b7e69bf0224adecfb0124e9b61f737ba21b0819882a9058c69cd92dcdac469a145272506&f_pdf=1&f_qt=0&f_realp=0&f_wma=1&f_dir=1&f_fla=1&f_java=1&f_gears=0&f_ag=1&res=1920x1080&cookie=1&url=https%253A%252F%252Fwww.psychicbazaar.com%252Fshop%252Fcheckout%252F%253Ftoken%253DEC-6H7658847D893744L\t-\tHit\tAN6xpNsbS0JS05bqjmnbJdZDkl-cVkTPQsAJDlIOgAIG4hcPTTlMFA==",
  "errors": [
    "Field [ev_va]: cannot convert [Liverpool] to Float"
  ]
}
{
  "line": "2012-11-14\t11:53:13\tDUB2\t3707\t92.237.59.86\tGET\td10wr4jwvp55f9.cloudfront.net\t\/ice.png\t200\thttps:\/\/www.psychicbazaar.com\/shop\/checkout\/?token=EC-6H7658847D893744L\tMozilla\/5.0%20(Windows%20NT%206.0)%20AppleWebKit\/537.11%20(KHTML,%20like%20Gecko)%20Chrome\/23.0.1271.64%20Safari\/537.11\tev_ca=ecomm&ev_ac=checkout&ev_la=id_state&ev_pr=SUCCESS&ev_va=Merseyside&tid=462879&uid=4434aa64ebbefad6&vid=1&lang=en-US&refr=https%253A%252F%252Fwww.paypal.com%252Fuk%252Fcgi-bin%252Fwebscr%253Fcmd%253D_flow%2526SESSION%345DiuJgdNO9t8v06miTqv5EHhhGukkGNH3dfRqrKhe0i-UM9FCbVNg26G10sRC%2526dispatch%253D50a222a57771920b6a3d7b606239e4d529b525e0b7e69bf0224adecfb0124e9b61f737ba21b0819882a9058c69cd92dcdac469a145272506&f_pdf=1&f_qt=0&f_realp=0&f_wma=1&f_dir=1&f_fla=1&f_java=1&f_gears=0&f_ag=1&res=1920x1080&cookie=1&url=https%253A%252F%252Fwww.psychicbazaar.com%252Fshop%252Fcheckout%252F%253Ftoken%253DEC-6H7658847D893744L\t-\tHit\tXbvEfkx7BvngWyY23OLDvyFi8mXe2E_nhBaJwkzCG3aNxUng1jz4hQ==",
  "errors": [
    "Field [ev_va]: cannot convert [Merseyside] to Float"
  ]
}
{% endhighlight %}

These validation errors occurred because the ecommerce site incorrectly tried to log customer address information in the `value` field of a custom structured event; the `value` field only supports numeric values (and is stored in Redshift in a float field). When we saw these validation errors, we notified the site and they corrected their Google Tag Manager implementation.

Currently these bad rows are simply stored for inspection in the Bad Rows bucket in S3, while Snowplow carries on with the raw event processing. This lets the Snowplow user tackle the tagging/data quality issues offline, without disrupting the loading of all their high-fidelity, now-validated event data into Redshift. It leaves open the possibility that the user can fix and reprocess the bad rows.

In the future we could look into ways of sending alerts when bad rows are generated, or even look into ways of automatically fixing bad rows and submitting them for re-processing.

This is straight forward stuff - but compare it with the approach taken by other web analytics vendors. If a Google Analytics user sends incorrectly configured data into GA, for example, one of two things happens:

1. GA silently ignores the data
2. GA accommodates the data, so that it corrupts reports produced in GA

For the GA user, spotting the error is impossible in either case. Not only has a data point been lost, but potentially an erroneous data point has been introduced, one that will be very hard to debug given that users can never inspect the underlying data.

This becomes more of a problem as we move to a Unviersal Analytics world: one in which companies feed GA with **all** their customer event data from a variety of systems. Ensuring that the system is fed with perfect data will only get harder, whilst dealing with situations where erroneous data has been pushed in will remain impossible.

That completes our brief look at event validation. We hope it is clear why this is such an important topic. For us at Snowplow, event validation is a key part of our quest for high-fidelity event analytics - so expect to hear more from us on this topic soon!

[high-fidelity]: /assets/img/blog/2013/04/high-fidelity-2000.jpg
[observepoint]: http://www.observepoint.com/
[snowplow-080]: /blog/2013/04/03/snowplow-0.8.0-released-with-all-new-scalding-based-data-enrichment/
[struct-events]: https://github.com/snowplow/snowplow/wiki/snowplow-tracker-protocol#wiki-event