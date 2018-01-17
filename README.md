# Configuring an IVR

## IVR Basics

### Structure

Each IVR uses a structure formally known as a directed graph, or more commonly, a flow chart.  It is a collection of nodes (the circles of a flowchart) and edges (the arrows of a flow chart). Each node represents some sort of event or action of the call, such as the robot voice reading a disclaimer, or asking for and receiving user input like an account number. Each of the edges simply represents a redirect from one node to another. So one node might gather a credit card number, and an edge might redirect to a second node that gathers the expiration date. For simplicity, throughout this document, edges will be refered to as redirects.

### Node Types

Each node has a set of properties that define its behavior, the most important of which is the node type. The node type defines the general class of functionality the node posesses. Each node type has a unique set of properties. The function of each node type is defined as follows:

#### Say

A say node simply reads a pre configured piece of text to the user. The text that is read can optionally utilize a dynamic templating system that can alter the text based on varying conditions or data associated with the user. This is covered more below.

#### Gather

A gather node is used to obtain input from the user, and then do something with that input. It will usually prompt the user for input by reading pre configured text, and then waits for the user to press buttons on his/her handset. The gather node will decide that the user is finished entering input by either waiting for a certain number of keys to be pressed, or waiting until a particular key is pressed (usually the # key). It then perform some action on this input. Usually it stores it for later use.

#### Split

A split node is used as a "fork in the road" so to speak. When we want the user to decide which direction to go, we use a split. An example use for a split would be asking the user if they want to press 1 to pay with a credit card, or press 2 to pay with an e check. The split node will read the options, and their associated keys, receive input from the user, and redirect to the appropriate node based on said input.

#### Split Condition

A split condition is similar to a split node in that it decides which node to redirect to. The split condition node doesn't rely on user input however. It chooses the redirect based on a condition defined in the action library (discussed below).

#### Action

An action node is a node that performs some pre defined action. There are several of these actions defined in something called the "action library" or simply "library". The action library is discussed in greater detail below.

#### Pause

A pause node just waits for a preconfigured amount of time before redirecting to another node. This is mostly utilized at the start of an IVR to add a ringing noise before the IVR connects. Many times merchants will redirect calls from their main line. Without a pause, the IVR will connect immediately without ringing, which some merchants feel is confusing to users.

#### SMS

An SMS node will send a text message to the caller. It is usually utilized for sending receipts. The content of the message utilizes dynamic templates, covered later.

#### Dial

A dial node is a way to redirect the call to another phone number. This is almost exclusively used as an operator function, where a use can choose to "speak to a real person". The call will then redirect to a merchants main line.

#### Hangup

A hangup node will end the call.

### Action Library

The action library is a collection of actions that are made available to several different node types (Gather, Split Condition, and Action). Although every action in the library is available to every node type that utilizes an action, typically actions that are used by one node type won't be used by the others. For example, actions used by Gather nodes typically store the input they receive in a global data store called the "context" or "model". Some examples of these types of actions are `setAccountID` or `setCreditCardNumber`.

Furthermore, actions utilized by split condition nodes are unique because they need to define a condition. They always decide either yes or no based on information available in the context. Take the action `rvoAllowedACHandCC` for example. If this action is used in a split node, it will look at the creditor id of the user and decide if that user is allowed to make both ACH and credit card payments and inform the split condition node by responding either yes or no. If the answer is yes, then that particular condition will be met, and the split condition node will redirect to the corresponding node.

Any actions that go beyond storing input for later or handling condition based splits are generally utilized by action nodes. These actions are where the real work happens like payment processing or authenticating users.

A common work flow utilizes gather functions to gather and store a series of inputs. When all the required input is gathered, an action node performs a major action on all of the previously gathered data. This can be seen in the log in workflow:

1. Gather node receives account number and stores it with `setAccountID` action.
2. Gather node receives last four ssn and then calls `authenticateAccount`. Authenticate account stores the ssn, looks for the previously stored account ID, and then sends both of these pieces of data to mammoth, which responds with data about the authenticated user, or a message saying the login failed. `authenticateAccount` then either stores the user data in the context for later use, or reports an error to the system (see error handling section)

Note that in this case, a gather node is calling the `authenticateAccount` action, rather than calling some type of `setSSN`, then redirecting to an action node that calls `authenticateAccount`. This is because in this case, we have all the required data to authenticate an account as soon as we receive the ssn input. We could simply store it and redirect to an action node, but here we have chosen to cut out a step and just perform the action inside the final gather node. Either way will work the same way.

### Error Handling

Errors can occur at essentially any point during an IVR call, so we need to handle these as elegantly as possible. Most nodes will define an `Error Redirect` property. If a particular node encounters an error, it will look at this property and redirect to the node specified. Usually, say nodes are created to read error messages to inform the user that something went wrong. An example of this would be on user authentication. If the login fails, that is an error. The log in node will look to the `Error Redirect` property which should tell it to redirect to a node that says something along the lines of "Your account could not be authenticated. Please try again."

If, for whatever reason, an `Error Redirect` property is not defined for a particular node, and that node does encounter an error, it will look for the `Default Error Redirect` defined on the IVR itself (not the node), and redirect the call there.

### Dynamic Templates

Many times when we want to read text to the user, we don't know exactly what that text should be when we configure it. Many times we'll want it to be different depending on what user is calling. For example, upon successful authentication, we often want to greet the user by name. But we won't know the user's name when we're configuring the prompt, so how can we solve this problem? Enter dynamic templates. Dynamic templates allow you to use placeholders to represent pieces of data that will change based on the situation. This can be accomplished with the follwing syntax: 

```
Hello, {{auth.account.first_name}}. You are successfully signed in.
```
We define a dynamic piece of data by enclosing it in double curly brackets, and then refer to it by the name of the variable. Here we can refer to treatments, account details, and any input that we've saved in previously run gather nodes. When this text is run, it will take the value stored in `auth.account.first_name` and substitute it into the text where the curly brackets are. So if the first name of the user is Joe, the text that gets read would be "Hello, Joe. You are successfully signed in."


Text that reads off a balance could be written as the follwing:

```
Your current balance is ${{auth.account.balance}}
```


We also can run conditional templates, that look at the true/false value of a variable, and say one thing if the variable is true, and another if it is false. An example of this would be credit/debit cards on debit only accounts.

```
to pay with a {{condition debit_only 'debit' 'credit'}} card, press 1
```
This form is a bit different. We indicate a condition with the word `condition` in front, followed by the name of the variable we want to test. In this case `debit_only` is a variable that is true if the account is a debit only account and false otherwise. We then put the text we want to be read if the value is true, and finally the text we want to be read if the value is false. So if it is a debit only account, the text would be read as "To pay with a debit card, press 1", and if it is not a debit only account, it would be "To pay with a credit card, press 1."

Dynamic templates are enabled in basically every place that reads text to the user, in any node type. It is also available for text message bodies used in SMS nodes.

## Property Reference

This section gives detailed explainations of all IVR level and Node level properties that can be configured through zeus.

### IVR Properties

System Settings are settings that pertain to the IVR system as a whole. Things like the default language, and phone number for accessing the IVR are examples of system settings. These settings are defined at the root level of the document.

| Name | Required/Optional | Description | Possible Values |
|------|-------------------|-------------|-----------------|
| merchant | required | The merchant using the IVR. Data accessed by users from this IVR will be restricted to this merchant | any possible merchant |
| access number | required | The phone number users dial to access the IVR | {+xxxxxxxxxxx} where x is any integer |
| default voice | optional | The voice that will be used if no voice is specified in a node | see [https://www.twilio.com/docs/api/twiml/say](https://www.twilio.com/docs/api/twiml/say) |
| default language | optional | The language that will be used if no language is specified in a node | see [https://www.twilio.com/docs/api/twiml/say](https://www.twilio.com/docs/api/twiml/say) |
| default timeout | optional | The timout in seconds that will be used if no timeout is specified in a node | any integer |
| default error redirect | required | The node to redirect to if no error redirect is defined for the node | any node in the ivr |

### Node Properties

#### General Node Properties

These settings are applicable to nodes of most if not all methods

| Name | Required/Optional | Description | Possible Values |
|----------|-----------------------|-----------------|---------------------|
| name | required | unique identifier for the node | any string |
| method | required | the node type | Say, Gather, Split, Split Condition, Sms, Hangup, Action, Pause, Dial |
| voice | optional | the twilio voice that will be used for this node | see [https://www.twilio.com/docs/api/twiml/say](https://www.twilio.com/docs/api/twiml/say) |
| language | optional | the twilio language that will be used for this node | see [https://www.twilio.com/docs/api/twiml/say](https://www.twilio.com/docs/api/twiml/say) |
| error redirect | optional | the node to redirect to if an error is thrown | any node in the ivr |

#### Say Properties

These settings are applicable only to say nodes

| Name | Required/Optional | Description | Possible Values |
|----------|-----------------------|-----------------|---------------------|
| template | required | handlebars template describing what will be said | any valid handlebars template (see [https://github.com/wycats/handlebars.js](https://github.com/wycats/handlebars.js)) |
| redirect | required |  the node to redirect to upon completion |  any other node in the ivr |

#### Gather Properties

These settings are applicable only to gather nodes

| Name | Required/Optional | Description | Possible Values |
|----------|-----------------------|-----------------|---------------------|
| prompt | required | the string that will be read before accepting user input | any string including dynamic templates |
| timeout | optional | the number of seconds the system will wait for input before hanging up | any positive integer (values over 20 are discouraged) |
| num digits | optional if finish on key defined | the number of digits of user input to accept | any positive integer (values over 20 are discouraged) |
| finish on key | optional if num digits is defined | the system will stop collecting user input when the user presses this key | any of the following: [0, 1, 2, 4, 5, 6, 7, 8, 9, *, #] |
| redirect | required | the node to redirect to upon completion | any other node in the ivr |
| action | required | the name of an action defined in the IVR library | the name of any action in the action library |

#### Split Properties

These settings are applicable only to split nodes

| Name | Required/Optional | Description | Possible Values |
|----------|-----------------------|-----------------|---------------------|
| timeout | optional | the number of seconds the system will wait for user input before hanging up | any positive integer (values over 20 are discouraged) |
| paths | required | a JSON array of objects describing the possible options (see  _Paths_) | see _Paths_ |
| invalid input redirect | required | id of the node to redirect to if the user enters invalid input |  any other node in the ivr |

*  Paths

    The paths setting of a split node is defined as an array of JSON objects. The array must contain at least 1 object and at most 10 objects. Each path object is defined as such:

    | Name | Required/Optional | Description | Possible Values |
    |----------|-----------------------|-----------------|---------------------|
    | key | required | the key pressed by the user to choose this path | any single digit integer |
    | prompt | required | what will be read to the user as a description of this path | any string (supports dynamic templates)|
    | redirect | required |  the node to redirect to if this path is chosen |  any node in the ivr |
    | Order | optional | The order the prompts should be read in, lower values will be read first | any integer |

    These paths will be read to the user in the form of: "Press [key] to [prompt]"


#### Split Condition Properties

| Name | Required/Optional | Description | Possible Values |
|----------|-----------------------|-----------------|---------------------|
| timeout | optional | the number of seconds the system will wait for user input before hanging up | any positive integer (values over 20 are discouraged) |
| paths | required | a JSON array of objects describing the possible options (see _Paths_) | see _Paths_ |
| default redirect | required | id of the node to redirect to if none of the conditions are met |  any other node in the ivr |



*  Condition Paths

    The paths setting of a condition split node is defined as an array of JSON objects. The array must contain at least 1 object. Each path object is defined as such:

    | Name | Required/Optional | Description | Possible Values |
    |------|-------------------|-------------|-----------------|
    | condition | required | The name of the boolean returning function defined in the library | the string representation of any function in the library returning a boolean value |
    | redirect | required |  the node to redirect to if this condition is met |  any node in the ivr |
    | Order | optional | The order the prompts should be considered in, lower values will be considered first, the first condition to be met will receive the redirect | any integer |


#### Action Properties

These settings are applicable only to action nodes

| Name | Required/Optional | Description | Possible Values |
|----------|-----------------------|-----------------|---------------------|
| action | required | the name of the library function to call | any valid library function that doesn't take input |
| redirect | required |  the node to redirect to after completing the action |  any other node in the ivr |
