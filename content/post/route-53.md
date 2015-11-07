+++
date = "2015-03-17T07:26:35-04:00"
title = "route 53"
tags = ["rc", "AWS", "go"]
author = "Tim Gossett"
+++

At work, we created an internal API for vending sub-domains of a particular, somewhat popular domain we own. The project is written in Go and wraps the AWS Route 53 API to manage DNS records. We decided to use [`github.com/mitchellh/goamz`](https://github.com/mitchellh/goamz), which is has pretty well-rounded implementations of most of the AWS APIs.

When using `goamz/route53`, things were working perfectly when `aws.GetAuth` was passed credentials directly--i.e., the things I normally set as `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`. It also worked fine when it fell back to `aws.SharedAuth` (pulling credentials from `$HOME/.aws/credentials`) or `aws.EnvAuth` (pulling credentials from the environment).

However, we deployed this service on AWS ElasticBeanstalk, so it's ultimately running in an EC2 instance. That means we can reach Instance Metadata service on the box at `http://169.254.169.254`, and get ephemeral Instance Credentials tailored specifically for that EC2 instance. We created a role to allow access to Route 53, and assigned that role in our Beanstalk config so that each instance of the sevice would be able to use Instance Credentials to talk to Route 53.

Further, `goamz/aws` supports this with `aws.getInstanceCredentials`. It hits a particular route at `http://169.254.169.254` and parses credentials from the response. And it was easy to switch: all we had to do was pass empty strings to `aws.GetAuth`, and it would fall through to `aws.getInstanceCredentials` (skipping `aws.SharedAuth` because there is no credential file, and skipping `aws.EnvAuth` because those particular environment variables are not set) to get credentials for the instance's role.

Then we hit a bump. We kept getting the following error from `goamz` (whitespace added for readability):

```
Request failed, got status code: 403. Response:
<?xml version="1.0" ?>
<ErrorResponse xmlns="https://route53.amazonaws.com/doc/2013-04-01/">
    <Error>
        <Type>Sender</Type>
        <Code>InvalidClientTokenId</Code>
        <Message>The security token included in the request is invalid</Message>
    </Error>
    <RequestId>49ea8b24-c929-11e4-bdf8-2bd74b0c762c</RequestId>
</ErrorResponse>
```

It turns out that the requests to Route 53 were missing a header. Here's the relevant snippet from the [docs](http://docs.aws.amazon.com/AmazonS3/latest/dev/RESTAuthentication.html#UsingTemporarySecurityCredentials):

> If you are signing your request using temporary security credentials, you must include the corresponding security token in your request by adding the `x-amz-security-token` header.

I traced through the code to the `sign` func in `goamz/route53/sign.go`:

```go
func sign(auth aws.Auth, path string, params map[string]string) {
	date := time.Now().In(time.UTC).Format(time.RFC1123)
	params["Date"] = date
	hash := hmac.New(sha256.New, []byte(auth.SecretKey))
	hash.Write([]byte(date))
	signature := make([]byte, b64.EncodedLen(hash.Size()))
	b64.Encode(signature, hash.Sum(nil))
 
	header := fmt.Sprintf("AWS3-HTTPS AWSAccessKeyId=%s,Algorithm=HmacSHA256,Signature=%s",
		auth.AccessKey, signature)
	params["X-Amzn-Authorization"] = string(header)
 }
```

If the `aws.Auth` struct has a non-nil `Token`, it should be added in a `X-Amz-Security-Token` header.

I added this to the bottom of the func:

```go
if auth.Token != "" {
	params["X-Amz-Security-Token"] = auth.Token
}
```

With this change, we were able to use Instace Credentials (which are Temporary Security Credentials) just fine.

I opened a [Pull Request](https://github.com/mitchellh/goamz/pull/238) to `goamz` with this change (and a small fix courtesy of `goimports`). There's a bit of a PR backlog on the repo at the time of writing, so I'm not sure my PR will get merged any time soon. Until then, my fork is go-gettable at [`github.com/MrGossett/goamz/route53`](https://github.com/MrGossett/goamz).