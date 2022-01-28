HTTP Server implementation in LabVIEW. This project is under active development and as such should not be considered a secure web server implementation. We'll do our best to document the kinds of attacks we do try to protect against and by using this project you promise to educate yourself about web application security, especially when plugging in custom functionality which we can't prevent from circumventing the protections we are able to implement.

- [Usage Agreement & Regulations Compliance Statement](#usage-agreement--regulations-compliance-statement)
- [Overview](#overview)
  - [Request Processing Overview](#request-processing-overview)
  - [Services](#services)
- [Security First Design](#security-first-design)
- [Sessions](#sessions)
  - [Session ID](#session-id)
  - [Session Expiry](#session-expiry)
  - [Session Variables](#session-variables)
- [Processing Flow](#processing-flow)
- [Handlers and Controllers](#handlers-and-controllers)
  - [Priority](#priority)
  - [Routing & Path Parameters](#routing--path-parameters)
  - [Steps Types](#steps-types)
    - [Connection Filtering](#connection-filtering)
    - [Request Initialization](#request-initialization)
    - [Request Routing](#request-routing)
    - [Controller Initialization](#controller-initialization)
    - [Controller Execution](#controller-execution)
    - [Response Handling](#response-handling)
    - [Cleanup](#cleanup)
    - [Exception Handling](#exception-handling)
- [Debugging](#debugging)

# Usage Agreement & Regulations Compliance Statement

 Usage of this code as provided does not constitute any guarantee of protection or compliance with public internet and private network regulations. Data protection consent regulations, such as GPDR, may only be important for public facing websites that may store user information, even anonymous, and even if just for details such as theming and locale preferences. For built-in capabilites, the mechanisms that can be enabled in this server will likely not trigger consent for regulations, such as GPDR, as the internal purposes are only for identifying preferences and a logged-in user which is usually defined as "strictly necessary". However, since this server allows plugging in additional functionality it is impossible to predict when consent and compliance requirements will be necessary for sites using this server implementation. Therefore it is up to developers using this library to familiarize themselves with their local and client facing regulations related to data protections that may result during client interactions that can identify an individual. These statements assume that non-public uses would already be covered under employer IT agreements and it is up to the user of this library to validate the correctness of these assumptions with their IT department. This document aims to clearly describe the security and privacy mechanisms available to developers, however it is up to those developers to follow the documentation available in-source and in accompanying documentation and if documentation is lacking, to do their own research on web security and to understand the code.

 If there are any questions regarding the previous paragraph, please contact the owner of this repository. Contacting the owner of this repository is not a guarantee of a response and it remains the responsibility of the user of this code for all known and unknown regulations that may relate to the usage of this code.

# Overview

This LV Http Server is intended to both be easy to use for common use-cases but is highly flexible in how it can process requests for those that need extensibility. For common uses such as serving files from a directory or simple URL matching to run LV code, the examples should be referenced. Examples will demonstrate the much smaller scope of details that are required for using those features than the rest of this readme covers.

The following is all that is needed to have a LabVIEW application serve files from a folder relative to the project/executable:
![Simple File Serving Example](https://raw.githubusercontent.com/illuminated-g/lv-http-server/main/Documentation/images/simplefileserve.png "Simple File Serving Example")

If there is functionality that should be counted among the "common" use-cases but isn't available, feel free to submit an issue describing the functionality.

## Request Processing Overview

The processing flow is modelled after Symfony PHP's event subscription system and will be more familiar to backend developers that work with modern web frameworks. This HTTP server provides interfaces to implement custom Handlers and Controllers. Handlers represent small processing steps to select and initialize how a Request is processed and a Response is generated. Controllers are the main mechanism for generating the primary Response to a request. As examples, there might be a Handler whose sole purpose is to identify a user account attached to the request session and then a Controller could be provided that responds to a specific URL path and can use the request and user information initialized prior to the controller being run to generate the appropriate response.

## Services

In addition to Handlers and Controllers, Services are another pluggable mechanism provided by the server. There are two main differences between services and handlers:

1. Handlers utilize HTTP and HTTP aware abstractions to manage data relevant to being a web server whereas Services should be implemented to not know about HTTP mechanisms and only rely on application types and business logic.
2. Each handler applies to a specific step in the processing flow and services can be useful (and are available) throughout the entire processing flow.

This means that the relationship between handlers/controllers and services should be that handlers and controllers change the HTTP format of the requests into application data types and can then use those converted types with services to perform the business logic. Additionally, services may provide related groupings of functionality, such as user creation, retrieval, and management, which would be useful during request initialization, controller operation, and response handling.

# Security First Design

One of the design goals of this server is to be as secure as possible with default configurations by exposing the fewest number of attack vectors feasible (and known). This results in the following default configuration:

- All effort to properly validate and use encoding of HTTP request and response components is done.
- Sessions are disabled. (No client information is tracked)
- CORS is disabled.
- CSRF mitigations are enabled when using built-in form mechamisms.

***Proper security, even in the default configurations, is highly dependant on proper implementation. Please file issues on the repository as soon as an improper design/implementation is discovered or if there should be a different default configuration.***

If there is a potential attack vector that hasn't been addressed please create an issue with a link to documentation about the method (such as a link to an OWASP page) and if you're feeling particularly helpful you can create a Pull Request of an implementation for review.

Another aim of this project is to ensure where possible that the utilities necessary to facilitate safe implementations are available, as much as possible. Please submit issues for missing utilities, such as encoding formats or data validation, that will make implementing secure handlers and controllers easier for developers.

# Sessions

Sessions are implemented to be able to securely support user logins and client variables. Like most of the rest of the server, session storage is implemented abstractly to allow customizing how sessions are stored. There is a default implementation of persisting session values to individual files on disk and other implementations (Sqlite, Databases, Redis?) are possible if more performance or management capabilities are required.

Sessions do not require logins and can be used to store preferences even for anonymized/guest clients. Every session gets assigned a randomized value to identify the session that gets sent to the client as a cookie and then variables attached to the session will be saved by the server depending on the concrete implementation used. Then in successive requests when the client supplies the sessions ID by sending the session cookie with it, the session is revalidated by the server and the saved session values will be available during the request processing.

## Session ID

Session IDs typically are not used by any handler or controller code directly but are available to support additional resource validation and security features. Session IDs are Base64-Url encoded SHA-256 hashes of a random 128-bit string. Then the ID is hashed and encoded again for storage on the server. When requests come in with a session ID, it is re-hashed and encoded to be looked up and validated. Session IDs actually sent to the client, are only ever saved on the server in their hashed form, at least by the built-in mechanisms, to prevent leaking accessible sessions if the session storage is ever compromised.

## Session Expiry

All client sessions will automatically expire, with a default timespan of 3 days. This can be configured with the server or during session creation to change how long sessions remain valid. Upon receiving a request related to an expired session, saved session data will be removed and a new empty session will be created. When supporting logins, this typically represents automatic logout of a stale login session and the client would be directed to login again. To prevent losing session data in an application, sessions can be "refreshed" to generate a new ID and expiry for a session while maintaining the

## Session Variables

The main use for sessions is to save named values that correspond to a specific client. As long as sessions are enabled

# Processing Flow

This server provides flexibility and extensibility via a handful of specific request processing steps that can have additional functionality plugged in. The following flowchart maps out the processing flow.

![Request Processing Flowchart](https://raw.githubusercontent.com/illuminated-g/lv-http-server/main/Documentation/images/Request%20Flow.drawio.png "Request Processing Flowchart")

<br/>

# Handlers and Controllers

Aside from Controllers, which can certainly grow quite large, the expectation is that the other handlers should be small and run quickly. In the flowchart above, each of the blue shapes represents an interface point and except for the Controller step, is a loop to run through all registered handlers for that step.

<br/>

For instance, the following code of a Response Handler is small and serves a single purpose:

![FileExistChecker Code](https://raw.githubusercontent.com/illuminated-g/lv-http-server/main/Documentation/images/fileexistchecker.png "FileExistChecker Code")

<br/>

The only point of this handler is to verify that the specified file exists when the Response is a FileResponse. It doesn't generate a replacement Response itself, instead it returns an error 7 to allow the exception handling functionality to generate the 404 error. This lets the functionality for whether the full error string will be sent to the client or not based on whether debugging is enabled or not to live in one place and not have processing code strongly coupled to error handling and output code. Any functionality, especially error handling, that should be available to multiple handlers should become a separate Handler instead of relying on reusable subVIs that get used in multiple places.

<br />

## Priority

All controllers and handlers have a priority assigned to control what order they are run in. This is a U16 value and defaults to 10,000. This priority value can be changed by overriding the *Priority.vi* member and providing a different value. Higher priority Handlers have a lower numeric value and lower priority Handlers have a higher numeric value; they are sorted in ascending numeric value.

To provide a consistent behavior for handlers with the same priority, a second half of the priority is assigned based on the order handlers are registered. These two U16 values are combined into a single U32 priority value with the value from *Priority.vi* being the more significant half.

Built-in handlers and controllers will typically have higher numeric values for their priority to allow more room for organizing custom handlers into higher priorities (lower numeric values). Some of them will support customizing their priority when needed.

<br/>

## Routing & Path Parameters

Controllers are generally selected by a regular expression they specify with a dynamic dispatch VI:
![Controller Path Regex](https://raw.githubusercontent.com/illuminated-g/lv-http-server/main/Documentation/images/pathregex.png "Controller Path Regex")

The built-in regular expression handling breaks out submatches as specific arguments available to the controller:
![Controller Path Arguments](https://raw.githubusercontent.com/illuminated-g/lv-http-server/main/Documentation/images/pathargs.png "Controller Path Arguments")

<br/>

## Steps Types

<br/>

### Connection Filtering

Connection Handlers support behavior such as client rate limiting and client blacklisting in a way that is more performant than sending a full response. There is a specific HTTP response for rate limiting that should be used if possible so the client knows to slow down. This step should not be considered for use as a security feature since spoofing network identities is fairly easy. Any actions taken here should only be considered for performance considerations.

<br/>

### Request Initialization

Request Handlers serve two primary purposes:

1. Generating an Early Response to bypass Controller selection and processing.
2. Preparing data based on the request that will be relevant regardless of the resource being requested.

As an example, Point 1 is useful when most of the available resources or an entire path segment require authorization. A Response Handler can be created that uses the session, cookies, headers, or some other mechanism for identifying a client and allow the processing flow to proceed normally or if an identity can't be verified then it can create an unauthorized response or redirect the client to a login resource.

Point 2 allows setting up arguments for the rest of processing as necessary based on request headers, POST data, Form input, etc. Setting the identity up from Point 1 and setting it as an argument also makes that value easy to consume throughout the rest of the processing flow.

Triggering an Early Response by returning a non-empty Response instance shortcuts the processing flow and skips directly to the Response Handling step.

Any information that's relevant to a specific controller should be handled in Controller Initialization or the Controller itself and information related to a specific resource should be handled in a Controller.

<br/>

### Request Routing

Routing Handlers are called to determine which Controller should be used to respond to the resource request. The resource path will typically be used to determine Controller selection and as such all Controllers are expected to provide a Regular Expression identifying the path(s) they generate Responses for and there is a built-in handler that runs to check the path against the regular expressions. Custom Routing Handlers may select Controllers by other criteria such as a specific hard-coded path, query parameters, headers, or anything else that fits the needs of the application.

If a controller is not identified after running through all registered handlers then it will be treated as an error and the processing will perform exception handling since no Controller was specified to generate a Response.

<br/>

### Controller Initialization

This step runs registered Controller Handlers that can perform additional initialization and creation of Arguments to be used by the selected Controller. The idea is to have as much information available before the Controller is run to separate initialization from execution when feasible. Sharing initialization routines across several Controllers is also supported by this design.

<br/>

### Controller Execution

The selected Controller is called to generate a Response to the Request. Not generating a response is considered an error and will trigger exception handling in the processing since there is no Response to send to the client. At a minimum, successful Controller execution should provide a generic Response with the status code set; all other aspects of a response are optional.

Individual Controllers should avoid setting values in a Response that apply to multiple controllers or the entire server. For instance, if the entire server should set a CORS header, that should be handled in Response Handling instead of being set by Controllers.

<br/>

### Response Handling

Response Handlers can be written to apply to specific Response types by attempting to cast the Reponse instance to the specific type. They can also apply to all Responses indiscriminently which is useful for tasks like setting CORS headers. Lastly, a Response Handler has the option to completely replace the Response generated by a Controller or purposefully returning an error based on the Response. Since this wastes effort performed by a Controller this is generally only useful for sharing error handling across multiple Controllers, such as verifying that a file for a FileResponse exists.

This step is the last chance to modify the Response before it is sent to the client. As soon as all Response Handlers are run and an error isn't generated, the Response will be sent to the client.

<br/>

### Cleanup

Complete Handlers should be used to share end-of-processing operations across multiple Controller types or whenever an action should be performed after the Response has already been sent to the client. An example would be generating an email or report but allowing the response to be sent before the lengthy operation so the client isn't left waiting and the user/app experience is much snappier. Another use-case is performing additional detailed logging for the request.

After this step, processing will complete and errors returned will be logged to the Trace Log though they will not trigger exception handling since there is no longer a connected client to respond to. Unlike other steps, a handler in this step returning an error does not skip the remaining handlers.

<br/>

### Exception Handling ###

Except for Cleanup, any Handler or Controller that returns an error will trigger Exception Handling. The registered Exception Handlers will be run until the first Exception Handler that clears the error, signifying it has handled the error and no further handling is required. Exception handlers can also modify the error code that was initially provided for successive Handlers to leverage.

An example of how this supports extensibility is a higher priority Exception Handler (lower numeric value) that turns several different related error codes into a single error code so that a single lower priority (higher numeric value) Handler can perform the handling of related errors without needing to know all the errors it should perform its action for. This is especially useful for extending existing functionality with new processing and Controllers that may generate new error codes but existing Exception Handling should be utilized. The new Handlers and Controllers can be used with a new Exception Handler that translates the new expected codes into already handled codes.

After the Handlers are run there are several possible results. If a Handler cleared the error then that signifies that the Reponse it returned should be sent to the client. If the error was not cleared then a generic 500 error will be sent to the client and if the server was configured with debugging enabled then the full error message will be sent in the response.
>***Sending error messages has the potential to inadvertently reveal sensitive system information to the client. Debugging should never be enabled in a publicly accessible system!***

<br/>

# Debugging

The HTTP Server has a few debugging and logging features built-in though unlike most other web servers there isn't a default location to save logs to disk. Any built-in logging will be accessible during run-time and can be configured to use logging paths by the containing application.

One of the key features for debugging the request processing flow is the Trace Log:
![Request Trace Log](https://raw.githubusercontent.com/illuminated-g/lv-http-server/main/Documentation/images/tracelog.png "Trace Log Strict Type Def")

This shows information about the client making the request, each of the steps that occurred during processing, and the response details. It will highlight any steps that return errors that would cause exception flows to be triggered. For Handlers that return an item, such as request initialization handlers returning an Early Reponse or Controllers returning the main Response, the type of the returned item is saved. The trace log also stores the execution time of each step or call and the overall processing duration for determining processing performance and which steps may be causing bottle-necks.