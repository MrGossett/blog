+++
date = "2015-03-07T21:06:46-04:00"
title = "aws tab complete"
tags = ["rc", "AWS", "bash"]
author = "Tim Gossett"
+++

I would go insane if I did't wire-up tab completion for the `aws` CLI. There are just too many commands to remember. To do that, I dropped this line in my `.awsrc`:

```bash
complete -C aws_completer aws
```

So now when I type `aws <tab>`, I get a list of possible commands, like this:

```sh
~ $ aws <tab>
autoscaling         ec2                 rds
cloudformation      ecs                 redshift
cloudhsm            elasticache         route53
cloudsearch         elasticbeanstalk    route53domains
cloudsearchdomain   elastictranscoder   s3
cloudtrail          elb                 s3api
cloudwatch          emr                 ses
cognito-identity    glacier             sns
cognito-sync        iam                 sqs
configservice       importexport        storagegateway
configure           kinesis             sts
datapipeline        kms                 support
deploy              lambda              swf
directconnect       logs
dynamodb            opsworks
```

And similarly for a subcommand like `aws elasticbeanstalk describe-<tab>`:

```sh
$ aws elasticbeanstalk describe-<tab>
describe-application-versions     describe-environment-resources
describe-applications             describe-environments
describe-configuration-options    describe-events
describe-configuration-settings
```