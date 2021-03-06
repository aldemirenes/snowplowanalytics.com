---
layout: post
title-short: Snowplow Event Recovery 0.1.0 released
title: "Snowplow Event Recovery 0.1.0 released"
tags: [snowplow, real-time, GCP, AWS, recovery, bad-rows]
author: Ben
category: releases
permalink: /blog/2019/01/16/snowplow-event-recovery-0.1.0-released/
---

We are excited to announce the release of [Snowplow Event Recovery][ser].

The different Snowplow pipelines being all non-lossy, if something goes wrong during schema validation or enrichment, the payloads (alongside the errors that happened) are stored in a bad rows storage solution, be it a data stream or object storage, instead of being discarded.

The goal of recovery is to fix the payloads contained in these bad rows so that they are ready to be
processed successfully by a Snowplow enrichment platform.

Snowplow Event Recovery lets you run data recoveries on data emitted by real-time Snowplow pipelines
on AWS and GCP.

Please read on after the fold for:

1. [Overview](#overview)
2. [Recovery scenarios](#csl)
3. [Snowplow Event Recovery on AWS](#aws)
4. [Snowplow Event Recovery on GCP](#gcp)
5. [Roadmap](#roadmap)
6. [Getting help](#help)

<!--more-->

<h2 id="overview">1. Overview</h2>

Our current approach to data recovery, [Hadoop Event Recovery][hadoop-recovery], suffers from a few
issues:

- It's limited to data produced by the batch pipeline
- You need to code your own recovery almost from scratch in JavaScript
- You cannot test this JavaScript except by running an actual recovery
- It doesn't promote reuse: if you run the same recovery twice, you'll need to copy/paste your
recovery code from one recovery to another

Snowplow Event Recovery aims to tackle most of these issues and make the data recovery process:

- Not require any coding for the most common cases
- Extensible when outside the most common cases
- Testable
- Unified across the real-time pipelines (AWS and GCP) and, in the future across all pipelines
(real-time and batch)

<h2 id="csl">2. Recovery scenarios</h2>

Keeping these goals in mind, we started by thinking about what a recovery is, in essence. For us,
it is a collection of what we've come to call a recovery scenario.

So, what are recovery scenarios? They are modular and composable processing units that will deal
with a specific case you want to recover from.

As such, recovery scenarios are, at their essence, made up of two things:

- An error filter, which will serve as a router between bad rows and their appropriate recovery
scenario(s)
- A mutation function, which will actually "fix" the payload

For example, if we wanted to recover a set of bad rows consisting of:

- Bad rows that were created due to a missing schema
- Bad rows that were created due to the payload not conforming to its schema
- Bad rows that were created due to an enrichment failing

We would use a different recovery scenario for each of them, so three in total:

- A first recovery scenario consisting of:
  - an error filter checking for missing schema errors
  - a mutate function which does nothing (assuming the schema has been added since the bad rows
occurred)
- A second recovery scenario consisting of:
  - an error filter checking for payloads not conforming to their schema errors
  - a mutate function which makes the payloads fit their schema
- A third recovery scenario consisting of:
  - an error filter checking for a particular enrichment failing errors
  - a mutate function which does nothing (assuming the enrichment was misconfigured and we just want
to rerun it)

<h3 id="out-of-the-box">2.1 Out of the box recovery scenarios</h3>

For the most common recovery scenarios, it makes sense to support them out of the box and not
require any coding. From the recoveries we've run in the past, we've compiled a list of recovery
scenarios that are supported out of the box by Snowplow Event Recovery.

In the table below, you can find what this list is made of, it contains:

- The name of the recovery scenario
- What the mutation function will do
- An example use case
- The parameters to this recovery scenario
<br>
<br>
<table class="table-responsive table-bordered table">
<thead>
<tr>
<th style="text-align:center">Name</th>
<th style="text-align:center">Mutation</th>
<th style="text-align:center">Example use case</th>
<th style="text-align:center">Parameters</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:center">Pass through</td>
<td style="text-align:center">Does not mutate the payload in any way</td>
<td style="text-align:center">A missing schema that was added after the fact</td>
<td style="text-align:center"><code>error</code></td>
</tr>
<tr>
<td style="text-align:center">Replace in query string</td>
<td style="text-align:center">Replaces part of the query string according to a regex</td>
<td style="text-align:center">Misspecified a schema when using the Iglu webhook</td>
<td style="text-align:center"><code>error</code>, <code>toReplace</code>, <code>replacement</code></td>
</tr>
<tr>
<td style="text-align:center">Remove from query string</td>
<td style="text-align:center">Removes part of the query string according to a regex</td>
<td style="text-align:center">Property was wrongfully tracked and is not part of the schema</td>
<td style="text-align:center"><code>error</code>, <code>toRemove</code></td>
</tr>
<tr>
<td style="text-align:center">Replace in base64 field in query string</td>
<td style="text-align:center">Replaces part of a base64 field in the query string according to a regex</td>
<td style="text-align:center">Property was sent as a string but should be an numeric</td>
<td style="text-align:center"><code>error</code>, <code>base64Field</code> (<code>cx</code> or <code>ue_px</code>), <code>toReplace</code>, <code>replacement</code></td>
</tr>
<tr>
<td style="text-align:center">Replace in body</td>
<td style="text-align:center">Replaces part of the body according to a regex</td>
<td style="text-align:center">Misspecified a schema when using the Iglu webhook</td>
<td style="text-align:center"><code>error</code>, <code>toReplace</code>, <code>replacement</code></td>
</tr>
<tr>
<td style="text-align:center">Remove from body</td>
<td style="text-align:center">Removes part of the body according to a regex</td>
<td style="text-align:center">Property was wrongfully tracked and is not part of the schema</td>
<td style="text-align:center"><code>error</code>, <code>toRemove</code></td>
</tr>
<tr>
<td style="text-align:center">Replace in base64 field in body</td>
<td style="text-align:center">Replaces part of a base64 field in the body according to a regex</td>
<td style="text-align:center">Property was sent as a string but should be an numeric</td>
<td style="text-align:center"><code>error</code>, <code>base64Field</code> (<code>cx</code> or <code>ue_px</code>), <code>toReplace</code>, <code>replacement</code></td>
</tr>
</tbody>
</table>


Note that, for every recovery scenario leveraging a regex, it's possible to use capture groups. For
example, to remove brackets but keep their content we would have a `toReplace` argument containing
`\\{(.*)\\}` and a `replacement` argument containing `$1` (capture groups are one-based numbered).

<h3 id="custom">2.2 Custom recovery scenarios</h3>

In addition to the outlined scenarios, we still wanted to make the idea of  recovery scenarios
extensible. As such, if the recovery you want to perform is not covered by the ones listed
above, you can define your own by following [the guide in the repository][custom-recovery-scenario].

If you think your recovery scenario will be useful to others, please consider opening a pull
request!

<h3 id="config">2.3 Configuration</h3>

Once you have identified the different recovery scenarios you will want to run, you can combine
them in the configuration that we will feed to the recovery job. Here, we make use of each and every
one of them as a showcase.

{% highlight json %}
{
  "schema": "iglu:com.snowplowanalytics.snowplow/recoveries/jsonschema/1-0-0",
  "data": [
    # Schema com.acme/my_schema/jsonschema/1-0-0 was added after the fact
    {
      "name": "PassThrough",
      "error": "Could not find schema with key iglu:com.acme/my_schema/jsonschema/1-0-0 in any repository"
    },
    # Typo in the schema name when using the Iglu webhook
    {
      "name": "ReplaceInQueryString",
      "error": "Could not find schema with key iglu:com.snowplowanalytics.snowplow/screen_vie/jsonschema/1-0-0 in any repository",
      "toReplace": "schema=iglu%3Acom.snowplowanalytics.snowplow%2Fscreen_vie%2Fjsonschema%2F1-0-0",
      "replacement": "schema=iglu%3Acom.snowplowanalytics.snowplow%2Fscreen_view%2Fjsonschema%2F1-0-0"
    },
    # Removes illegal curlies in query strings (e.g. templates that haven't been filled)
    {
      "name": "RemoveFromQueryString",
      "error": "Exception extracting name-value pairs from querystring",
      "toRemove": "\\{.*\\}"
    },
    # Replaces a string by an integer in ue_px, it can be reused for ReplaceInBase64FieldInBody
    {
      "name": "ReplaceInBase64FieldInQueryString",
      "error": "instance type (string) does not match any allowed primitive type (allowed: [\"integer\"])\n    level: \"error\"\n    schema: {\"loadingURI\":\"#\",\"pointer\":\"/properties/sessionIndex\"",
      "base64Field": "ue_px",
      "toReplace": "\"sessionIndex\":\"(\\d+)\"",
      # $1 refers to the first capture group
      "replacement": "\"sessionIndex\":$1"
    },
    # Replaces the device created timestamp by a string
    {
      "name": "ReplaceInBody",
      "error": "instance type (integer) does not match any allowed primitive type (allowed: [\"string\"])\n    level: \"error\"\n    schema: {\"loadingURI\":\"#\",\"pointer\":\"/items/properties/dtm\"",
      "toReplace": "\"dtm\":(\\d+)",
      "replacement": "\"dtm\":\"$1\""
    },
    # Removes a field which shouldn't be there
    {
      "name": "RemoveFromBody",
      "error": "object instance has properties which are not allowed by the schema: [\"test\"]",
      "toRemove": "\"test\":\".*\",?"
    },
    # Same as ReplaceInBase64FieldInQueryString
    {
      "name": "ReplaceInBase64FieldInBody",
      "error": "instance type (string) does not match any allowed primitive type (allowed: [\"integer\"])\n    level: \"error\"\n    schema: {\"loadingURI\":\"#\",\"pointer\":\"/properties/sessionIndex\"",
      "base64Field": "ue_px",
      "toReplace": "\"sessionIndex\":\"(\\d+)\"",
      # $1 refers to the first capture group
      "replacement": "\"sessionIndex\":$1"
    },
    # Our custom recovery scenario, replaces a wrong Iglu webhook path
    {
      "name": "ReplaceInPath",
      "error": "Payload with vendor com.iglu and version v1 not supported",
      "toReplace": "com.iglu/v1",
      "replacement": "com.snowplowanalytics.iglu/v1"
    }
  ]
}
{% endhighlight %}

<h3 id="config">2.4 Testing</h3>

It's possible to test an entire recovery without running it or a custom recovery scenario by
following [the dedicated guide in our repository][recovery-testing].

<h2 id="aws">3. Snowplow Event Recovery on AWS</h2>

For AWS users, the recovery will take the form of a Spark job which you can run through EMR, for
example. It will read bad rows from an S3 location, run the recovery on this data, and store the
recovered payloads in another S3 location.

You can run the job using the JAR directly (which is hosted at
`s3://snowplow-hosted-assets/3-enrich/snowplow-event-recovery/`):

{% highlight bash %}
spark-submit \
  --class com.snowplowanalytcs.snowplow.event.recovery.Main \
  --master master-url \
  --deploy-mode deploy-mode \
  snowplow-event-recovery-spark-0.1.0.jar
  --input s3://bad-rows-location/
  --output s3://recovered-collector-payloads-location/
  --config base64-encoded-configuration
{% endhighlight %}

Or through an EMR step:

{% highlight bash %}
aws emr add-steps --cluster-id j-XXXXXXXX --steps \
  Name=snowplow-event-recovery,\
  Type=CUSTOM_JAR,\
  Jar=s3://snowplow-hosted-assets/3-enrich/snowplow-event-recovery/snowplow-event-recovery-spark-0.1.0.jar,\
  MainClass=com.snowplowanalytics.snowplow.event.recovery.Main,\
  Args=[--input,s3://bad-rows-location/,--output,s3://recovered-collector-payloads-location/,--config,base64-encoded-configuration],\
  ActionOnFailure=CONTINUE
{% endhighlight %}

Note that the configuration discussed above will need to be base64-encoded.

<h2 id="gcp">4. Snowplow Event Recovery on GCP</h2>

For GCP users, leveraging the data outputted by [the Snowplow Google Cloud Storage Loader][sgcsl],
recovery will take the shape of a Beam job runnable on Dataflow. It will read bad rows from a GCS
location specified through a pattern, run the recovery on this data, and store the recovered
payloads in a PubSub topic (ideally your PubSub topic containing the raw payloads so that fixed
payloads can be directly picked up by the enrichment process).

You can run the job using the zip archive, which can be downloaded from Bintray
[here][bintray-archive]:

{% highlight bash %}
./bin/snowplow-event-recovery-beam \
  --runner=DataFlowRunner \
  --project=project-id \
  --zone=europe-west2-a \
  --gcpTempLocation=gs://location/ \
  --inputDirectory=gs://bad-rows-location/\* \
  --outputTopic=projects/project/topics/topic \
  --config=base64-encoded-configuration
{% endhighlight %}

Or using a Docker container:

{% highlight bash %}
docker run \
  -v $PWD/config:/snowplow/config \ # if running outside GCP
  -e GOOGLE_APPLICATION_CREDENTIALS=/snowplow/config/credentials.json \ # if running outside GCP
  snowplow-docker-registry.bintray.io/snowplow/snowplow-event-recovery:0.1.0 \
  --runner=DataFlowRunner \
  --project=project-id \
  --zone=europe-west2-a \
  --gcpTempLocation=gs://location/ \
  --inputDirectory=gs://bad-rows-location/\* \
  --outputTopic=projects/project/topics/topic \
  --config=base64-encoded-configuration
{% endhighlight %}

Note that, here too, the configuration discussed above will need to be base64-encoded.

<h2 id="roadmap">5. Roadmap</h2>

Continuing our data quality journey, we will next work towards a new bad row format. You can read
more about this initiative in [our RFC][rfc].

On the Snowplow front, the next releases will include:

- [R112 Baalbek][r112] which will aim to improve the batch pipeline
- [R113][r113] which will focus on the real-time pipeline and incorporate community pull requests

After these two releases, the pipeline team will focus its effort on the new bad row format.

<h2 id="help">6. Getting help</h2>

For more details on this release, please check out the [release notes][release] on GitHub.

If you have any questions or run into any problem, please visit [our Discourse forum][discourse].

[release]: https://github.com/snowplow-incubator/snowplow-event-recovery/releases/0.1.0
[ser]: https://github.com/snowplow-incubator/snowplow-event-recovery/

[discourse]: https://discourse.snowplowanalytics.com/

[hadoop-recovery]: https://github.com/snowplow/snowplow/wiki/Hadoop-Event-Recovery
[custom-recovery-scenario]: https://github.com/snowplow-incubator/snowplow-event-recovery#custom-recovery-scenario
[recovery-testing]: https://github.com/snowplow-incubator/snowplow-event-recovery#testing
[sgcsl]: https://snowplowanalytics.com/blog/2018/11/13/snowplow-google-cloud-storage-loader-0.1.0-released/
[rfc]: https://discourse.snowplowanalytics.com/t/a-new-bad-row-format/2558
[bintray-archive]: https://bintray.com/snowplow/snowplow-generic/snowplow-event-recovery

[r112]: https://github.com/snowplow/snowplow/milestone/162
[r113]: https://github.com/snowplow/snowplow/milestone/165
