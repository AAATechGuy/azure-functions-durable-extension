# Human Interaction & Timeouts
## Phone Verification
This sample demonstrates how to build a function orchestration that involves human interaction. Whenever a real person is involved in an automated process, the process must be able to send notifications to the person, receive responses back asynchronously from the person, and account for the person being unavailable (and this last part is where timeouts become important).

This sample in particular implements an SMS-based phone verification system. These types of flows are often used when verifying a customer's phone number of when incorporating multi-factor authentication (MFA) into an online authentication system. This is a powerful example because the entire implementation is done using a couple small functions and does not require any external data store, like a database.

## Before you begin
If you haven't done so already, make sure to read the [overview](~/articles/overview.md) before jumping into samples. It will really help ensure everything you read below makes sense.

All samples are combined into a single function app package. To get started with the samples, do the following:

1. Create a new function app at https://functions.azure.com/signin.
2. Follow the [installation instructions](~/articles/installation.md) to configure Durable Functions.
3. Download the [DFSampleApp.zip](~/files/DFSampleApp.zip) package.
4. Unzip the sample package file into `D:\home\site\wwwroot` using Kudu or FTP.

This article will specifically walk through the following function in the sample app:

* **E4_SmsPhoneVerification**
* **E4_SendSmsChallenge**

> [!NOTE]
> This walkthrough assumes you have already gone through the [Hello Sequence](./sequence.md) sample walkthrough. If you haven't done so already, it is recommended to first go through that walkthrough before starting this one.

## Scenario overview
Phone verification is an important technique used to 1) verify that end users of your application are not spammers and 2) to verify that users of your application are who they say they are. Multi-factor authentication is a common use case for protecting user accounts from hackers. The challenge with implementing your own phone verification is that it requires a **stateful interaction** with a human being. An end user is typically provided some code (e.g. a 4-digit number) and is expected to respond back to the system **in a reasonable amount of time** to complete the operation.

Ordinary Azure Functions are stateless (as are many other cloud endpoints on other platforms), so these types of interactions will involve managing state externally in a database or some other persistent store. In addition, the interaction must be broken up into multiple functions that can be coordinated together. For example, you need at least one function for deciding on a code, persisting it somewhere, and sending it to the user's phone. Additionally, you need at least one other function to receive a response from the user and somehow map it back to the original function call in order to do the code validation. A timeout is also an important aspect to ensure security. You can imagine that this can get fairly complex pretty quickly!

The complexity of this scenario is greatly reduced when using Durable Functions. As you will see in this sample, an orchestrator function can manage the stateful interaction very easily and without involving any external data stores. Because orchestrator functions are *durable*, these interactive flows are also highly reliable.

## Configuring Twilio integration
This sample involves using the [Twilio](https://www.twilio.com/) service to send SMS messages to a mobile phone. Azure Functions already has support for Twilio via the [Twilio binding](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-twilio), and we'll make use of that here.

The first thing you you will need is a Twilio account. You can create one free at https://www.twilio.com/try-twilio. Once you have an account, you'll need to add the following three **app settings** to your project.

| App Setting Name | Value Description |
| - | - |
| **TwilioAccountSid**  | The SID for your Twilio account |
| **TwilioAuthToken**   | The Auth token for your Twilio account |
| **TwilioPhoneNumber** | The phone number associated with your Twilio account. This is used to send SMS messages. |

## The counter orchestration
The **E4_SmsPhoneVerification** function uses the standard function.json for orchestrator functions.

[!code-json[Main](~/../samples/csx/E4_SmsPhoneVerification/function.json)]

Here is the code which implements the function:

[!code-csharp[Main](~/../samples/csx/E4_SmsPhoneVerification/run.csx)]

This is the most lines of code of all the samples, so let's walk through it carefully. Once started, this orchestrator function does the following:

1. Gets a phone number to which we will *send* the SMS notification.
2. Calls **E4_SendSmsChallenge** to send an SMS message to the user and returns back the expected 4-digit challenge code.
3. Creates a durable timer which triggers 90 seconds from the current time.
4. In parallel with the timer, waits for a **SmsChallengeResponse** event from the user.

At this point, the user should have received an SMS message with a four digit code. They have 90 seconds to send that same 4-digit code back to the orchestrator function instance to complete the verification process. If they submit the wrong code, they get an additional three tries to get it right (within the same 90 second window).

> [!NOTE]
> It may not be obvious at first, but this orchestrator function is completely deterministic. This is because the <xref:Microsoft.Azure.WebJobs.DurableOrchestrationContext.CurrentUtcDateTime> property is used to calculate the timer expiration time, and this property returns the same value on every replay at this point in the orchestrator code. This is important to ensure we get the same `winner` result in every repeated call to `Task.WhenAny`.

> [!WARNING]
> It's very important to cancel timers using a `CancellationTokenSource` if we no longer need them to expire, like we do in the examle above when a challenge response is accepted.

For more information on timers, see the [Durable Timers](../topics/timers.md) topic.

## Sending the SMS message
The **E4_SendSmsChallenge** function uses the Twilio binding to send the SMS message with the 4-digit code to the end user. The function.json is defined as follows:

[!code-json[Main](~/../samples/csx/E4_SendSmsChallenge/function.json)]

And here is the code which generates the 4-digit challenge code and sends the SMS message:

[!code-csharp[Main](~/../samples/csx/E4_SendSmsChallenge/run.csx)]

One important behavior detail here is that this **E4_SendSmsChallenge** function only gets called once, even if the process crashes or gets replayed. This is good because you don't want the end user getting multiple SMS messages. The `challengeCode` return value is automatically persisted, so the orchestrator function always knows what the correct code is.

## Running the sample
Using the HTTP-triggered functions included in the sample, you can start the orchestration using the below HTTP POST request.

```plaintext
POST http://{app-name}.azurewebsites.net/orchestrators/E4_SmsPhoneVerification
Content-Length: 14
Content-Type: application/json

"+1425466XXXX"
```
```plaintext
HTTP/1.1 202 Accepted
Content-Length: 260
Content-Type: application/json; charset=utf-8
Location: http://{app-name}.azurewebsites.net/orchestrations/71cd5995eee5414fb42b58ee38351892

{"id":"71cd5995eee5414fb42b58ee38351892","pollUrl":"http://{app-name}.azurewebsites.net/orchestrations/71cd5995eee5414fb42b58ee38351892","sendEventUrl":"http://{app-name}.azurewebsites.net/orchestrations/71cd5995eee5414fb42b58ee38351892/SendEvent/{eventName}"}
```

The orchestrator function will receive the supplied phone number and immediately send it an SMS message with a randomly generated 4-digit verification code - e.g. *2168*. The function will then wait 90 seconds for a response.

To reply with the code, you can use <xref:Microsoft.Azure.WebJobs.DurableOrchestrationClient.RaiseEventAsync*> inside another function or invoke the **sendEventUrl** HTTP POST webhook referenced in the 202 response above, replacing `{eventName}` with the name of the event, `SmsChallengeResponse`:

```plaintext
POST http://{app-name}.azurewebsites.net/orchestrations/71cd5995eee5414fb42b58ee38351892/SendEvent/SmsChallengeResponse
Content-Length: 4
Content-Type: application/json

2168
```

If sent before the timer expires, you should see that the orchestration has completed and that the `output` field is set to `true`, indicating a successful verification.

```plaintext
GET http://{app-name}.azurewebsites.net/orchestrations/71cd5995eee5414fb42b58ee38351892
```
```plaintext
HTTP/1.1 200 OK
Content-Length: 144
Content-Type: application/json; charset=utf-8

{"runtimeStatus":"Completed","input":"+1425466XXXX","output":true,"createdTime":"2017-05-08T20:10:44Z","lastUpdatedTime":"2017-05-08T20:12:07Z"}
```

Alternately, if you let the timer expire (willingly or unwillingly), or if you entered the wrong code four times, you should be able to query for the status and see a `false` orchestration function output, indicating that phone verification failed.

```plaintext
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 145

{"runtimeStatus":"Completed","input":"+1425466XXXX","output":false,"createdTime":"2017-05-03T21:28:42Z","lastUpdatedTime":"2017-05-03T21:30:29Z"}
```

## Wrapping up
At this point, you should have a better understanding of some of the advanced capabilities of Durable Functions, notably <xref:Microsoft.Azure.WebJobs.DurableOrchestrationContext.WaitForExternalEvent*> and <xref:Microsoft.Azure.WebJobs.DurableOrchestrationContext.CreateTimer*>, and how these can be combined with `Task.WaitAny` to implement a reliable timeout system (which is often useful when doing any interaction with real people).