---
layout: post
title:  "Directly Query CSVs in BigQuery using External Tables"
date:   2020-05-25
permalink: directly-query-csvs-in-BigQuery-using-external-tables
categories: BigQuery, Google Cloud
---

It’d be great if all of our data came from nice, clean SQL databases, or flat files that have been meticulously scrubbed. Unfortunately, that is rarely the case. There are almost always Excel spreadsheets that get emailed to you from business users, Google Sheets that are shared across teams, or CSVs that come from who knows where.

“Manually-curated data”, or MCD,  is how I refer to data that has been assembled, collected, entered or ever once modified by humans.

Having used BigQuery for almost 4 years now, I’ve come across several ETL patterns to import these sources of manual data on a regular basis. BigQuery has several features to import data for one-time use, but these strategies are my favorite for managing MCD that is periodically updated or appended.


## External Tables

External tables in BigQuery are my personal favorite option. By leveraging Google Cloud Storage (different from Google Drive), external tables give you the ability to run your queries directly over an unlimited number of raw or compressed CSV files.

 Here’s how they work:



1. **Create a Cloud Storage Bucket:** You must create it in one of the zones that currently supports External tables. Right now in the US, that’s only us-midwest-1.
![Create Google Cloud Bucket]({{ site.baseurl }}/images/create-google-cloud-bucket.png)
  1. **Add your data to the bucket:** You can add a single CSV, or multiple CSVs, as long as they all have the same schema.
![Upload CSVs to Google Cloud Bucket]({{ site.baseurl }}/images/upload-csvs-google-cloud-bucket.png)
  1. **Create your external table:** In BigQuery, go to your existing dataset to create a new table. You’ll begin by setting the table source to “Google Cloud Storage” and setting the table type to “External”. You’ll then want to set the URI (universal resource identifier) that tells BigQuery which files to query.
  * If you only have a single file, you can select its fully-qualified file path as the URI.
  * If you have multiple files, you can use asterisks to have your URI match several files with different names.
  * Once you’ve set the above table parameters, you’ll need to set the schema. You can use BigQuery’s auto-detect schema feature, which I’ve found quite handy, or you can edit the schema yourself.
![Create a BigQuery External Table]({{ site.baseurl }}/images/create-bigquery-external-table.png)
  1. **Query your data:** Once you’ve set up your external table, you’ll be able to reference it with the full power of BigQuery. If you’ve uploaded multiple files, their results will be available in a single table.

![Query a BigQuery External Table]({{ site.baseurl }}/images/query-external-table.png)

You might wonder how BigQuery is able to directly query CSVs files without loading them first. Under the hood, every time the table is queried, BigQuery loads all of the CSVs into a temporary table and then performs the query on that data.

## Pros:

*   **Ease of use:** Changes in the Cloud Storage Bucket are automatically reflected in the BigQuery table. This makes modifying, updating or deleting data extremely straight forward.
*   **Big data:** External tables work really well for large amounts of data. We have one of these running with over 40k compressed CSVs at WK.
*   **Pipelines:** These work very well for daily data feeds. Each day, we place a new CSV into the Cloud Storage Bucket and it’s automatically available the next time the table is queried.

## Cons:

*   **Performance:** Since an entire load procedure is happening every time you query the data, there’s definitely a fixed performance penalty associated with querying external tables. If the data is frequently accessed (by a dashboard or analyst groups), you may be better off with an ETL process that loads the data incrementally into a Native table.
*   **Partitioning:** An additional implication of this combined process is that you can’t use native BigQuery partitioning or clustering strategies. There are some advanced methods that are explained in further detail [here](https://cloud.google.com/bigquery/docs/hive-partitioned-queries-gcs).
*   **Errors:** Even if you have 40k files in Cloud Storage, if a single one of them has an error, your entire query will fail. That’s good news or bad news depending on who you are, but BigQuery provides very limited information on the errors to end users. Most of the time, you’ll need to dive into the error log to find out what went wrong (great post on how to do that here).
*   **Bucket Locations:** As of right now, there are only a few regions that support queries from external tables. The only one in the US is us-central1. This means that you’ll likely need to move your data to a bucket located in one of these regions in order to work with external tables. Other regions outside the US can be found [here](https://cloud.google.com/bigquery/external-data-sources#external_data_source_limitations).

## What you should use External Tables for:

*   Data that is large in volume, but is infrequently accessed
*   Data that maintains a constant schema
*   Data that is frequently updated, modified or deleted

## What you shouldn’t use External Tables for:

*   Data that is large in volume, and frequently accessed
*   Applications requiring very fast query responses (ie. dashboards)
