---
layout: post
title:  "Structured JSON in R"
date:   2020-05-13
permalink: structured-json-in-R
categories: R
---

Working with JSON in R can be a pain in the butt. Recently, I encountered a pretty frustrating error caused by some sensible defaults in the rjson and jsonlite packages. Let’s dive in:

In this example, I need to read JSON into R, modify it, and write it back in the exact format in which I received it. The JSON will be used for another API-related task, so it’s structure must be identical to the template.

## JSON string ##

{% highlight r %}

json_string <- {"data_source_id": "1000000", "account_ids": ["1000"],
                "attributes":["field1", "field2", "field3"]}

{% endhighlight %}

This string is comprised of three separate entities:
* **data_source_id**: a simple key/value pair
* **account_ids**: a single value JSON array
* **attributes**: a multi-value JSON array

Now, the operational flow that I want to implement looks something like this:

1. Ingest JSON as text
2. Convert to R list object
3. Modify R list object
4. Convert back to JSON string

Unfortunately, using the defaults for rjson and jsonlite won’t quite get us there. Below, I’ll show you what happens and how to fix it.

## Using jsonlite with defaults

Here’s what happens when we try to use jsonlite with the defaults. I’ll use pipes for simplicity.

{% highlight r %}

json_string %>% # our JSON string
	jsonlite::fromJSON() %>% # convert from JSON to an R list object
	# list modification code happens here
	jsonlite::toJSON()  # convert R list object back to JSON

> {"data_source_id":["1000000"],"account_ids":["1000"],
  "attributes":["field1","field2","field3"]}

{% endhighlight %}

Our JSON string has largely stayed in-tact. However, did you notice that the first value of our JSON string now has brackets around it? It went from `"data_source_id":"1000000"` to `"data_source_id":["1000000"]`.

This means that it has been changed from a key/value pair to a single value array. This can and will cause problems when using this JSON with an API.

## Using rjson with defaults

Here’s what happens when we try to use rjson with the defaults:

{% highlight r %}

json_string %>%
     rjson::fromJSON() %>%
     # list modification code happens here
     rjson::toJSON()

> {"data_source_id":"1000000","account_ids":"1000",
    "attributes":["field1","field2","field3"]}

{% endhighlight %}

Notice how the brackets around `1000` disappeared? In this case, rjson has changed our `account_ids` from a single value array to a key/value pair. Again, this will cause problems when interacting with programmatic systems.

## How to fix it with jsonlite

{% highlight r %}

json_string %>%
	jsonlite::fromJSON(simplifyVector = FALSE) %>%
	# list modification code happens here
	jsonlite::toJSON(auto_unbox = TRUE)

> {"data_source_id":"1000000","account_ids":["1000"],
"attributes":["field1","field2","field3"]}

{% endhighlight %}


I made two changes here. By setting `simplifyVector = FALSE` in `jsonlite::fromJSON()`, I force the function to preserve the structure of our JSON and not simplify any of the elements.

Furthermore, when converting it from a list to a JSON, I need to specify auto_unbox = TRUE. This ensures that single value elements should continue to be classified as an array, not as a key/value pair.

## How to fix it with rjson

{% highlight r %}

json_string %>% # our JSON string
     rjson::fromJSON(simplify = FALSE) %>%
     # modification code happens here
     rjson::toJSON()

> {"data_source_id":"1000000","account_ids":["1000"],
    "attributes":["field1","field2","field3"]}

{% endhighlight %}

In this case, all that’s needed is to specify `simplify = FALSE` in the `rjson::fromJSON()` function. This forces the data in the JSON to preserve its original structure.

## Conclusion

Both of these packages take a different approach to handling JSON, but both have settled on the same default pattern: simplifying the underlying data. For data analysis, which is likely the use-case in R most of the time, this makes sense. However, when working with structural JSON constraints, it helps to know which arguments are available.

If you aren’t familiar with JSON, I’d highly recommend spending some time familiarizing yourself with it. JSON is by far the most popular way to store structured, nested and hierarchical data in a computationally interchangeable format.

## References:

[json.org](https://www.json.org/json-en.html)

[jsonlite package](https://cran.r-project.org/web/packages/jsonlite/index.html)

[rjson package](https://cran.r-project.org/web/packages/rjson/index.html)
