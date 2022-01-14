HTTP Server implementation in LabVIEW. Utilizes IG TCP Streams and provides mechanisms akin to Web Services.

- [Processing Flow](#processing-flow)
- [Handlers and Controllers](#handlers-and-controllers)
  - [Priority](#priority)
  - [Steps Types](#steps-types)
    - [Connection Filtering](#connection-filtering)
    - [Request Initialization](#request-initialization)
    - [Request Routing](#request-routing)
    - [Controller Initialization](#controller-initialization)
    - [Controller Execution](#controller-execution)
    - [Response Handling](#response-handling)
    - [Cleanup](#cleanup)
    - [Exception Handling](#exception-handling)

# Processing Flow

This server provides flexibility and extensibility via a handful of specific request processing steps that can have additional functionality plugged in. The following flowchart maps out the processing flow.

<details open>
    <summary>Expand for Processing Flowchart</summary>

![Request Processing Flowchart](https://raw.githubusercontent.com/illuminated-g/lv-http-server/main/Documentation/images/Request%20Flow.drawio.png "Request Processing Flowchart")
</details>

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

All controllers and handlers have a priority assigned to control what order they are run in. This is a U16 value and defaults to 10,000. This priority value can be changed by overriding the *Priority.vi* member and providing a different value. Handlers will be run starting from lower priority numeric values to higher values. (Lower number = higher priority).

To provide a consistent behavior for handlers with the same priority, a second half of the priority is assigned based on the order handlers are registered. These two U16 values are combined into a single U32 priority value with the value from *Priority.vi* being the more significant half.

Built-in handlers and controllers will typically have higher numeric values for their priority to allow more room for organizing custom handlers into higher priorities (lower numeric values). Some of them will support customizing their priority when needed.

## Steps Types

### Connection Filtering

Connection Handlers support behavior such as client rate limiting and client blacklisting in a way that is more performant than sending a full response. There is a specific HTTP response for rate limiting that should be used if possible so the client knows to slow down. This step should not be considered for use as a security feature since spoofing network identities is fairly easy. Any actions taken here should only be considered for performance considerations.

### Request Initialization

Request Handlers serve two primary purposes:

1. Generating an Early Response to bypass Controller selection and processing.
2. Preparing data based on the request that will be relevant regardless of the resource being requested.

As an example, Point 1 is useful when most of the available resources or an entire path segment require authorization. A Response Handler can be created that uses the session, cookies, headers, or some other mechanism for identifying a client and allow the processing flow to proceed normally or if an identity can't be verified then it can create an unauthorized response or redirect the client to a login resource.

Point 2 allows setting up arguments for the rest of processing as necessary based on request headers, POST data, Form input, etc. Setting the identity up from Point 1 and setting it as an argument also makes that value easy to consume throughout the rest of the processing flow.

Triggering an Early Response by returning a non-empty Response instance shortcuts the processing flow and skips directly to the Response Handling step.

Any information that's relevant to a specific controller should be handled in Controller Initialization or the Controller itself and information related to a specific resource should be handled in a Controller.

### Request Routing

Routing Handlers are called to determine which Controller should be used to respond to the resource request. The resource path will typically be used to determine Controller selection and as such all Controllers are expected to provide a Regular Expression identifying the path(s) they generate Responses for and there is a built-in handler that runs to check the path against the regular expressions. Custom Routing Handlers may select Controllers by other criteria such as a specific hard-coded path, query parameters, headers, or anything else that fits the needs of the application.

If a controller is not identified after running through all registered handlers then it will be treated as an error and the processing will perform exception handling since no Controller was specified to generate a Response.

### Controller Initialization

This step runs registered Controller Handlers that can perform additional initialization and creation of Arguments to be used by the selected Controller. The idea is to have as much information available before the Controller is run to separate initialization from execution when feasible. Sharing initialization routines across several Controllers is also supported by this design.

### Controller Execution

The selected Controller is called to generate a Response to the Request. Not generating a response is considered an error and will trigger exception handling in the processing since there is no Response to send to the client. At a minimum, successful Controller execution should provide a generic Response with the status code set; all other aspects of a response are optional.

Individual Controllers should avoid setting values in a Response that apply to multiple controllers or the entire server. For instance, if the entire server should set a CORS header, that should be handled in Response Handling instead of being set by Controllers.

### Response Handling

Response Handlers can be written to apply to specific Response types by attempting to cast the Reponse instance to the specific type. They can also apply to all Responses indiscriminently which is useful for tasks like setting CORS headers. Lastly, a Response Handler has the option to completely replace the Response generated by a Controller or purposefully returning an error based on the Response. Since this wastes effort performed by a Controller this is generally only useful for sharing error handling across multiple Controllers, such as verifying that a file for a FileResponse exists.

This step is the last chance to modify the Response before it is sent to the client. As soon as all Response Handlers are run and an error isn't generated, the Response will be sent to the client.

### Cleanup

Complete Handlers should be used to share end-of-processing operations across multiple Controller types or whenever an action should be performed after the Response has already been sent to the client. An example would be generating an email or report but allowing the response to be sent before the lengthy operation so the client isn't left waiting and the user/app experience is much snappier. Another use-case is performing additional detailed logging for the request.

After this step, processing will complete and errors returned will be logged to the Trace Log though they will not trigger exception handling since there is no longer a connected client to respond to. Unlike other steps, a handler in this step returning an error does not skip the remaining handlers.

### Exception Handling ###

Except for Cleanup, any Handler or Controller that returns an error will trigger Exception Handling. The registered Exception Handlers will be run until the first Exception Handler that clears the error, signifying it has handled the error and no further handling is required. Exception handlers can also modify the error code that was initially provided for successive Handlers to leverage.

An example of how this supports extensibility is a higher priority Exception Handler (lower numeric value) that turns several different related error codes into a single error code so that a single lower priority (higher numeric value) Handler can perform the handling of related errors without needing to know all the errors it should perform its action for. This is especially useful for extending existing functionality with new processing and Controllers that may generate new error codes but existing Exception Handling should be utilized. The new Handlers and Controllers can be used with a new Exception Handler that translates the new expected codes into already handled codes.

After the Handlers are run there are several possible results. If a Handler cleared the error then that signifies that the Reponse it returned should be sent to the client. If the error was not cleared then a generic 500 error will be sent to the client and if the server was configured with debugging enabled then the full error message will be sent in the response. ***Sending error messages has the potential to reveal sensitive information to the client and debugging should never be enabled in a publicly accessible system!*** 