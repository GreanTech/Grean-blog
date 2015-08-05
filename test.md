# Webtasks for the builder: Integrate Auth0, Mailgun, Twilio
If you have managed to stay awake for just shorter periods of time in recent years, you will have noticed the proliferation and disparitemaily of software services available to the builder of applications.

These services all have APIs for you to feast on. Build an application and use a service for payments, email, PDF generation, customer feedback - and, of course, one for authentication. 

In most cases you will orchestrate and integrate these disparate services through your own application code, calling on APIs and setting up web hooks for notifications. 

But sometimes you really just want to have one service hook into the next without having to host and operate the often times fairly small pieces of integration logic. 

If that's you, read on. With the introduction of the new innovative concept of [webtasks](https://webtask.io/) you may just be able to avoid hosting your own integration logic. 

### Webtasks you say?

Webtasks, and the supporting infrastructure by Auth0, provide a very elegant way to run your code without deployment or hosting. Write your Javascript code and generate a webtask token for use with your code and the webtask infrastructure.

![Run code with an HTTP call. No provisioning. No deployment.](./webtasks.png)

You don't have to understand the more technical aspects of webtasks to read on. But if you are concerned about security, or just want to enjoy a piece of cool tech, go the [web site](webtask.io) and try it out. Start with the great [conceptual introduction](https://webtask.io/docs/how).

To prepare a profile for setting up web tasks, install and initialize the webtask command line client:

```
sudo npm install -g wt-cli
wt profile init --profile sms-profile your@email.com
```

## Make Auth0 work with SMS for password reset

In order to illustrate the the concept of "hands free" integration, consider extending Auth0's capabilities for account verification and password reset with text messaging, SMS:

- Your web site allows users to sign up with a username and password of their own choosing. 
- Your users may provide either an email or a mobile phone number for password reset.

The _database connection_ (a built-in account store) in Auth0, though, is built around email as the vehicle of communication.  So we need a way to let Auth0 do email, unknowingly, while converting some of these emails into text messages (SMS).

## Auth0, Mailgun, Twilio - and Webtasks - play nice

The image below illustrates how Auth0, Mailgun, and Twilio should be hooked up in order to provide the SMS capabilities. The numbers reference the list below, and the red, dotted lines are those that, as we will see, require webtasks. 

![Service orchestration](./services.png) 

1. If a user registers with a phone number instead of an email, create an Auth0 user account with an email address formatted something like this: `acme+4512345678@sms.mydomain.com`. The plus sign and the digits form the embedded, in this case Danish, phone number. 
2. Auth0 sendes out account and password verification emails as always, not aware of these pseudo email addresses
3. Mailgun is configured with a the domain `sms.mydomain.com` and a dedicated route to intercept and store messages to these specialized email addresses
4. When intercepted Mailgun _should_ extract the phone number from the `To` field of the email, generate an appropriate text, and use Twilio to send out the text to the extracted phone number
5. Twilio sends out the message, e.g. "_Respond with YES if you have just set up an account with ACME Inc._"
6. The user receives the text message, and immediately responds as instructed
7. Twilio receives the message and _should_ now locate the corresponding original Auth0 email message, extract the verification link, and issue a GET request to that link.
8. Once Auth0 receives the verification request, the new account, or the new password, is activated

## Pseudo email addresses and Mailgun routing
Our application registers a new user through the Auth0 API. As the user has only provided a mobile phone number, the application constructs a pseudo email address from the phone number following a predefined format.

Auth0 will send out a verification email to the provided email address constructed to contain the phone number. The email is sent to a Mailgun managed domain, in our example `sms.mydomain.com`. This domain is then set up with a Route as illustrated below.

![Mailgun route](./RoutesMailgun.png)

The important part is the routing action where the webtask is activated. On a recipient match, the action stores the message and triggers a post of the entire message to our webtask.

## Webtask'ing the Mailgun to Twilio integration 

As indicated in the above diagram, step 4 must extract the number, and send a message via Twilio. As Mailgun supports only a simple webhook, and Twilio has no direct API hookable straight from Mailgun, a web task is introduced to extract the phone number from the email recipient field, and then send an SMS through the Twilio API.

A webtask receives both its configuration data and the posted data in the context object (as well as any query string parameters).

```javascript
// Auth0's web task infrastructure supports a wide range of Node.js modules
var twilio = require('twilio'); 

module.exports = function(context, callback) {
    var twilio_sid = context.data.twilio_sid;
    var twilio_key = context.data.twilio_key;
    var twilio_number = context.data.twilio_number;
    var pwd_sms_body = context.data.pwd_sms_body;
    var reg_sms_body = context.data.reg_sms_body;

    var phone = phoneFromEmail(context.data.To);
    
    // ... 
    
    var client = new twilio.RestClient(twilio_sid, twilio_key);
    client.messages.create({
        to: phone,
        from: twilio_number,
        body: registration ? reg_sms_body : pwd_sms_body,
    }, function(err, message) { /* ... */ }
    
    // ... 
    callback(null);
}
```

This code is then fed to the webtask command line tool, `wt` to install the code together with the configuration parameters and an access token.

```
wt create SendVerificationSMS.js -p sms-profile \
    --secret twilio_key=<SECRET KEY> \
    --secret twilio_sid=<SUBSRIPTION ID> \
    --param twilio_number=<PHONE NUMBER> \
    --param pwd_sms_body="Your password has changed. Reply with YES to verify" \
    --param reg_sms_body="You have registered for an account. Reply YES to verify"
```

This will generate a URL that you can use to insert into Mailgun: `https://webtask.it.auth0.com/api/run/wt-your-email_com-0/sendverificationsms?webtask_no_cache=1` as shown in the above image from the Mailgun route configuration.

## Twilio and back to Auth0

Twill sends out the message as per the API request. The user may then choose to respond in which case Twilio simply relays the response to a webhook configured as shown here: 

![Twilio number](./TwilioNumbers.png)

But again, once the user replies with a "YES", Twilio is not able to dig out the original email message, locate the verification link and ""click"" it.

To achieve this, we again introduce a webtask, this time inserted as the Twilio SMS webhook.

## Webtask'ing the Twilio to Auth0 integration

This webtask is more complicated as Twilio carries no state between the original message and the user's reply. Therefore the webtask searches to find the original message as stored by the route in Mailgun. 

Note that for this example we didn't want to introduce more services than was needed to deliver the functionality. But it is, of course, possible to have the first webtask, `SendVerificationSMS.js` store the message with a dedicated storage service instead of inside Mailgun's somewhat search unfriendly message store.

(The webtask infrastructure offers no persistence, every webtask execution is from the same starting point.)

The `ReceiveVerificationSMS.js` should work like this:

```javascript
var request = require('request');
var htmlparser = require('htmlparser2');

// matches e.g. <acme+4512345678@sms.mydomain.com> and captures the phone number
var phone_pattern = new RegExp(/acme.*(\+\d+).*@.+/);

module.exports = function(context, req, res) {
    var mailgun_key = context.data.mailgun_key;
    var mailgun_domain = context.data.mailgun_key;
    var correct_sms_response = context.data.correct_sms_response.toLowerCase();
    var wrong_sms_message = context.data.wrong_sms_message;
    var reset_valid_hours = context.data.reset_valid_hours ? parseInt(context.data.reset_valid_hours) : 8;

    if (context.data.Body.toLowerCase() != correct_sms_response) { /* ... */ }
    else {
        var sender = context.data.From;
        var mgBaseUrl = 'https://api.mailgun.net/v3/' + mailgun_domain + '/';

        // 
        // Get the latest stored events from Mailgun, sorted with the most recent first
        // Get the message from the first match on phone number
        //
        var now = new Date();
        var beginDate = new Date(now.getTime() - 1000*60*60*reset_valid_hours);
        mailgunRequest(mgBaseUrl + 'events', 'GET', { 'event': 'stored', 'begin': now.toUTCString(), 'end': beginDate.toUTCString() },
            // ... find event matching the phone number
                        mailgunRequest(msgUrl, 'GET', {},
                            function (error, response, body) {
                                if (!error && response.statusCode == 200) {
                                    var mailMsg = JSON.parse(body);
                                    clickActivationLink(mailMsg['body-html']);
                                    mailgunRequest(msgUrl, 'DELETE', {}, function (error, response, body) { /* ... */ });
                                }
                            })
            // ... 
        );
    }


    function clickActivationLink(bodyhtml) { /* ... */ }

    function mailgunRequest( url, verb, qsParams, callback ) { /* ... */ }

    function phoneFromEmail(email) { /* ... */ }
}
```

Like before, also this code is  fed to the webtask command line tool, `wt`:

```
wt create ReceiveConfirmationSMS.js -p sms-profile \
    --secret mailgun_key=key-<SECRET_KEY> \
    --secret mailgun_domain=sms.mydomain.com \
    --param reset_valid_hours=7 \
    --param correct_sms_response="YES" \
    --param wrong_sms_message="Try again. Reply YES or don't reply"
```

This command will generate a URL to insert into Twilio as a web hook for receiving messages as shown in the above screen shot from a Twilio configuration page: 

`https://webtask.it.auth0.com/api/run/wt-your-email_com-0/receiveconfirmationsms?webtask_no_cache=1` 

## Done - but is this ready for production?

The above illustrates primarily the power of webtasks. Still this concept is used in production with a large Scandinavian insurance company. To achieve a robust and separated experience across environments, separate webtask containers are used with separate _webtask tokens_.  (Also, naturally, the Javascript code is a more robust, fault tolerant version)

_About Grean. Grean provides an Authorization Management service tightly integrated with Auth0. For more please see [grean.com](http://grean.com)_
