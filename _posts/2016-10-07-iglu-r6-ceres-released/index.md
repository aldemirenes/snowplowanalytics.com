---
layout: post
title: Iglu 6 Ceres released with significant updates to Igluctl
title-short: Iglu 6 Ceres
tags: [iglu, json, json schema]
author: Yali
category: Releases
permalink: /blog/2016/10/07/iglu-r6-ceres-released
---

We are pleased to announce a new Iglu release with some significant updates to [Igluctl][igluctl] - our Iglu command-line tool.

Read on for more information on Release 6 Ceres, named after the [first postage stamp release in France] [ceres]:

1. [New option to lint schemas to a higher standard](/blog/2016/10/07/iglu-r6-ceres-released/#severity)
2. [Publish schemas and jsonpath files to S3](/blog/2016/10/07/iglu-r6-ceres-released/#publish-to-s3)
3. [Other updates](/blog/2016/10/07/iglu-r6-ceres-released/#other)

![ceres-img] [ceres-img]

<!--more-->

<h2 id="severity">1. New option to lint schemas to a higher standard</h2>

Snowplow users will define JSON Schemas for event and context types, and then use Igluctl to auto-generate the associated Redshift table definition using the `igluctl static generate` command.

Often, a JSON Schema might be entirely valid. However, it is not precise enough to fully determine the corresponding Redshift table definition. Two examples:

<h3 id="severity-numeric-type">1a. Determining the correct numeric type in Redshift</h3>

If you have a schema that defines a numeric field e.g.

{% highlight json %}
{
  "$schema": "http://iglucentral.com/schemas/com.snowplowanalytics.self-desc/schema/jsonschema/1-0-0#",
  "description": "Schema for an example event",
  "self": {
    "vendor": "com.example_company",
    "name": "example_event",
    "format": "jsonschema",
    "version": "1-0-0"
  },

  "type": "object",
  "properties": {
    "exampleIntegerField": {
      "type": "integer"
    }
  },
  "minProperties":1,
  "additionalProperties": false
}
{% endhighlight %}

When generating the associated Redshift table definition, which Redshift numeric type should be assigned to the `example_integer_field`? Redshift supports three integer types:

1. `smallint`, with a range from -32768 to +32767
2. `integer`, with a range from -2147483648 to +2147483647
3. `bigint`, with a range from -9223372036854775808 to 9223372036854775807

The existing version of the JSON Schema doesn't have enough information to enable Igluctl to determine which of the above field types in Redshift to use.

Now, if you lint the above schema with the default severity level it will pass, because it is a valid JSON Schema:

{% highlight bash %}
$ /path/to/igluctl lint schemas/com.example_company/example_event --severityLevel 1
SUCCESS: Schema [/path/to/schema/registry/schemas/com.example_company/example_event/jsonschema/1-0-0] is successfully validated
TOTAL: 1 Schemas were successfully validated
TOTAL: 0 invalid Schemas were encountered
TOTAL: 0 errors were encountered
{% endhighlight %}

However, if you lint the above schema with the increased severity level 2 it will fail because the schema under-determines the associated Redshift table definition:

{% highlight bash %}
$ /path/to/igluctl lint schemas/com.example_company/example_event --severityLevel 2
FAILURE: Schema [/path/to/schema/registry/schemas/com.example_company/example_event/jsonschema/1-0-0] contains following errors:
1. Numeric Schema doesn't contain minimum and maximum properties
TOTAL: 0 Schemas were successfully validated
TOTAL: 1 invalid Schemas were encountered
TOTAL: 1 errors were encountered
{% endhighlight %}

If we now update the schema to include the `minimum` and `maximumg` properties:

{% highlight json %}
{
  "$schema": "http://iglucentral.com/schemas/com.snowplowanalytics.self-desc/schema/jsonschema/1-0-0#",
  "description": "Schema for an example event",
  "self": {
    "vendor": "com.example_company",
    "name": "example_event",
    "format": "jsonschema",
    "version": "1-0-0"
  },

  "type": "object",
  "properties": {
    "exampleNumericField": {
      "type": "integer",
      "minimum": 0,
      "maximum": 10000
    }
  },
  "minProperties":1,
  "additionalProperties": false
}
{% endhighlight %}

Linting the schema with the higher severity level now works:

{% highlight bash %}
$ /path/to/igluctl lint schemas/com.example_company/example_event --severityLevel 2
SUCCESS: Schema [/path/to/schema/registry/schemas/com.example_company/example_event/jsonschema/1-0-0] is successfully validated
TOTAL: 1 Schemas were successfully validated
TOTAL: 0 invalid Schemas were encountered
TOTAL: 0 errors were encountered
{% endhighlight %}

Now when we use Igluctl to generate our Redshift DDL we can see that Igluctl has correctly set the corresponding Redshift column type to `smallint`:

{% highlight bash %}
$ /path/to/igluctl static generate schemas/com.example_company/example_event
File [/path/to/schema/registry/schemas/com.example_company/example_event/jsonschema/1-0-0] was written successfully!
$ cat sql/com.example_company/example_event_1.sql
-- AUTO-GENERATED BY igluctl DO NOT EDIT
-- Generator: igluctl 0.2.0-rc1
-- Generated: 2016-10-02 12:46

CREATE SCHEMA IF NOT EXISTS atomic;

CREATE TABLE IF NOT EXISTS atomic.com_example_company_example_event_1 (
    "schema_vendor"         VARCHAR(128)  ENCODE RUNLENGTH NOT NULL,
    "schema_name"           VARCHAR(128)  ENCODE RUNLENGTH NOT NULL,
    "schema_format"         VARCHAR(128)  ENCODE RUNLENGTH NOT NULL,
    "schema_version"        VARCHAR(128)  ENCODE RUNLENGTH NOT NULL,
    "root_id"               CHAR(36)      ENCODE RAW       NOT NULL,
    "root_tstamp"           TIMESTAMP     ENCODE LZO       NOT NULL,
    "ref_root"              VARCHAR(255)  ENCODE RUNLENGTH NOT NULL,
    "ref_tree"              VARCHAR(1500) ENCODE RUNLENGTH NOT NULL,
    "ref_parent"            VARCHAR(255)  ENCODE RUNLENGTH NOT NULL,
    "example_numeric_field" SMALLINT      ENCODE LZO,
    FOREIGN KEY (root_id) REFERENCES atomic.events(event_id)
)
DISTSTYLE KEY
DISTKEY (root_id)
SORTKEY (root_tstamp);

COMMENT ON TABLE atomic.com_example_company_example_event_1 IS 'iglu:com.example_company/example_event/jsonschema/1-0-0';
{% endhighlight %}

<h3 id="severity-string-types">1b. Determining the correct string types in Redshift</h3>

The same issue of a JSON Schema field definition under-determining the associated Redshift column type occurs for string fields. If we have the following schema, for example:

{% highlight json %}
{
  "$schema": "http://iglucentral.com/schemas/com.snowplowanalytics.self-desc/schema/jsonschema/1-0-0#",
  "description": "Schema for an example event",
  "self": {
    "vendor": "com.example_company",
    "name": "example_event",
    "format": "jsonschema",
    "version": "1-0-1"
  },

  "type": "object",
  "properties": {
    "exampleNumericField": {
      "type": "integer",
      "minimum": 0,
      "maximum": 10000
    },
    "exampleStringField": {
      "type": "string"
    }
  },
  "minProperties":1,
  "additionalProperties": false
}
{% endhighlight %}

It is clear that the column type for the `example_string_field` should be `VARCHAR`. However, there is nothing to indicate how long the field should be. As a result, the schema under-determines the associated Redshift DDL, and linting the schema with the increased severity level will fail:

{% highlight bash %}
$ /path/to/igluctl lint schemas/com.example_company/example_event/jsonschema/1-0-1 --severityLevel 2
FAILURE: Schema [/path/to/schema/registry/schemas/com.example_company/example_event/jsonschema/1-0-1] contains following errors:
1. String Schema doesn't contain maxLength nor enum properties nor appropriate format
TOTAL: 0 Schemas were successfully validated
TOTAL: 1 invalid Schemas were encountered
TOTAL: 1 errors were encountered
{% endhighlight %}

If we update the field definition to include a `maxLength` property:

{% highlight json %}
{
  "$schema": "http://iglucentral.com/schemas/com.snowplowanalytics.self-desc/schema/jsonschema/1-0-0#",
  "description": "Schema for an example event",
  "self": {
    "vendor": "com.example_company",
    "name": "example_event",
    "format": "jsonschema",
    "version": "1-0-1"
  },

  "type": "object",
  "properties": {
    "exampleNumericField": {
      "type": "integer",
      "minimum": 0,
      "maximum": 10000
    },
    "exampleStringField": {
      "type": "string",
      "maxLength": 100
    }
  },
  "minProperties":1,
  "additionalProperties": false
}
{% endhighlight %}

The schema does validate against the higher `severityLevel`:

{% highlight bash %}
$ /path/to/igluctl lint schemas/com.example_company/example_event/jsonschema/1-0-1 --severityLevel 2
SUCCESS: Schema [/path/to/schema/registry/schemas/com.example_company/example_event/jsonschema/1-0-1] is successfully validated
TOTAL: 1 Schemas were successfully validated
TOTAL: 0 invalid Schemas were encountered
TOTAL: 0 errors were encountered
{% endhighlight %}

Now Igluctl generates the associated Redshift table DDL with the correct field length:

{% highlight bash %}
$ /path/to/igluctl static generate schemas/com.example_company/example_event
File [/Users/yalisassoon/Development/qa/igluctl/test-severity-levels/./sql/com.example_company/example_event_1.sql] already exists and probably was modified manually. You can use --force to override
File [/Users/yalisassoon/Development/qa/igluctl/test-severity-levels/./sql/com.example_company/example_event/1-0-0/1-0-1.sql] was written successfully!
$ cat sql/com.example_company/example_event/1-0-0/1-0-1.sql
-- WARNING: only apply this file to your database if the following SQL returns the expected:
--
-- SELECT pg_catalog.obj_description(c.oid) FROM pg_catalog.pg_class c WHERE c.relname = 'com_example_company_example_event_1';
--  obj_description
-- -----------------
--  iglu:com.example_company/example_event/jsonschema/1-0-0
--  (1 row)

BEGIN TRANSACTION;

  ALTER TABLE atomic.com_example_company_example_event_1
    ADD COLUMN "example_string_field" VARCHAR(100) ENCODE LZO;

  COMMENT ON TABLE atomic.com_example_company_example_event_1 IS 'iglu:com.example_company/example_event/jsonschema/1-0-1';

END TRANSACTION;
{% endhighlight %}

<h2 id="publish-to-s3">2. Publish schemas and JSON Path files to S3</h2>

Previously Igluctl enabled users to publish schemas stored locally to a remote Iglu registry using the `igluctl static push` command.

However, users that wanted to publish schemas to S3-backed static registries, or publish JSON Path files to S3 so they can be used to load event and context data into Redshift, had to use another tool to do so. (Most commonly Amazon's excellent [AWS CLI][aws-cli]).

Igluctl now has a new command: `s3cp`, for copying files locally to S3. This means that you can publish JSON Schemas to S3-backed static registries:

{% highlight bash %}
$ /path/to/igluctl static s3cp ./schemas snowplow-com-mycompany-iglu-schemas-bucket --accessKeyId ABCDEF --secretAccessKey GHIJKILM/12345XYZ --region us-east-1
{% endhighlight %}

and publish JSON path files to s3:

{% highlight bash %}
$ /path/to/igluctl static s3cp ./jsonpaths snowplow-com-mycompany-iglu-jsonpaths-bucket --accessKeyId ABCDEF --secretAccessKey GHIJKILM/12345XYZ --region us-east-1
{% endhighlight %}

<h2 id="other">3. Other updates</h2>

The updated Igluctl includes a number of other small but important updates:

1. [Ability to publish public schemas to the Iglu Scala Schema Registry] [public-schemas]. This means that Igluctl can now publish schemas to Snowplow Mini, which currently requires that the schemas stored on the local service are public rather than private
2. Improved [Windows support] [issue-209] including [an important bug fix] [issue-208]. This comes with proper documentation for Windows users getting started with Igluctl

For a complete list of updates see the [changelog][changelog].

[ceres]: https://en.wikipedia.org/wiki/Ceres_series_(France)
[ceres-img]: /assets/img/blog/2016/10/ceres.jpg
[igluctl]: https://github.com/snowplow/iglu/tree/master/0-common/igluctl
[aws-cli]: https://aws.amazon.com/cli/
[public-schemas]: https://github.com/snowplow/iglu/issues/202
[changelog]: https://github.com/snowplow/iglu/blob/master/CHANGELOG

[issue-209]: https://github.com/snowplow/iglu/issues/209
[issue-208]: https://github.com/snowplow/iglu/issues/208