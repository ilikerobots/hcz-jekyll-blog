---
layout: post
title:  "Testing Amazon Alexa skills with Chai and fixtures"
date:   2018-01-26  02:00:00
categories: javascript, software 
---

# Testing Amazon Alexa skills with Chai and fixtures

Hello, my name is Mike, and I developed my first Alexa skill without writing any test cases. With that confession out 
of the way, I will defend my developer honor to some degree: I did perform testing, but it was a clumsy affair using 
Amazon's testing support from withing the Lambda management console.  While those tools are good, this process greatly
slowed my development cycle, as running tests meant repeatedly deploying my Lambda skill and manually executing test 
cases.

After my first skill, [Estonia One Hundred](https://www.amazon.com/StarHeight-Media-Estonia-One-Hundred/dp/B0796Z22K2/) ([source](https://github.com/ilikerobots/estonia-one-hundred-alexa)), 
was certified, I realized I needed a better solution for testing.  

There are several [testing strategies suggested by Amazon](https://developer.amazon.com/docs/custom-skills/test-a-custom-skill.html) 
and several articles on Medium and elsewhere on the Internet.  However, none met my needs exactly. I wanted
 1) tests to be executed locally
 1) test cases built quickly by "recording" actual interactions

I describe my custom solution below.

Note: I refer to _Alexa_ throughout the article as if she were a sentient actor in the computational processes discussed.
This is just a convenient shorthand. I'm well aware she, I mean, _it_, is a software construct and I definitely don't 
wish her good morning or ask her if she missed me while I was at work.

# Pre-requisites

This article assumes your skill was developed with similar setup as mine: specifically a Node.js implementation using
the Alexa SDK v1.0+.  While my skill utilizes an endpoint served by Amazon Lambda, I believe the strategy described 
should work identically if served elsewhere.

# Obtaining text fixtures

Interaction with an Alexa skill endpoint is stateless, which is why Lambda is such a suitable home for these endpoints.  
With each pertinent interaction with a user, Alexa sends JSON containing both  ```session``` and ```request``` data 
(plus a few other details) to the skill endpoint.  This simple structure makes static test fixtures very effective.  We 
can model any program state and action with a single JSON structure.

But, hand coding each JSON fixture would be time consuming.  Luckily, Amazon provides an easy way to capture 
Service Request JSON which can quickly be converted to fixtures.  From the Test tab of your Development Skill,
the Service Simulator can be used to perform interactions with an already deployed skill.   This simulator is 
session-friendly, in the sense that multiple utterances can be provided and they will be linked into the same session.
Simply enter your utterances so as to put your skill in its proper state (more accurately, to set ```session``` attributes
to their proper value), and copy and paste the displayed Service Request JSON.  You've just captured a test fixture.


The [Alexa Skills Kit CLI can also be used to generate Service Request JSON](https://developer.amazon.com/docs/smapi/ask-cli-command-reference.html#simulate-command) 
for test fixtures.  Using the ```ask simulate``` command will send a Service Request to a deployed
endpoint.  The original Service Request is returned in the deubgging information printed after the request.   This tool 
is especially useful for generating fixtures for launch intents and single interaction phrases, neither of which the web-based 
Service Simulator is capable of doing.

## Test first?

There's a bit of a chicken and egg situation when generating test fixtures this way.  A version of the skill endpoint 
sufficiently close to generate a desired response must already be deployed.  This could be seen as violating a 
test-first policy.  A developer deeply concerned about this can always craft text fixtures manually.  However, in practice,
deploying a simple skeletal skill that includes key states and intents can be used to generate a fixture template.
 Once produced, subsequent test fixtures can be derived incrementally from this template (as described in the final section of this
 article).


# Representing fixtures

Once you've obtained fixture, there are many ways you might wish to organize them in your codebase.  I first chose a 
simple strategy of defining a module for each state I wish to test.  An excerpt:

```ecmascript 6
const Skill = require('../index.js');
const SESSION_ID = "SessionId.f9e6dcbb-b7da-4b47-905c.etc.etc";
const USER_ID = "amzn1.ask.account.VO3PVTGF563MOPBY.etc.etc";

module.exports = {
    GET_PHRASE: {
        "session": {
            "new": false,
            "sessionId": SESSION_ID,
            "application": {"applicationId": Skill.APP_ID},
            "attributes": {
                "STATE": "_PHRASE_MODE"
            },
            "user": {"userId": USER_ID}
        },
        "request": {
            "type": "IntentRequest",
            "requestId": "request5678",
            "intent": {
                "name": "EstonianPhraseIntent",
                "slots": {}
            },
            "locale": "en-US",
            "timestamp": "2018-01-06T14:25:57Z"
        },
        "version": "1.0"
    },
    REPEAT_PHRASE_ZERO_AFTER_FIRST_PHRASE: {
        "session": {
            "new": false,
            "sessionId": SESSION_ID,
            "application": { "applicationId": Skill.APP_ID},
            "attributes": {
                "STATE": "_PHRASE_MODE",
                "phrases_used": [
                    0
                ]
            },
            "user": { "userId": USER_ID}
        },
        "request": {
            "type": "IntentRequest",
            "requestId": "EdwRequestId.da2fc3ab-5fe9-46ff-b6d8-779a90d567d0",
            "intent": {
                "name": "AMAZON.RepeatIntent",
                "slots": {}
            },
            "locale": "en-US",
            "timestamp": "2018-01-27T14:21:41Z"
        },
        "version": "1.0"
    }
};
```

As you can see, I trimmed some non-essential parts of the JSON Service Request and also externalized some 
characteristics, such as ```APP_ID``` and ```USER_ID```.  Neither steps are necessary, but I felt these improved 
conciseness and readability.


# Testing with fixtures

I utilized a [mocha](https://mochajs.com)/[chai](http://chaijs.com) combination for testing.  There are plenty of guides
to setting this up for a Node.js project, so instead I'll focus instead on the test cases themselves.

The basic strategy is to load one or more test fixtures, execute our skill handler with a fixture, then
asynchronously verify desired aspects of the response.  

The asynchronous skill handler function we will test has signature ```function (event, context, callback)```.  The 
```context``` parameter is a structure that defines ```succeed``` and ```fail``` handlers.  By building a custom 
```context``` structure containing our test assertions, we can define custom expectations for each execution of our 
handler.

First, we build a helper function that will take a test function and return a context structure that utilizes this test
callback.  We also include mocha's ```done``` callback, so that we can notify mocha that our test has completed, whether 
successful or not.

```ecmascript 6
const lambdaContext = function (test_func, done) {
    return {
        'succeed': function (data) {
            if (DEBUG_RESPONSE) {
                console.log(JSON.stringify(data, null, '\t'));
            }
            test_func(data);
            done()
        },
        'fail': done
    };
};
```

We can see above that if our handler succeeds, the handler's response will be verified against our given test function. 
We then notify mocha that our test has completed.  In the case our handler fails outright, we call mocha's ```done``` 
callback immediately, passing our error message.


With our helper in place, our test cases need only call our skill handler with a fixture and a test context constructed 
using the above helper:

```ecmascript 6
const Skill = require('../index.js'); 
const expect = require("chai").expect;
const fixtures = require('../fixtures/phrase_mode_fixtures.js');

describe('PhraseMode', function () {
    describe('Get new phrase intent', function () {
        it('Repeat phrase after first hearing', function (done) {
            Skill['handler'](fixtures.REPEAT_PHRASE_ZERO_AFTER_FIRST_PHRASE, lambdaContext(function (data) {
                expect(data.sessionAttributes.STATE).to.equal("_PHRASE_MODE");
                expect(data.sessionAttributes.phrases_used).to.be.an('array').of.length(1).that.includes(0);
                expect(data.response.outputSpeech.ssml).to.contain("Thank you");
                expect(data.response.outputSpeech.ssml).to.contain("<audio src=");
            }, done));
        });
    });
});
```

Here we test that two session attributes are set to expected values, and that the response speech contains expected 
SSML markup. Already our tests are looking fairly concise and readable.

# Dynamic fixtures

One shortcoming of the above test cases is that the test is not fully descriptive.  The test specifies a
fixture of ```REREPEAT_PHRASE_ZERO_AFTER_FIRST_PHRASE```, but the reader must either take it for granted that the test
is set up as expected or head off to another file to inspect.  It would be much more descriptive if we could specify 
the salient aspects of the test fixture directly in the test case.

Luckily, since most of our Service Requests will be very similar and consist of much boilerplate code, we can easily
create a generalized Service Request fixture factory:
```ecmascript 6
intentRequestFixture: function(attributes, intent) {
    return {
        "session": {
            "new": Object.keys(attributes).length === 0,
            "sessionId": SESSION_ID,
            "application": {"applicationId": Skill.APP_ID,},
            "attributes": attributes,
            "user": {"userId": USER_ID}
        },
        "request": {
            "type": "IntentRequest",
            "requestId": "request5678",
            "intent": {
                "name": intent,
                "slots": {}
            },
            "locale": "en-US",
            "timestamp": "2018-01-06T14:25:57Z"
        },
        "version": "1.0"
    };
}
```

Then, using this factory, we can rewrite our test above, specifying only our current attributes and new intent, as:
```ecmascript 6
it('Repeat phrase after first hearing', function (done) {
    Skill['handler'](fixtures.intentRequestFixture({
            "STATE":"_PHRASE_MODE", 
            "phrases_used": [0],
        }, "AMAZON.RepeatIntent"),
        lambdaContext(function (data) {
        expect(data.sessionAttributes.STATE).to.equal("_PHRASE_MODE");
        expect(data.sessionAttributes.phrases_used).to.be.an('array').of.length(1).that.includes(0);
        expect(data.response.outputSpeech.ssml).to.contain("Thank you");
        expect(data.response.outputSpeech.ssml).to.contain("<audio src=");
    }, done));
});
```

Now the cases are much more readable.  Additionally, it becomes very easy to derive additional test cases without
the need to obtain new fixtures.


# Conclusion

The strategy described above allows Amazon Alexa skills to be tested locally with easily generated and adapted
text fixtures.  More improvements could certainly be made (e.g. testing other locales), but, as is, you'll now 
have to try much harder to make excuses for not writing test cases for your Alexa skill!

# Source

These articles were based on work I did creating [Estonia One Hundred](https://www.amazon.com/StarHeight-Media-Estonia-One-Hundred/dp/B0796Z22K2/), a 100th birthday gift to a country I love, Estonia.  See the [full source code of Estonia One Hundred](https://github.com/ilikerobots/estonia-one-hundred-alexa).





