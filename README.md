CxEngage Documentation
=============

Getting Started 
---------------

CxEngage accepts events from various systems, allowing you to stitch them together to understand and respond to customer's journeys as they happen. Before you dive in and start creating customer journey patterns though, there is a bit of work that you'll need to do to get events into the system.

To begin, if you would like to integrate with any 3rd-party systems like SendGrid or DataSift, you'll need to create an account with them first. We come pre-integrated with a number of solutions, but if there is one that you use that isn't available, don't hesitate to reach out to us. We're creating new integrations every week, and we will prioritize our work based on customer feedback.

For any custom system, you can submit events into our API. 

## Setting up your patterns

Here is where we start creating patterns for things that matter to your enterprise. To enable this we have a simple, yet powerful DSL.

### Intro to the CxEngage DSL

CxEngage's customer Domain-Specific Language (DSL) offers incredible flexibility, providing the ability to set up almost any type of pattern of events to match. At it's most basic, every pattern is of the form: **when** _"events happen"_ **then** _"notify or do something"_.

### "Whens"

The following keywords are available to describe the **"when"** portion of the pattern:

```
When keywords
 = 
 >
 <
 not
 and
 or
 within
 seq
 all
 any
 count
 fail
 seconds
 minutes
 hours
 days
```


For example, the following is a pattern where if a customer calls into a contact center twice and abandons the call twice within an hour, then we would call the customer back:

```clojure
;;When
(within 1 hours
        (count 2 (event (= CallAction "abandoned")))
        
;Then
(send echo message {:message "Call abandoned twice, call customer back"})
```
_(Echo is used for convenience in this case to illustrate the example, but is also useful for testing your patterns.)_

Now, what if a customer calls a 3rd time and gets an agent before the agent calls back? Since we don't want the agent to call him or her back, we can account for this scenario by using the fail keyword as follows: 

```clojure
;;When
(all (within 1 hours
                 (count 2 (event (= CallAction "abandoned")))))
                 (fail (count 1 (event (= CallAction "answered"))))
       
;;Then
(send echo message {:message "Call abandoned twice , call customer back"}))
```



As another example, what if when a customer calls in once, talks to an agent, and then calls back within 30 minutes, we want the customer to be routed to a more experienced second-tier agent? That pattern would look like this:

```clojure
;;When
(within 30 minutes
        (seq (event (= CallAction "answered")
                    (event (= CallAction "inqueue"))))
                    
;Then
(send echo message {:message "Transfer call to senior agent"})
```

#### Writing "Whens" 

Let's start with trying to write a simple expression. A pattern I use a lot is of the form when CustomerSegment is Platinum, do "something". The pattern for that is written this way 

```clojure
(event (= customerSegment "platinum"))
```

* All CxEngage DSL expressions are of the form - When trigger Then
* So, let's go through all the keywords from the expression above
   1.  When keyword - This is what is used to put in before the expression you want to match
   3.  event - this keyword just means event
   5.  = - this keyword or symbol just means equal
   6.  the values of the events that the pattern is looking for is shown here
   7.  then - this keyword is used before the reaction you would like the system to perform

Now, let's say you want the pattern to match for anyone with customer Segment Platinum with a failed check-in, you will write the expression pretty much the same way but add another event in the list. 

```clojure
(event (and (= customerSegment "platinum")
            (= event "flcheck")))
```
As you can see, we use the **and** keyword to do this. Here is a list of those type of operators

``` 
 =             
 >
 <
 Not
 And
 Or
```

Ok, now let's say if we would like the pattern to match if two events happen. For example, a pattern that matches if a person with Customer Segment platinum has 2 failed checkin events. To do this, all we need to do is use the **count** keyword

```clojure
(count 2 (event (and (= customerSegment "platinum")
                     (= event "flcheck")))) 
```
The other option for items similar to the count keyword are 

```
 within      Within Duration Trigger 
 seq         Sequence [Trigger]    
 all         All [Trigger]         
 any         Any [Trigger]          
 count       Count Int Trigger       
 fail        Fail Trigger        
```


Now, if we want to add another event type to the pattern, for example, you would like the pattern to match when there is 2 failed checkins and 1 cancelled ticket. We use the **allOf** keyword for this. The Pattern is written this way - 
```clojure
(all (count 2 (event (and (= customerSegment "platinum") 
                            (= eventType "flcheck")))) 
       (count 1 (event (and (= customerSegment "platinum") 
                            (= eventType "cnclticket")))))
```

Now, for this to be more useful, we would only want this to happen for a particular time window, for example, 2 failed checkins and 1 cancelled ticket in the last hour. To do this we use the **within** keyword and one of the following duration keywords, **hour**. 

```clojure
(within 1 hours 
        (all (count 2 (event (and (= customerSegment "platinum") 
                                    (= eventType "flcheck")))) 
               (count 1 (event (and (= customerSegment "platinum") 
                                    (= eventType "cnclticket"))))))
```

The options for duration are the following. The unit used is integer

```
 Seconds       
 Minutes       
 Hours        
 Days          
```

Now, if we want the pattern to only match if a cancelled ticket happens after a failed check-in, we can use the **seq** keyword. The previous pattern would match if there is a cancelled ticket first and then 2 failed check ins. We only want the pattern to match if a cancel ticket event happens after a failed check in event

```clojure
(all (seq (event (= eventType "flcheck")) 
                   (event (= eventType "cnclticket"))))
```

All the more commonly used operators are shown being used above. You can use other keywords such as **fail** etc similarly. 

### "Thens"

The following keywords are available to describe the **"then"** portion of the pattern:

```
    Then keyword
    par        
    seq        
    delay      
    send       
    if         
    success
    failure 
    timeout    
    template 
    set 
```

Continuing the examples from above, we can replace the _echos_ with the above _"thens"_. Now, when a caller abandons the call twice, we can use our Twilio integration to trigger an outbound call:

```clojure
;;When
(all (within 1 hours
               (count 2 (event (= CallAction "abandoned"))))

;;Then
(send twilio call {:to-phone-number *phone-number*}))
```

If we would like to send an SMS to a customer after the two abandons to let them know they will receive a call shortly, we can write the pattern using _seq_ (short for _sequence_) as: 

```clojure
;;When
(all (within 1 hours
               (count 2 (event (= CallAction "abandoned"))))
                
;;Then
(seq
  (send twilio sms {:to-phone-number *phone-number*
                    :message "We apologize for the long waits, an agent
                              will call you back shortly"})
  (send twilio call {:to-phone-number *phone-number*}))
```

In this example, we use the _seq_ keyword because we want the SMS to go out before the agent calls
back. One thing to note here is that if the SMS fails, the next send does not execute.

### Execution Blocks

Reactions are formed from two primitive building blocks: **par** and **seq**.
These two keywords describe the method in which their children should be evaluated.
Both of the execution blocks can be nested within one another.

#### par

**par** blocks will evaluate in parallel. One must be conscious of the repercussions of
evaluating their blocks in parallel. Results will be returned in a non-deterministic manner,
thus all members of the parallel block should be independent of one another. Also to note is
that if one member of the block fails, the entire block fails. If a retry is defined, the entire
block will be resent regardless of existing inflight invocations.

```clojure
; Syntax
(par <members> <options>)

; Send three messages to echo in parallel
(par
  (send echo message {:message "Hello World1"})
  (send echo message {:message "Hello World2"})
  (send echo message {:message "Hello World3"}))
```


#### seq

**seq** blocks will be evaluated sequentially. As the sequential block is executed in a deterministic
manner, it is safe to create dependencies between elements (in a sequential manner). As with the parallel
block, the block will fail if the current member fails. On failure, if a retry is defined, the sequential
block will restart from its first member.

```clojure
; Syntax
(seq <members> <options>)

; Send one message to echo, delay one minute, then send another message to echo
(seq
  (send echo message {:message "Hello World1"})
  (delay 1 minutes)
  (send echo message {:message "Hello World2"}))
```

### Variables

The Notification DSL supports a simple form of variables and assignment. All variables defined in
a reaction are global in scope. This means they can be accessed by any member of the reaction after
definition.

Values can come from one of four sources in the Notification DSL:
* Canonical Event Record
* Endpoint Response
* Reaction Variable
* Constant

Value access of the above types is done as follows:
```clojure
; Access the value custId, in the canonical event record
*custId*

; Access the value call-id, in an endpoint response
$call-id

; Access the value success-call, in the reaction
success-call

; Use a constant value.
; Constants can be strings, boolean, or numbers
"gold"
42
false
```

User defined variables can be created using the **set** command, which can be nested within execution blocks.

```clojure
; Create a variable message-sent, using the call-id value from an endpoint response
(set message-sent $call-id)

; Create a variable cust-seg, using a constant
(set cust-seg "gold")

; Create a variable id, using the custId value in the canonical event record
(set id *custId*)
```

### Actions

Within an execution block the following actions can be taken:

#### send

The **send** command is used to send notifications to specific endpoints. The endpoint, type, and params are
mandatory fields. Additional options will are outlined in the options section. The params must include all
mandatory fields outlined in the definition of the endpoint. A single send may be used outside of an execution block. This can be useful when using the **if** command.

```clojure
; Syntax
(send <endpoint> <type> {<params>} <options>)

; Send a message notification to the echo endpoint, with the message parameter
; mapped to the constant value "Hello World"
(send echo message {:message "Hello World"})

; Send an sms message to the twilio endpoint, with to-phone-number from the event record;
; from-phone-number as a constant; and message from a reaction variable.

(send twilio sms {:to-phone-number *phone-number*
                  :from-phone-number "1-506-555-1234"
                  :message sms-message})
```



#### event

The **event** command will create and send a new event into the rest receiver. This functionality is useful
for creating chains of events. Events support message templates and the control flow operators: **success** and **failure**.

```clojure
; Syntax
(event {<params>} <options>)

; Send a new event with the same key attr (id) and the type "b"
(event {:id *id* :type "b"})
```



#### delay

The **delay** command will delay further evaluation of a reaction. This delay respects the execution block
in which it is evaluated. In a sequential block, it will block further evaluation until the delay expires.
In a parallel execution block the delay is evaluated in parallel to the other members of the block, thus
causing the delay to act as a *minimum evaluation time* command rather than a strict delay.

Valid time durations are:
* seconds
* minutes
* hours
* days

```clojure
; Syntax
(delay <duration> <unit>)

; Create a delay of five minutes
(delay 5 minutes)

; Create a delay of 30 seconds
(delay 30 seconds)
```


#### expect

The **expect** command will pause execution of the reaction until an event enters the notification service which matches the provided predicate. It is recommended to use a **within** when using **expect**, otherwise the reaction may never finish.

```clojure
; Syntax
(expect <predicate>
  (success <then>)
  (failure <then>)
  <options>)

; Wait 5 minutes for an event with type "b" and the current id
(expect (and (= $id *id*)
             (= $type "b"))
  (within 5 minutes)
  (success (set got-event true))
  (failure (set got-event false)))
```


#### await

The *await* command is used to perform asynchronous communication with an endpoint. In truth,
*await* is syntactic sugar to perform a poll against a command in an endpoint. The body of the
*await* must consist of an *if* statement, whose predicate represents what you are waiting for.
The *success* and *failure* within the *if* will be executed based on the final outcome of
the await. A *within* option must also be present to denote the total time the poll should last.
The poll cycle is hard coded to wait one second between polls.

```clojure
(seq
  (await twilio call-status {:sid 12345}
    (if (= "answered" $call-status)
        (success (set msg "The call was answered"))
        (failure (set msg "No one was there")))
    (within 10 minutes))
  (send twilio sms {:message msg ...}))
```



#### if

The **if** command is the most complex in the notification DSL.

```clojure
; Syntax
(if <predicate> (success <then>) (failure <else>))
; If the cust-seg event record value is "gold" then send message-one to echo otherwise send
; message-two

(if (= *cust-seg* "gold")
  (success (send echo message {:message message-one}))
  (failure (send echo message {:message message-two})))
```


Predicates can be composed of the following commands:
```clojure
; AND
(and <predicate> <predicate>)

; OR
(or <predicate> <predicate>)

; NOT
(not <predicate>)

; =
(= <predicate> <predicate>)

; >
(> <predicate> <predicate>)

; <
(< <predicate> <predicate>)

; or a value of any type
*id*
42
"Hello World"
```

The *then* and *else* elements can be either an execution block or a single command.

### Options

The following options can be applied to **seq**, **par**, **send**, and **await** commands

#### retries

If a command has a defined retry option, when the command fails, it will be retried. A command will
only be treated as *failed* when all of its retries have been exhausted.

```clojure
; Syntax
(retries <count>)

; Send a message to echo with 2 retries
(send echo message {:message "Hello World"}
  (retries 2))
```

#### within

When a command is evaluated which has a defined timeout, if it does not complete within the defined duration,
the event will be failed. Retries will be respected by timeouts, if the timeout occurs, and retries remain,
the command will be retried.

```clojure
; Syntax
(within <duration> <unit>)

; Send three messages to echo in parallel with a five second timeout
(par
  (send echo message {:message "Hello World"})
  (send echo message {:message "Hello World"})
  (send echo message {:message "Hello World"})
  (within 5 seconds))
```



#### template

Parameters defined using the **template** option will be sent to the endpoint as the rendered
version of the provided template. The template will have access to the canonical event record when
being rendered. The value for the template can be any of the value types defined in the Notification DSL.
Parameters which are to be templated, do not need to be defined in the main send parameter definition.


```clojure
; Syntax
(template {<params>})

;Use a template when sending an sms to twilio
(send twilio sms {:to-phone-number *phone-number*
                  :from-phone-number "1-506-555-1234"}
  (template {:message "Hello {{first-name}}"}))

(send twilio sms {:to-phone-number *phone-number*
                  :from-phone-number "1-506-555-1234"}
  (template {:message +TM1}))
```

#### success and failure

The **success** and **failure** options can be used to provide additional actions to be
taken based upon the success and/or failure of a command. One does not need to define both success
and failure option if not required. In the presence of a **failure** option the failure of the command is swallowed and the reaction
will continue processing after evaluating the *then* commands. The **failure** option will only be
evaluated once a command has exhausted all of its defined retries. The **success** block is the only scope in which commands have access to the endpoint response
values (those prefaced with a **$**). If the persistence of a response value is required for the
duration of the reaction, a **set** command can be used to persist the value in a reaction variable.



```clojure
; Syntax
(success <then>)
(failure <then>)

; Send an sms, and send a different message to echo depending on the success of the sms
(send twilio sms {:to-phone-number *phone-number*
                  :from-phone-number "1-506-555-1234"}
  (success (send echo message {:message "It sent"}))
  (failure (send echo message {:message "No it didn't"})))
```




# Using the API

CxEngage can be integrated with by using CxEngage API.

## Authentication 

url - https://auth.cxengage.net/1.0

To connect with the CxEngage API, you need to authenticate with a token.

In order to authenticate a user, a user must provide its `client_id` and `client_secret`. You can access your client id and secret by logging into [cxengage.net](https://www.cxengage.net) and clicking on your avatar.

Once a user has its access token, he or she will be able to make authenticated requests to any resources it has permission to access.


grant_type - client_credentials
client_id - Provided client id
client_secret - Provided client secret


**Example Request:**

```http
POST /1.0/token HTTP/1.1
Host: auth.cxengage.net/1.0
Authorization: Basic {{client-id:client-secret base64 encoded}} 
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
```

```bash
curl -X POST https://auth.cxengage.net -u {{client-id}}:{{client-secret}} \
-d "grant_type=client_credentials"
```

**Example Response:**

```json
{"access_token":"token",
 "token_type":"bearer",
 "expires_in":3600}
```

Now that you have your access token, you can make authenticated requests.

```bash
curl -X GET https://api.cxengage.net/1.0/tenants/{{tenant-name}}
     -H 'Authorization: Bearer {{Token}}'
```


## Sending in Events 

url - https://events.cxengage.net
  
Events are sent into CxEngage via a REST based API as in the example below. Note that the url is different from the CxEngage API

```http
POST /1.0/tenants/userevents/event HTTP/1.1
Host: events.cxengage.net
Content-Type: application/json
Authorization: Bearer {{token}}
```

```json
{
"username" : "1"
}
```

**Example Response:**

```json
{"event":{"username":"1"},
"key-attr":"1",
"id":"EV11-1",
"contributing?":false}
```

## CxEngage API 
url - https://api.cxengage.net/1.0

## Tenants

Retrieve Tenant info

__Request__

```http
GET /1.0/tenants/{{tenant-name}} HTTP/1.1
Host: api.cxengage.net
Authorization: Bearer {{token}}
```
**Return**

```json

{
    "id": "tenant1",
    "name": "Tenant Number One"
}

```

**curl Example**

```bash
curl -X GET https://api.cxengage.net/1.0/tenants/{{tenant-name}} \
     -H 'Authorization: Bearer {{token}}'
```


## Key Attribute

**Parameters**

key

```
key attribute
```

**Retrieve key attribute for the given tenant**

Request
```http
GET /1.0/tenants/{{tenant-name}}/key-attribute HTTP/1.1
Host: api.cxengage.net
Content-Type: application/json
Authorization: Bearer {{token}}
Cache-Control: no-cache
```

Response

```json
{
    "key": "id"
}
```

curl Example

```bash
curl -X GET https://api.cxengage.net/tenants/{{tenant-name}}/key-attribute \
     -H 'Authorization: Bearer {{token}}'

```

**Update key attribute for the given tenant**

Request

```http
POST /tenants/{{tenant-name}}/key-attribute HTTP/1.1
Host: {{cxenage-api}}
Content-Type: application/json; charset=utf-8
Authorization: Bearer {{token}}
```

Response

```json
{
    "key": "username"
}
```
 
curl Example

```bash
curl -X POST https://api.cxengage.net/tenants/{{tenant-name}}/key-attribute \
 -H 'Authorization: Bearer {{token}}' \
 -H 'Content-Type: application/json; charset=utf-8' \
 -d '{"key" : "username"}' 
```

   
## Patterns

**Mandatory Parameters**

name 

```
Name of Pattern
```

**Optional Parameters**

description 
```
Description of pattern
```
status 
```
Boolean value. Set to true/false to enable/disable the pattern
```
when 
```
When portion of pattern
```
then 
```
Then portion of pattern
```

**Retrieves all patterns from chosen tenant**

Request

```http
GET /1.0/tenants/{{tenant-name}}/patterns HTTP/1.1
Host: api.cxengage.net
Content-Type: application/json
Authorization: Bearer {{token}}
Cache-Control: no-cache
```

Response

```json

  {
   "id": "PT1",
   "then": "(par (send echo message {:message \"We should probably call the customer\"}) (send echo message {:message \"We should probably call the customer now now now now\"}))",
   "when": "(within 1 minutes (allOf (count 4 (event (and (= customerSegment \"Gold\") (= eventType \"flcheck\")))) (event (and (= customerSegment \"Gold\") (= eventType \"cnclTicket\"))))))",
   "description": "Loyalty Pattern for Gold customers",
   "name": "Loyalty Pattern"
  }

```

**Create a pattern on the given tenant**

Request

```http
POST /1.0/tenants/{{tenant-name}}/patterns HTTP/1.1
Host: api.cxengage.net
Content-Type: application/json; charset=utf-8
Authorization: Bearer {{token}}
```

```json
{ 
  "name": "Sample Pattern",
  "description": "Sample",
  "status":true,
  "when": "(when (event (= \"id\" \"1234\")))",
  "then": "(seq (send echo message {:message \"Hello world\"}))"
}
```

Response

```json
{ 
  "id": "PT2",
  "name": "Sample Pattern",
  "description": "Sample",
  "status":true,
  "when": "(when (event (= \"id\" \"1234\")))",
  "then": "(seq (send echo message {:message \"Hello world\"}))"
}
```

curl Example

```bash
curl -X POST https://api.cxengage.net/1.0/tenants/{{tenant-name}}/patterns \
 -H 'Authorization: Bearer {{token}}' \
 -H 'Content-Type: application/json; charset=utf-8' \
 -d '{"then":"(send echo message {:message \"Hello curl\"})","when":"(event (= type \"curl\"))","status":true,"name":"curl Pattern"}'
```


**Retrieves chosen pattern for the given tenant**

Request

```http
GET /1.0/tenants/{{tenant-name}}/patterns HTTP/1.1
Host: api.cxengage.net
Content-Type: application/json
Authorization: Bearer {{token}}
```

Response

```json
{
  "id":"PT2",
  "when":"(when (event (= \"id\" \"1234\")))",
  "then":"(seq (send echo message {:message \"Hello world\"}))",
  "status":true,
  "description":"Sample Pattern",
  "name":"Sample Pattern"
}
```
curl Example

```bash
curl -XGET https://api.cxengage.net/tenants/{{tenant-name}}/patterns \
     -H 'Authorization: Bearer {{token}}'
```


**Update the pattern for the given tenant based on the pattern id given**

Request

```http
PUT /1.0/tenants/{{tenant-name}}/patterns/PT5 HTTP/1.1
Host: api.cxengage.net
Content-Type: application/json; charset=utf-8
Authorization: Bearer {{token}}
Cache-Control: no-cache

```

```json
{
"then" : "(send sendgrid email {:to *email*, :subject \"Welcome to a wonderful put experience\"})"
}
```

Response

```json

{"id":"PT5",
"then":"(send sendgrid email {:to *email*, :subject \"Welcome to a wonderful put experience\"})",
"when":"(event (or (= username \"cxengage\")))",
"status":true,
"name":"Updated Pattern"
}    
```

curl Example
```bash
curl -X PUT https://api.cxengage.net/1.0/tenants/userevents/patterns/PT5 \
 -H 'Authorization: Bearer {{token}}' \
 -H 'Content-Type: application/json; charset=utf-8' \
 -d '{"then" : "(send sendgrid email {:to *email*, :subject \"Welcome to a wonderful put experience\"})"}'

```
   
## Templates

**Retrieves templates for the given tenant**

Request

```http
GET /1.0/tenants/{{tenant-name}}/templates/TM1 HTTP/1.1
Host: api.cxengage.net
Content-Type: application/json; charset=utf-8
Authorization: Bearer {{token}}
Cache-Control: no-cache
```

Response

```json

    {
        "id": "TM1",
        "template": "Hi {{FirstName}}, sorry we missed your call. Call Joe at
+14153159430 re: 401K needs. Gelson Bank.",
        "description": "Default SMS message with click-to-call",
        "name": "SMS"
    }
```

curl Example

```bash
curl -XGET https://api.cxengage.net/tenants/{{tenant-name}}/templates/TM1 \
     -H 'Authorization: Bearer {{token}}'
```

**Update chosen template**

Request

```http
POST /1.0/tenants/{{tenant-name}}/templates HTTP/1.1
Host: api.cxengage.net
Content-Type: application/json; charset=utf-8
Authorization: Bearer {{token}}
Cache-Control: no-cache
```

```json
  {
    "template": "Hi {{first-name}}\r\n\r\nWe apologize for the inconvenience. Please contact us at {{email-address}} at your earliest convenience.\r\n\r\nThank You\r\n",
    "description": "SMS apologizing to customer",
    "name": "SMS"
  }
```

Response

```json
{
    "id": "TM1",
    "template": "Hi {{first-name}}\r\n\r\nWe apologize for the inconvenience. Please contact us at {{email-address}} at your earliest convenience.\r\n\r\nThank You\r\n",
    "description": "SMS apologizing to customer",
    "name": "SMS"
}
```

curl Example

```bash

curl -X POST https://api.cxengage.net/1.0/tenants/userevents/templates \
 -H 'Authorization: Bearer {{token}}' \
 -H 'Content-Type: application/json; charset=utf-8' \
 -d '{"template": "Welcome to the 4.0 version of CxEngage", "description" : "sample description", "name" : "Welcome"}'

```

**Retrieve chosen template**

Request

```http
GET /1.0/tenants/{{tenant-name}}/templates/TM1 HTTP/1.1
Host: api.cxengage.net
Content-Type: application/json; charset=utf-8
Authorization: Bearer {{token}}
Cache-Control: no-cache
```

Response

```json

    {
        "id": "TM1",
        "template": "Hi {{FirstName}}, sorry we missed your call. Call Joe at
+14153159430 re: 401K needs. Gelson Bank.",
        "description": "Default SMS message with click-to-call",
        "name": "SMS"
    }
```

curl Example

```bash
curl -XGET https://api.cxengage.net/tenants/{{tenant-name}}/templates/TM1 \
     -H 'Authorization: Bearer {{token}}'
```

**Update chosen template**

Request

```http
PUT /1.0/tenants/{{tenant-name}}/templates/TM1 HTTP/1.1
Host: api.cxengage.net
Content-Type: application/json; charset=utf-8
Authorization: Bearer VjVYw3yJaujNoW+k5CGfe23mnJVOdhJZZ/sdrwhpZ41z
Cache-Control: no-cache
```

```json
  {
   
    "name": "Test Template"
  }
```

Response

```json
{
    "id": "TM1",
    "template": "Hi {{first-name}}\r\n\r\nWe apologize for the inconvenience. Please contact us at {{email-address}} at your earliest convenience.\r\n\r\nThank You\r\n",
    "description": "SMS apologizing to customer",
    "name": "Test Template"
}
```

curl Example

```bash
curl -XPUT https://api.cxengage.net/1.0/tenants/{{tenant-name}}/templates/TM1 \
     -H 'Authorization: Bearer {{token}}' \
     -H 'Content-Type: application/json; charset=utf-8' \
     -d '{"name":"Test Template"}'
```

**Delete chosen template**

Request

```http
DELETE /1.0/tenants/{{tenant-name}}/templates/TM1 HTTP/1.1
Host: api.cxengage.net
Content-Type: application/json; charset=utf-8
Authorization: Bearer {{token}}
Cache-Control: no-cache
```

Response

```json
{

  "TM1"
}
```

curl Example

```bash
curl -IX DELETE https://api.cxengage.net/tenants/{{tenant-name}}/templates/TM1 \
 -H 'Authorization: Bearer {{token}}' \ 
```

 

## Listeners

**Retrieve listeners**

Request

```http
GET /1.0/tenants/{{tenant-name}}/listeners HTTP/1.1
Host: api.cxengage.net
Content-Type: application/json
Authorization: Bearer {{token}}
Cache-Control: no-cache
```

Response

```json

  [
    {
        "status": true,
        "topic": "CxDemo_0720v2",
        "name": "My Salesforce Listener",
        "version": "26.0",
        "type": "salesforce",
        "mapping": {
            "user": "user__c",
            "type": "Type",
            "stage": "StageName",
            "amount": "Amount",
            "daystoclose": "daysToClose__c",
            "product": "Product__c"
        },
        "id": "LI2"
     
    },
    {
        "id": "LI1",
        "name": "Demo Datasift",
        "type": "datasift",
        "mapping": {
            "username": "interaction.author.username",
            "sentiment": "salience.content.sentiment",
            "id": "twitter.user.screen_name"
        },
        "hash": "xxxxxxxxxxxxxx",
        "status": true
    }
]
```

curl Example
```bash
curl -XGET https://api.cxengage.net/tenants/{{tenant-name}}/listeners \ 
     -H 'Authorization: Bearer {{token}}'
```

**Create listener**

Request

```http
POST /1.0/tenants/{{tenant-name}}/listeners HTTP/1.1
Host: api.cxengage.net
Content-Type: application/json; charset=utf-8
Authorization: Bearer {{token}}
```

Salesforce Listener Example

Request

```
  POST tenants/tenant-name/listeners
```

```json

{
    "status": true,
    "topic": "CxDemo_0720v2",
    "name": "My Salesforce Listener",
    "version": "26.0",
    "type": "salesforce",
    "mapping": {
        "user": "user__c",
        "type": "Type",
        "stage": "StageName",
        "amount": "Amount",
        "daystoclose": "daysToClose__c",
        "product": "Product__c"
    } 
}
```
Response

```json
{
    "id": "LI3",
    "status": true,
    "topic": "CxDemo_0720v2",
    "name": "My Salesforce Listener",
    "version": "26.0",
    "type": "salesforce",
    "mapping": {
        "user": "user__c",
        "type": "Type",
        "stage": "StageName",
        "amount": "Amount",
        "daystoclose": "daysToClose__c",
        "product": "Product__c"
    }
}
```

curl Example

```bash
curl -XPOST https://api.cxengage.net/tenants/{{tenant-name}}/listeners \
     -H 'Authorization: Bearer {{token}}' \
-H 'Content-Type: application/json; charset=utf-8' \
-d '{"name":"My Salesforce Listener","type":"salesforce","mapping":{"user":"user__c","type":"Type","stage":"StageName","amount":"Amount","daystoclose":"daysToClose__c","product": "Product__c"},"status":true}'
```

Datasift Listener Example

**Request**

```http
POST /1.0/tenants/{{tenant-name}}/listeners HTTP/1.1
Host: api.cxengage.net
Content-Type: application/json; charset=utf-8
Authorization: Bearer {{token}}
```

```json

{
    "name": "sentiment listener",
    "type": "datasift",
    "mapping": {
        "username": "interaction.author.username",
        "retweet_count": "twitter.retweet.count",
        "network": "interaction.type",
        "sentiment": "salience.content.sentiment",
        "followers": "twitter.user.followers_count",
        "user_url": "interaction.author.link",
        "profile_image": "twitter.user.profile_image_url"
    },
    "hash": "datasifthash",
    "status": true
}
```
Response

```json
{
    "id": "LI1",
    "name": "sentiment listener",
    "type": "datasift",
    "mapping": {
        "username": "interaction.author.username",
        "retweet_count": "twitter.retweet.count",
        "network": "interaction.type",
        "sentiment": "salience.content.sentiment",
        "followers": "twitter.user.followers_count",
        "user_url": "interaction.author.link",
        "profile_image": "twitter.user.profile_image_url"
    },
    "hash": "datasifthash",
    "status": true
}
```

curl Example

```bash
curl -XPOST https://api.cxengage.net/tenants/{{tenant-name}}/listeners \
     -H 'Authorization: Bearer {{token}}' \
-H 'Content-Type: application/json; charset=utf-8' \
-d '{"name":"Demo Datasift","type":"datasift","mapping":{"username":"interaction.author.username","sentiment":"salience.content.sentiment","id":"twitter.user.screen_name"},"hash":{{{datasift-hash}},"status":true}'
```

**Retrieves chosen listener**

Request

```http
GET /1.0/tenants/{{tenant-name}}/listeners/LI4 HTTP/1.1
Host: api.cxengage.net
Content-Type: application/json
Authorization: Bearer VjVYw3yJaujNoW+k5CGfe23mnJVOdhJZZ/sdrwhpZ41z
Cache-Control: no-cache
```

Response

```json

    {
        "status": true,
        "topic": "CxDemo_0720v2",
        "name": "My Salesforce Listener",
        "version": "26.0",
        "type": "salesforce",
        "mapping": {
            "user": "user__c",
            "type": "Type",
            "stage": "StageName",
            "amount": "Amount",
            "daystoclose": "daysToClose__c",
            "product": "Product__c"
        },
        "id": "LI4"
     
    }
```

curl Example

```bash
curl -XGET https://api.cxengage.net/tenants/{{tenant-name}}/listeners/LI4 \
     -H 'Authorization: Bearer {{token}}'  
```

**Update chosen listener**

Request

```http
PUT /1.0/tenants/{{tenant-name}}/listeners/LI4 HTTP/1.1
Host: api.cxengage.net
Content-Type: application/json; charset=utf-8
Authorization: Bearer {{token}}
Cache-Control: no-cache
```

```json

  {"name" : "Updated Name for listener"}
    
```

Response

```json

{
    "id": "LI4",
    "status": false,
    "topic": "CxDemo_0720v2",
    "name": "Updated Name for listener",
    "version": "26.0",
    "type": "salesforce",
    "mapping": {
        "user": "user__c",
        "type": "Type",
        "stage": "StageName",
        "amount": "Amount",
        "daystoclose": "daysToClose__c",
        "product": "Product__c"
    }
}

```

### curl Example

```bash
curl -XPUT https://api.cxengage.net/tenants/{{tenant-name}}/listeners/LI4 \
     -H 'Authorization: Bearer {{token}}' \
-H 'Content-Type: application/json; charset=utf-8' \
-d '{"name":"Updated Name for listener"}'

```



   ```
   [Update Listener](https://github.com/cxengage/cxengage.github.io/blob/master/cxengage-api-doc/Update-Listener.md) 
   ```
   Update a specific listener for a given tenant
   ```
   [Delete Listener](https://github.com/cxengage/cxengage.github.io/blob/master/cxengage-api-doc/Delete-Listener.md) 
   ```
   Delete a specific listener for a given tenant
   ```

## Integrations

   [Get Integrations](https://github.com/cxengage/cxengage.github.io/blob/master/cxengage-api-doc/Get-Integrations.md) 
   ```
   Get all integrations for a given tenant
   ```
   [Create Integration](https://github.com/cxengage/cxengage.github.io/blob/master/cxengage-api-doc/Create-Integration.md) 
   ```
   Create a new integration for a given tenant, i.e for Salesforce, Twilio, Sendgrid and Datasift
   ```
   [Get Integration](https://github.com/cxengage/cxengage.github.io/blob/master/cxengage-api-doc/Get-Integration.md) 
   ```
   Get a specific integration for a given tenant
   ```
   
## Augments
   
   [Get Augments](https://github.com/cxengage/cxengage.github.io/blob/master/cxengage-api-doc/Get-Augments.md)
   ```
   Get all Augments for a given tenant
   ```
   [Create Augment](https://github.com/cxengage/cxengage.github.io/blob/master/cxengage-api-doc/Create-Augment.md)
   ```
   Create a new Augment for a given tenant
   ```
   [Get Augment](https://github.com/cxengage/cxengage.github.io/blob/master/cxengage-api-doc/Get-Augment.md)
   ```
   Get a specific Augment for a given tenant
   ```
   
