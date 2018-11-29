# Quarantyne &middot; Modern Web Firewall 🛃 [![Build Status](https://travis-ci.org/quarantyne/quarantyne.svg?branch=master)](https://travis-ci.org/quarantyne/quarantyne) 
_Automated web security made simple_ 

__TL;DR__ Quarantyne is a reverse-proxy that protects web applications and APIs from fraudulent behavior, misuse, bots and cyber-attacks in real-time.
- [Requirements](#requirements)
- [Presentation](#presentation)
- [Features](#features)
- [Quick Run](#quick-run)
- [Configuration](#configuration)
- [Passive vs Active](#passivevsactive)
- [Coverage](#coverage)
- [Distributions](#distributions)
- [License](#license)

## Requirements
- Java 8

## Presentation
Quarantyne is a reverse-proxy written in java. It fronts a web application or API and protects it from malicious requests and misuse. It cannot stop them all, but it will definitely make it harder and more expensive to perform.

It's like a firewall but smarter, because it does not just block traffic because the user-agent is not in a whitelist. Quarantyne also performs deep request inspection to detect if, for example, the password used has been compromised before, or if the email is disposable, with minimal configuration and no changes in your application.  Our [coverage](#coverage) section precisely lists what Quarantyne can identify.

## Features
### Wide coverage of common HTTP threats and misuse
See [coverage](#coverage) for a complete list of the threats and misuse Quarantyne can identify and stop.

### Deep traffic analysis
Quarantyne performs deep inspection of web traffic going to your application to verify that the data being sent is not compromised or junk.

### Generic integration
Quarantyne adds extra HTTP headers to the request it proxies to your service. For example, an HTTP request coming from AWS will bear the following headers:

- `X-Quarantyne-Labels: PCX`
- `X-Quarantyne-RequestId: 08a0e31a-f1a5-4660-9316-0fdf5d2a959d`

### Active protection
Quarantyne can be configured to stop malicious requests 
from reaching your servers, avoiding wasting computing/DB/cache resources, metrics skew, junk data... See (Passive vs Active)[#passivevsactive].

### Metrics & health reporting
Quarantyne binds to an internal `adminPort`, where metrics (latencies, success rate...) as well as the health of the proxy are reported. 

### Privacy friendly / GDPR compliance
Quarantyne is offline software. It runs inside your private network 
and does not communicate over the Internet with anyone to share data 
about your traffic, your business, or your users.

### Ops Friendly.
Single jar with 0 dependencies. Metrics are available on 
`[proxyHost]:[adminPort]/metrics`. Service health is available 
on `[proxyHost]:[adminPort]/health`


## Quick run
### Run the jar
Quarantyne ships as a single 0-dependencies executable jar. Download a release and run:

    $ java -jar quarantyne.jar

### Build from source
Clone this repo or and run the following

    $ ./gradlew run

You should  see the following:
```bash
"2018-10-20T18:41:17.830-0700" [main] INFO com.quarantyne.proxy.Main - ==> quarantyne
"2018-10-20T18:41:17.833-0700" [main] INFO com.quarantyne.proxy.Main - ==> proxy   @ 127.0.0.1:8080
"2018-10-20T18:41:17.833-0700" [main] INFO com.quarantyne.proxy.Main - ==> remote  @ httpbin.org:80
"2018-10-20T18:41:17.834-0700" [main] INFO com.quarantyne.proxy.Main - ==> admin   @ http://127.0.0.1:3231
```

You are all set! By default, Quarantyne starts on 
127.0.0.1:8080, and proxies traffic to http://httpbin.org.

Send a few requests to http://127.0.0.1:8080/headers via various means. If fraudulent behavior is detected, you should see `X-Quarantyne-Label` HTTP headers in the request receive by your application. Hint: try with curl.


## Configuration
Two complementary configuration systems are used: command-line arguments and an external (local or remote) JSON configuration file.

### Command-line arguments
Run the following command to display the help and what arguments are available

    $ java -jar quarantyne -h

The `--traffic-config-file` is an optional JSON configuration file that tells Quarantyne how requests to your service are structured. It enables deep traffic analysis and increase coverage.

### Traffic config JSON file
The traffic config file is optional and can either be an absolute local path or a [remote HTTP(S) URL](https://s3-us-west-2.amazonaws.com/releases.quarantyne.com/quarantyne.json) to a JSON file containing a single JSON object with the following structure. Describing the structure of your HTTP requests helps Quarantyne perform deep inspection of critical data such as password, emails or countries.

```json
{
  "login_action": {
    "path": "/anything",
    "identifier_param": "email",
    "secret_param": "password"
  },
  "register_action": {
    "path": "/anything",
    "identifier_param": "email",
    "secret_param": "password"
  },
  "email_param_keys": ["email", "contact[email]"],
  "country_iso_code_param_keys": ["country_code"],
  "blocked_request_page": "https://raw.githubusercontent.com/AndiDittrich/HttpErrorPages/master/dist/HTTP500.html",
  "blocked_classes": ["all"]
}
```

Quarantyne is able to parse payloads submitted via `POST`/`PUT` with a `Content-Type` of  `application/json` or `application/x-www-form-urlencoded`.

Root properties are optional.

| Property | Definition  | Notes 
 ----- | :-------: | :-----: |
 `*_action` |  A `POST`/`PUT` data payload | `login_action` describes the data structure sent when logging in. `register_action` defines the data structure sent when registering / creating an account.
 `*_action.path` |  Path where data is submitted | Must start by `/`
 `*_action.identifier_param` | Form/JSON key name where the user identifier is sent
 `*_action.secret_param` | Form/JSON key where the user password is sent  
 `email_param_keys` | Form/JSON key where email addresses are sent
 `country_iso_code_param_keys` | Form/JSON key where country iso codes are sent
 `blocked_request_page` | HTTP response to return when blocking a request| It's better when this looks like a legit page/error as to not tip off the attack. Even better if you can inject fake data :)
 `blocked_classes` | An array of attack classes to block. | `[]` is equivalent to passive mode. `['all']` stops every class of attack Quarantyne can detect. See [coverage](#coverage)


## Passive vs. Active
### Passive mode
Quarantyne lets you decide how you want to handle requests it flags. Quarantyne's default configuration is to __NOT__ block tainted traffic. This traffic will make its way to your server and will be labelled as such via HTTP headers. 

Passive mode is the recommended way to get familiar with Quarantyne and to get a sense of what's going on inside your web traffic. In your application, log or plot the incoming Quarantyne labels and you might be surprised (or not) by what you find!

### Active Mode
In active mode, Quarantyne prevents tainted traffic from reaching your application. Blocking happens only you configure explicitely Quarantyne to do so. The [configuration](#configuration) section explains how traffic blocking can be enabled.

## Coverage
Quarantyne is able to detect the following threats and misuse.

| Label | Definition | Behavior | Implemented |
 ----- | :-------: | :-----: | :---
__LBD__ | Large Body Data  | Overload target's form processor with POST/PUT request with body > 1MB | yes
__FAS__ | Fast Browsing | Request rate faster than regular human browsing | yes
__CPW__ | Compromised Password | Password used is known from previous data breach. Possible account takeover | yes
__DMX__ | Disposable Email | Email used is a disposable emails service | yes
__IPR__ | IP Address Rotation | Same visitor is rotating its IP addresses | no
__SHD__ | Suspicious Request Headers| Abnormal HTTP Request headers  | yes
__SUA__ | Suspicious User-Agent | User Agent not from a regular web browser | yes
__PCX__ | Public Cloud Execution | IP address belongs to a public cloud service like AWS or GCP | no
__IPD__ | IP/Country discrepancy | Country inferred from visitor IP is different from country field in submitted request | no
__SGE__ | Suscpicious Geolocation | This request is not usually received from this geolocation. Possible account takeover. | no


## Distributions
### Heroku Buildpack
https://github.com/quarantyne/heroku-buildpack-quarantyne

### Docker image
Coming soon

## License 
[Apache 2](https://github.com/quarantyne/quarantyne/blob/master/LICENSE)

