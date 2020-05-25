---
layout: post
title: Gitlab CI + AWS S3
categories: [troubleshooting]
tags: [gitlab, ci, aws, s3]
description: Troubleshooting CI problems
---

Hello there,

First of all, great weekly posts (sarcasm).
Today I was seeked to help with some CI of our homologation environment and the problem was simple but it can pass unoticed thats why I resolved to post to help whom may need.

Here at my company we use a local Gitlab as repository and the Gitlab runner to build and deply our applications. I'm not very experienced with Gitlab but after a few troubleshootings in the past I understand how it works and where to look which can help a lot under this circunstanses.
Now the problem, one of my coworkers came to me with the error below on the pipeline and asking to add the permission to the AWS Key used on the pipeline to run our deploy on Homologation.

{% highlight yaml %}
$ aws s3 rm $HLG_S3 --recursive
fatal error: An error occurred (AccessDenied) when calling the ListObjectsV2 operation: Access Denied
ERROR: Job failed: command terminated with exit code 1
{% endhighlight %}

Looking at the AWS IAM the user had full permissions on the Bucket in question, so I asked to look at the project in Gitlab because theres two options, or the AWS Key is wrong or something changed in the project which in fact the runner changed, they started to use a runner in docker, prior was an EC2 that we used to have. So in this new context I went to check how are the ".gitlab-ci.yml" was configurated and everything looked Ok, below is the code snippet of the deploy for homologation.

{% highlight yaml %}
deploy-homolog:
stage: deploy
variables:
    AWS_ACCESS_KEY_ID: $HLG_KEY_ID
    AWS_SECRET_ACCESS_KEY: $HLG_KEY_SECRET
script:
    - aws s3 rm $HLG_S3 --recursive
    - aws s3 cp ./release/ $HLG_S3 --recursive
    - aws cloudfront create-invalidation --distribution-id $HLG_CLOUDFRONT_ID --paths '/*'
environment: Homolog
when: on_success
variables:
    GIT_STRATEGY: none
only:
    - homolog
{% endhighlight %}

Going to 'Settings -> CI / CD -> Variables' the homolog key was indeed correct, so what was wrong? Maybe the Variable wasn't been correct exported to the new Runner, so I went to check.
I first added the line ``- echo $AWS_ACCESS_KEY_ID`` on the file ".gitlab-ci.yml" after the ``script:`` section to be the first line executed, and as I thought the variable was empty.

{% highlight yaml %}
$ echo $AWS_ACCESS_KEY_ID

$ aws s3 rm $HLG_S3 --recursive
fatal error: An error occurred (AccessDenied) when calling the ListObjectsV2 operation: Access Denied
ERROR: Job failed: command terminated with exit code 1
{% endhighlight %}

Well, now I need to check if the variable from the Gitlab is being loaded in the runner, so I change the ``- echo $AWS_ACCESS_KEY_ID`` to ``- export`` to list all variables and everything seems ok.

{% highlight yaml %}
$ export
declare -x AWS_DEFAULT_REGION="[my region]"
declare -x HLG_KEY_ID="[my key]"
declare -x HLG_KEY_SECRET="[my secret]"
declare -x PRD_KEY_ID="[my key]"
declare -x PRD_KEY_SECRET="[my secret]"
...
declare -x HLG_CLOUDFRONT_ID="[my cdn id]"
declare -x PRD_CLOUDFRONT_ID="[my cdn id]"
declare -x HLG_S3="s3://[my s3 endpoint]"
declare -x PRD_S3="s3://[my s3 endpoint]"
...
$ aws s3 rm $HLG_S3 --recursive
fatal error: An error occurred (AccessDenied) when calling the ListObjectsV2 operation: Access Denied
ERROR: Job failed: command terminated with exit code 1
{% endhighlight %}

_Ps. I added within the brackets the reference of what it is so I won't expose any sensitive information._

So the Gitlab indeed was fowarding the information but the runner wasn't able to export to the variables.
Looking at the history of the ".gitlab-ci.yml" file I noticed something that my eyes didn't notice, there was two ``variables: `` in the file, one in the beggining and other in the end.

{% highlight yaml %}
...
variables:
    AWS_ACCESS_KEY_ID: $HLG_KEY_ID
    AWS_SECRET_ACCESS_KEY: $HLG_KEY_SECRET
...
variables:
    GIT_STRATEGY: none
...
{% endhighlight %}

I removed the second reference and start the pipeline once more, and jackpot! The job finished successfully.

{% highlight yaml %}
$ aws s3 rm $HLG_S3 --recursive
...
$ aws s3 cp ./release/ $HLG_S3 --recursive
...
$ aws cloudfront create-invalidation --distribution-id $HLG_CLOUDFRONT_ID --paths '/*'
...
Job succeeded
{% endhighlight %}

It was a easy fix mistake but I took a hour to find and resolve even after searched on google. In the end I informed the developer because the last coworker changed the file to some tests and left the code with those two references and also to adjust for the production pipeline.

<br><br>
I hope you find this useful to your troubleshootings as well.

{% highlight yaml %}
    $ sp current                                                     [17:21:51]
    Album        Piece of Mind (2015 Remaster)
    AlbumArtist  Iron Maiden
    Artist       Iron Maiden
    Title        Where Eagles Dare - 2015 Remaster
    https://open.spotify.com/track/5QtlbCKhAL70eQM9dbzoR8?si=Pu8vJzTGRd6OfDpF3XqjmQ
{% endhighlight %}

