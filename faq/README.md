# FAQ

## Table of Contents
<!-- TOC depthFrom:3 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [What is Pact good for?](#what-is-pact-good-for)
- [What is Pact not good for?](#what-is-pact-not-good-for)
- [Why doesn't Pact use JSON Schema?](#why-doesnt-pact-use-json-schema)
- [Why does Pact use concrete JSON documents rather than using more flexible JSONPaths?](#why-does-pact-use-concrete-json-documents-rather-than-using-more-flexible-jsonpaths)
- [Why is there no support for specifying optional attributes?](#why-is-there-no-support-for-specifying-optional-attributes)
- [Why are the pacts generated and not static?](#why-are-the-pacts-generated-and-not-static)
- [How do I test against the latest development and production versions of consumer APIs?](#how-do-i-test-against-the-latest-development-and-production-versions-of-consumer-apis)
- [What does PACT stand for?](#what-does-pact-stand-for)
- [Why Pact may not be the best tool for public testing APIs?](#why-pact-may-not-be-the-best-tool-for-public-testing-apis)
- [Why Pact may not be the best tool for testing pass through APIs?](#why-pact-may-not-be-the-best-tool-for-testing-pass-through-apis)
- [Do I still need end-to-end tests?](#do-i-still-need-end-to-end-tests)
- [How can I handle versioning?](#how-can-i-handle-versioning)
- [Using Pact where the Consumer team is different from the Provider team](#using-pact-where-the-consumer-team-is-different-from-the-provider-team)
- [How to prevent a consumer from deploying with an invalid contract](#how-to-prevent-a-consumer-from-deploying-with-an-invalid-contract)
- [How do I test OAuth or other security headers?](#how-do-i-test-oauth-or-other-security-headers)
- [How do I test binary files in responses, such as a download?](#how-do-i-test-binary-files-in-responses-such-as-a-download)

<!-- /TOC -->

### What is Pact good for?
Pact is most valuable for designing and testing integrations where you (or your team/organisation/partner organisation) control the development of both the consumer and the provider, and the requirements of the consumer are going to be used to drive the features of the provider. It is fantastic tool for developing and testing intra-organisation microservices.

### What is Pact not good for?
* Performance and load testing.
* Functional testing of the provider - that is what the provider’s own tests should do. Pact is about checking the contents and format of requests and responses.
* Situations where you cannot load data into the provider without using the API that you’re actually testing (eg. public APIs). Why?
* Testing “pass through” APIs, where the provider merely passes on the request contents to a downstream service without validating them. Why?

### Why doesn't Pact use JSON Schema?

Whether you define a schema or not, you will still need a concrete example of the response to return from the mock server, and a concrete example of the request to replay against the provider. If you just used a schema, then the code would have to generate an example, and generated values are not very helpful when used in tests, nor do they give any readable, meaningful documentation. If you use a schema *and* an example, then you are duplicating effort. The schema can almost be implied from an example. The ability to specify more flexible matching like "an array of any length" that is currently missing from v1 matching will be available in v2 matching (beta is out now, see [v2 flexible matching](https://github.com/realestate-com-au/pact/wiki/v2-flexible-matching)).

### Why does Pact use concrete JSON documents rather than using more flexible JSONPaths?

Pact was written by a team that was using microservices that had read/write RESTful interfaces. Flexible JSONPaths are useful when reading JSON documents, but no good for creating concrete examples of JSON documents to POST or PUT back to a service.

### Why is there no support for specifying optional attributes?

Firstly, it is assumed that you have control over the provider's data (and consumer's data) when doing the verification tests. If you don't, then maybe Pact is [not the best tool for your situation](https://github.com/realestate-com-au/pact#what-is-it-good-for).

Secondly, if you think about it, if Pact supports making an assertion that element `$.body.name` may be present in a response, then you write consumer code that can handle an optional `$.body.name`, but in fact, the provider gives `$.body.firstname`, no test will ever fail to tell you that you've made an incorrect assumption. Remember that a provider may return extra data without failing the contract, but it must provide at minimum the data you expect.

### Why are the pacts generated and not static?

* Maintainability: Pact is "contract by example", and the examples may involve large quantities of JSON. Maintaining the JSON files by hand would be both time consuming and error prone. By dynamically creating the pacts, you have the option to keep your expectations in fixture files, or to generate them from your domain (the recommended approach, as it ensures your domain objects and their JSON representations in the pacts can never get out of sync).

* Provider states: Dynamically setting expectations on the mock server allows the use of provider states, meaning you can make the same request in different tests, with different expected responses. This allows you to properly test all the code paths in your consumer (eg. with different response codes, or different states of the resource). If all the interactions were loaded at start up from a static file, the mock server wouldn't know which response to return. See this [gist](https://gist.github.com/bethesque/7fa8947c107f92ace9a4) as an example.

### How do I test against the latest development and production versions of consumer APIs?

See [this article](http://rea.tech/enter-the-pact-matrix-or-how-to-decouple-the-release-cycles-of-your-microservices/).

### What does PACT stand for?

It doesn't stand for anything. It is the word "pact", as in, another word for a contract. Google defines a "pact" as "a formal agreement between individuals or parties." That sums it up pretty well.

### Why Pact may not be the best tool for public testing APIs?
Each interaction in a pact should be verified in isolation, with no context maintained from the previous interactions. Tests that depend on the outcome of previous tests are brittle and land you back in integration test hell, which is the nasty place you’re trying to escape by using pacts.

So how do you test a request that requires data to already exist on the provider? Provider states allow you to set up data on the provider by injecting it straight into the datasource before the interaction is run, so that it can make a response that matches what the consumer expects. They also allow the consumer to make the same request with different expected responses (eg. different response codes, or the same resource with a different subset of data).

If you use Pact to test a public API, the only way to set up the right provider state is to use the very API that you’re actually testing, which will make the tests slower and more brittle compared to the “normal” pact verification tests.

If this is still a better situation for you than integration testing, or using another tool like VCR, then go for it!

### Why Pact may not be the best tool for testing pass through APIs?
During pact verification, Pact does not test the side effects of a request being executed on a provider, it just checks that the response body matches the expected response body. If your API is merely passing on a message to a downstream system (eg. a queue) and does not validate the contents of the body before doing so, you could send anything you like in the request body, and the provider would respond the same way. The “contract” that you really want is between the consumer and the downstream system. Checking that the provider responded with a 200 OK does not give you any confidence that your consumer and the downstream system will work correctly in real life.

What you really need is a “non-HTTP” pact between your consumer and the downstream system. Check out this [gist](https://gist.github.com/bethesque/0ee446a9f93db4dd0697) for an example of how to use the Pact contract generation and matching code to test non-HTTP communications.

### Do I still need end-to-end tests?

**TL;DR: Yes**

The answer to this question depends on your organisation's risk profile. There is generally a trade off between the amount of confidence you have that your system is bug free, and the speed with which you can respond to any bugs you find. A 10 hour test suite may make you feel secure that all the functionality of your system is working, but it will decrease your ability to put out a new release quickly when a bug is inevitably found.

If you work in an environment where you prioritise "agility" over "stability", then maybe you would be better off investing the time that you would have spent maintaining end-to-end tests in improving your production monitoring.

If you work in a more traditional "Big Bang Release" environment, a carefully selected small set of end-to-end tests that focus on the core business value provided by your system should provide the confidence you need to release. Consider "Semantic monitoring" (a type of "testing in production") as an alternative.

### How can I handle versioning?

Consumer driven contracts to some extent allows you to do away with versioning. As long as all your contract tests pass, you should be able to deploy changes without versioning the API. If you need to make a breaking change to a provider, you can do it in a multiple step process - add the new fields/endpoints to the provider and deploy. Update the consumers to use the new fields/endpoints, then deploy. Remove the old fields/endpoints from the provider and deploy. At each step of the process, all the contract tests remain green.

Using a [Pact Broker](https://github.com/bethesque/pact_broker), you can tag the production version of a pact when you make a release of a consumer. Then, any changes that you make to the provider can be checked agains the production version of the pact, as well as the latest version, to ensure backward compatiblity.

If you need to support multiple versions of the provider API concurrently, then you will probably be specifying which version your consumer uses by setting a header, or using a different URL component. As these are actually different requests, the interactions can be verified in the same pact without any problems.

### Using Pact where the Consumer team is different from the Provider team
Pact is "consumer driven contracts", not "dictator driven contracts". Just because it's called "consumer driven" doesn't mean that the team writing the consumer gets to write a pact and throw it at the provider team without talking about it. The pact should be the starting point of a collaborative effort.

The way Pact works, it's the pact verification task (in the provider codebase) that fails when a consumer expects things that are different from what a provider responds with, even if the consumer itself is "wrong". This is a little unfortunate, but it's the nature of the beast.

Running the pact verification task in a separate CI build from the rest of the tests for the provider is a good idea - if you have it in the same build, someone is going to get cranky about another team being able to break their build.

It's very important for the consumer team to know when pact verification fails, because it means they cannot deploy the consumer. If the consumer team is using a different CI instance from the provider team, consider how you might communicate to the consumer team when pact verification has failed. You should do one of the following:

* Configure the pact verification build to send an email to the consumer team when the build fails.
* Even better, if you can, have a copy of the provider build run on the consumer CI that just runs the unit tests and pact verification. That way the consumer team has the same red build that the provider team has, and it gives them a vested interest in keeping it green.

Verify a pact by using a URL that you know the latest pact will be made available at. Do not rely on manual intervention (eg. someone copying a file across to the provider project) because this process will inevitably break down, and your verification task will give you a false positive. Do not try to "protect" your build from being broken by instigating a manual pact update process. The pact verify task is the canary of your integration - manual updates would be like giving your canary a gas mask.

### How to prevent a consumer from deploying with an invalid contract

Related to the previous question, is how we prevent the Consumer from releasing before ensuring the Provider also supports the contract.

As a general rule, if the contract doesn't change, there is no need to run the Provider build, albeit there is currently no easy way to know this in the Broker (see feature request https://github.com/bethesque/pact_broker/issues/48). 

Consider one or more of the following strategies:

**Collaboration**

Well, for starters, you _must_ be collaborating closely with the Provider team! 

**Effective use of code branches**

It is of course very important that new assumptions on the contract be validated by the Provider before the Consumer can be safely released. Have branches tested against the Provider before you merge into master.

**Use source control to detect a modified contract:**

If you also checked the master pact files into source control, your CI build could conditionally act - if the contract has changed, you must wait for a green provider build, if not you can safely deploy!

**Use Pact Broker Webhooks:**

Use [webhooks](https://github.com/bethesque/pact_broker/blob/master/lib/pact_broker/doc/views/webhooks.markdown) on the [Pact Broker](https://github.com/bethesque/pact_broker) to trigger a build on the Provider as soon as a new contract is submitted to the server.

### How do I test OAuth or other security headers?

For interactions such as OAuth2 defined by a standard and implemented with a library implementing that standard, we would recommend _against_ using Pact for these scenarios. Standards are well defined and don't change often, and likely you have simpler testing options available (probably something your framework provides).

For APIs that _use_ these headers, things are a little more complicated, particularly on the provider side - the side that actually needs the token to be valid. Why?

When Pact reads the pact files for verification on the Provider side, it needs to have a valid token, and if that token has been persisted in a Pact file it has probably expired.

**Here are some options**

* Create a Mock authentication service used during testing - this gives you the best control.
* If using the JVM, you can use [request filters](https://github.com/DiUS/pact-jvm/tree/master/pact-jvm-provider-gradle#modifying-the-requests-before-they-are-sent) to modify the request headers before they are sent to the Provider.
* Configure a relaxed OAuth2 validation service on the Provider that accepts any valid headers, so long as the match the spec (e.g. `Authorization` header). You might leverage the [provider states](http://docs.pact.io/documentation/provider_states.html) feature for this.
* Use Ruby's `Timecop` or similar library to manipulate the runtime clock.


*NOTE*: Any option that modifies the request before sending to the running provider increases your chances of missing a key part of the interaction and therefore puts you at risk. Use carefully.

See the following links for some further discussion:

* https://github.com/realestate-com-au/pact/issues/49#issuecomment-65346357
* https://groups.google.com/forum/#!searchin/pact-support/oauth%7Csort:relevance/pact-support/zTnDlOgdYhU/tq_Yx8MnIgAJ
* https://groups.google.com/forum/#!topic/pact-support/tSyKZMxsECk
* http://stackoverflow.com/questions/40777493/how-do-i-verify-pacts-against-an-api-that-requires-an-auth-token/40794800?noredirect=1#comment69346814_40794800

### How do I test binary files in responses, such as a download?

We suggest matching on the core aspects of the interaction - the request itself, and the response headers e.g.

```
{
   state: 'I have a picture that can be downloaded',
   uponReceiving: 'a request to download some-file',
   withRequest: {
       method: 'GET',
       path: '/download/somefile'        
   },
   willRespondWith: {
       status: 200,
       headers: 
       {
           'Content-disposition': 'attachment; filename=some-file.jpg'
       }        
   }
}
```