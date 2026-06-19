# [A Bucket of Phish](https://tryhackme.com/room/hfb1abucketofphish)

## From the Hackfinity Battle CTF event.

DarkInjector has been using a Cmail phishing website to try to steal our credentials. We believe some of our users may have fallen for his trap. Can you retrieve the list of victim users?

Here's the link to the website: http://darkinjector-phish.s3-website-us-west-2.amazonaws.com

### What is the flag?

1. To confirm the S3 backend and check whether the bucket exposes any direct object access, probe a non-existent path on the site:

```bash
curl http://darkinjector-phish.s3-website-us-west-2.amazonaws.com/test
```

**Results:**

```html
<html>
  <head>
    <title>404 Not Found</title>
  </head>
  <body>
    <h1>404 Not Found</h1>
    <ul>
      <li>Code: NoSuchKey</li>
      <li>Message: The specified key does not exist.</li>
      <li>Key: test</li>
      <li>RequestId: WGNH8413FSZAFX3C</li>
      <li>
        HostId:
        fs7vU7hpcTRv1MugC6Qodpy2RZG9tn7mE6VM0fdKQAcjMi1dMfxq1iBHkhmXfJvyC4nyzlwntVkfXcy2Mbfb7FCYUXWXMNSH
      </li>
    </ul>
    <h3>
      An Error Occurred While Attempting to Retrieve a Custom Error Document
    </h3>
    <ul>
      <li>Code: NoSuchKey</li>
      <li>Message: The specified key does not exist.</li>
      <li>Key: error.html</li>
    </ul>
    <hr />
  </body>
</html>
```

> The S3 XML error response is characteristic of AWS and confirms that the bucket is not serving a custom 404 page. A sign that it may be misconfigured. The absence of an `error.html` object further suggests the bucket was set up hastily.

2. Use the AWS CLI to list the contents of the bucket without authentication. S3 buckets that have public list access enabled can be enumerated by anyone using the `--no-sign-request` flag, which skips credential signing entirely:

```bash
aws s3 ls s3://darkinjector-phish --no-sign-request
```

**Results:**

```
2025-03-17 02:46:17        132 captured-logins-093582390
2025-03-17 02:25:33       2300 index.html
```

> The bucket is publicly listable. Two objects are present: `index.html` and a suspicious file named `captured-logins-093582390`. The name makes it clear this is a credential harvest file where the phishing site was storing stolen credentials.

3. Download the credential harvest file directly via the static website URL to read its contents:

```bash
curl http://darkinjector-phish.s3-website-us-west-2.amazonaws.com/captured-logins-093582390
```

> The file contains the list of victim users who submitted their credentials to the phishing page, along with the flag:

[SCREEN01]
