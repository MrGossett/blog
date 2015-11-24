+++
date = "2015-11-09T21:09:14-04:00"
title = "eb config --> deploy"
tags = ["AWS", "bash"]
author = "Tim Gossett"
+++

I learned something useful the other day. We use AWS ElasticBeanstalk (let's just call it EB) at work for deploying the various components of our SOA. I use [the `eb` CLI tool](http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/eb-cli3.html) a lot when working with those EB apps.

EB stores the configuration data it needs for each app as a YAML file in S3. One of the commands the `eb` CLI provides is [`eb config`](http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/eb3-config.html), which allows management of those YAML files from the command line:

* `eb config list` to list the saved configurations in S3
* `eb config save` to download a YAML file that describes the current configuration of the app
* `eb config put` to upload a YAML file from local disk to S3
* etc.

What I didn't know was that just `eb config` will pull down the app's current configuration and send it to a buffer in the current `EDITOR`. If any changes are saved in that buffer, then `eb` will deploy the changed configuration to the running EB app.

This means that updating, e.g., a security group is as simple as `eb config`, make the change, then save to deploy.
