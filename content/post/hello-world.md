---
title: "Struggles with a GCP Cloud Functions Stacktrace"
date: 2020-05-18T17:00:24-04:00
draft: false
keywords: ["GCP", "Cloud Functions", "Stacktrace", "exception handling", "logging", "error reporting"]
description: "Struggles with inspecting GCP Cloud Functions raised exceptions"
---

I decided to explore restructuring some of our ETL processes that feed data to `BigQuery` in order to have them stop saving data files locally. Currently, we pull data from multiple sources, apply some transformations to them, save local copies of the transformations as well as push the latter to `BigQuery`. In doing this, we essentially make it possible for our on-premise data silos to tap into those locally saved files.
However, as you may have noticed from [my about me page](/post/about), I don't work for Google or Amazon. We don't just have hardware with tons of storage waiting for me to fill up. I have no VM with a storage capacity above 50 Gb that is dedicated to a non-database file system. Only the database file systems get to go past that hard limit. In many ways, this makes sense since compression tools and log aggregators have gotten very good over the years. So, I don't want to keep data files lying around on machines with somewhat limited storage once I have processed them. But I still have to process tons of data every day and push it to `BigQuery`. Of course, the answer had to lie somewhere between `Storage` and `BigQuery`. And, you guessed it right! It's `Cloud Functions`.
These suckers are so good that they will let you trigger some cool operations just because a file got dropped off in a given `Storage` bucket. The cool kids with a certain "Web Services" persuasion will say: _but this is exactly what lambdas do!_. And then, I say: _cooool!_.

So, how do we go from local files to stacktraces on some public cloud? If you're already bored reading this, the answer is:
1.  write some buggy/crashing code,
2.  then `gcloud functions deploy`,
3.  and finally `gstuil cp`.

But, for those of us with a modicum of patience, let's find out in a much more elegant manner.

> ### Deploying a function

The goal here is to write a few lines of python code to import data from `Storage` to `Bigquery`. Lucky for us, this already comes in a very well maintained package unsurprisingly called [google-cloud-bigquery](https://pypi.org/project/google-cloud-bigquery/). Using that package, we get a valid path to our data file in `Storage`, create a connection to `BigQuery` and load our data. Evidently, we need to make sure the destination datasets and tables exist in `BigQuery`. The following does exactly that:

{{< gist AbdouSeck 40ea3d0d332340f67ce9ed57ec250444 "main.py" >}}
__The [`utils.py`](AbdouSeck/40ea3d0d332340f67ce9ed57ec250444) module is a file with some useful constants and functions defined to help with dataset and table names in BigQuery, as well as to wait for the load job to finish.__

The deployment shell script looks like the following:

{{< gist AbdouSeck 40ea3d0d332340f67ce9ed57ec250444 "deploy.sh" >}}

> ### The main issues

Looking at that python script, there are so many places where something could go wrong:

>   1. Line 17 makes this wild assumption that the `PROJECT_ID` environment variable is actually set. It was provided to `gcloud functions deploy` via the `--env-vars-file` option, but it may very well be possible that it will not actually propagate.
>   2. The subsquent bit of code after that (lines 18-21) does assume that `GCP` will run our function on a unix machine, since we're assuming that `os.path.join` will use `/` for a path separator.
>   3. Line 23 makes this wild assumption that every file dropped off in our bucket is going to always have a name with at least one underscore; meaning that we can always get the values `ds` and `tbl` without hitting a `ValueError` exception complaining about there not being enough values to unpack.
>   4. There is also this highly unlikely scenario where between the time the function is triggered to run and the time it reaches line number 46, someone crazy fast person goes ahead and deletes the copied file or even the bucket itself. As a result, we would end trying to load a file that's no longer there.
>   5. Last, but far from being least, we have the `wait_for_job` function throw a `RuntimeError` if the load job is unsuccessful.

Any one of these scenarios could cause the script to throw an error. So, it's very vital that we be able to see some type of logs when the script raises some exception. You'll be happy to know that the logs do exist and can be fetched--I mean, this is `Google` after all. In any case, when your script runs, you can inspect most the available logs from within your `GCP` console or by using `gcloud logs` in your terminal.

> ### The actual struggles

Much of the struggles stemmed from line number 49 of the python script. I accidentally forgot to prepend the argument to the `source_uris` parameter with the `gs://` protocol. The latter is essential in helping the client library get to the actual file that needs to be loaded to `BigQuery`. Luckily, our `wait_for_job` should be able to catch this and throw a `RuntimeError` exception. However, according to the logs, the following happened to my function:

{{< highlight bash >}}
storage-2-bq-loader  some-exec-id  2020-05-18 17:28:19.625  execution took 2121 ms, finished with status: 'crash'
{{< / highlight >}}

That's about all I could get. In hindsight, I know to remember to add the `gs://` protocol indicator, but I still want my stacktrace! As far as [the documentation on `Cloud Functions` error reporting](https://cloud.google.com/functions/docs/monitoring/logging#functions-log-helloworld-python) is concerned, the `wait_for_job` function should have dumped something to the standard error of my process. That something should have been reported in the logs of the function. It never made it there.

That is my struggles.


> ### A little bit of hope

There is a glimmer of hope--but in the form of another operations-oriented product called [`Stackdriver` or `Operations`](https://cloud.google.com/products/operations). This, of course, entails that I have to tweak my function to report exceptions to `Stackdriver` rather than to throw said exceptions to the standard error of my process. In terms of source code changes, I would have to wrap the entirety of my function around a `try-except` to handle the base `Exception`. Only at this point, do I report my exception to `Stackdriver`. Further, reporting exceptions to `Stackdriver` can be handled using another Google-maintained python package called [`google-cloud-error-reporting`](https://cloud.google.com/error-reporting/docs/setup/python). In terms of source code changes, we'd likely end up with the following:

{{< gist AbdouSeck 40ea3d0d332340f67ce9ed57ec250444 "new_main.py" >}}

This can still fail when trying to create our `reporter` value.

> ### Conclusion

While I am not particularly satisfied with adding more code and redeploying my function, I think I can _kind of_ live with this.

Like all types of struggles, it often takes an outsider to help us realize what we're doing wrong. So, feel free to share your wisdom below.