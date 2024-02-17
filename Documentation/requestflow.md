# IG HTTP Server Request Processing

Request processing in the IG HTTP Server is broken up into many extensible steps that make it easy to quickly provide simple responses to web requests all the way up to building a full application backend service complete with user authentication, access control, debug tracing, and error handling.

The rest of this document will follow  the flow chart shown below, describing each step in sequence.

Error checking is done after each step and diverts execution flow to the error handling logic shown at the bottom of the flowchart. In general, error handling bypasses any remaining processing steps and runs exception handling. This is the intended flow for triggering actions such as automatically redirecting to a login page when an authorization check fails, invalid requests are made, a specified request path is not available, etc. This is an opportunity to create a more specific response to the client and have that response sent to the client. For more details see the Error Handling section further in the document.

Each blue colored operation is a step of the process that can be extended with additional handling logic that is distributed with IG HTTP Server or with custom application logic that is registered with the server. With the exception of the `Controller` step, which is responsible for generating the main response, all pluggable steps can run through several registered handlers. For more information about registering handlers and implementing custom request processing logic head back to the top-level [readme](../README.md). All of the optional behavior that can be enabled in the server such as user authentication, sessions, file serving, websocket upgrades, and more all utilize the same exact handler extensibility.

![Request Flow Chart](images/Request%20Flow.drawio.png)

### Start

A very good place to begin, no? IG HTTP Server utilizes IG TCP Stream to manage the underlying TCP connection and stream read/writes. Actually this server provides an extremely light layer over top of IG TCP Stream to gain access to the TCP connection reference but that's a topic for another document. When the HTTP server creates the underlying TCP server, a VI reference callback is registered with the TCP server that is called anytime a new TCP connection is established with a client. Actually, there are two, each corresponding to a secure HTTPS TCP server and an insecure HTTP TCP server. (Since HTTP and HTTPS typically listen on two different ports, two different TCP server instances are used.) The only difference between the two callbacks is to correctly tell the processing code whether or not to upgrade the connection to a TLS secured HTTPS connection. The only task a connection callback performs when executed is to launch a processing instance as an asynchronous clone.

From this point on, all code is run underneath `Process Async.vi`, the reentrant VI that organizes the request processing logic.

### Connection Filtering (Connection Handlers)

Unlikely to be used for most applications using IG HTTP Server, this step allows deciding whether to close the connection before reading and parsing the incoming request. Since it's not terribly difficult to spoof IP and MAC addresses, this step provides little to no benefit with regards to security. It can be helpful in slowing down or throttling requests from hyperactive clients but most meaningful request filtering will happen after the request has been parsed. This means request headers, the resource being requested, and additional information will be available and provide much more context about whether to continue servicing the request or not.

As implied, if a Connection Filtering handler returns True for close, the connection is immediately closed without reading the request or sending any response to the client. It's generally helpful to the client to provide an error response if the request won't be serviced, so again, use this step sparingly if at all.

### Read Request

This is a built-in processing step that reads the incoming request from the client and parses the HTTP message representing the request. This includes information such as the HTTP version, the path of the resource being requested, the HTTP method being used (GET, POST, PUT, etc), all headers, posted form data, file uploads, and request body content. The total request size is limited to 50MB (I haven't added configuration for this limit yet, hopefully I remember to update this once I do!) so that clients can't try uploading overly large files or content bodies and overloading the server. This implementation supports chunked transfer encoding as well as URL encoded and multipart forms. With multipart forms, a temporary upload directory can be configured with the server and files will be saved directly to disk and available to be read/renamed/moved throughout the rest of the processing flow. Any files not moved out of the temporary folder are deleted at the end of the request handling. If an upload folder is not configured then file content is kept as a string in memory to be used during the request.

### Request Init (Request Handlers)

Once the request has successfully been parsed, the first set of handler(s) are run on the request. The only exception is if the connection came in on HTTP and the server is configured to only support HTTPS. In this case a hard-coded HTTPS redirect handler is invoked to generate a redirect response and the rest of the request processing is skipped up to the Response Handling and sending the redirect response to the client. This tells the client to make the request again but with HTTPS (and with the configured port if the standard port isn't used) so that the request and any response data is transferred with TLS encryption. This shortcutting of the request processing is accomplished by returning *True* for the *Early Response* value, which will be discussed further momentarily.

When the hard-coded HTTPS redirection isn't triggered, the processing logic iterates over all registered Request Handlers until a handler returns an error value or returns a Response instance to trigger the Early Response shortcutting. Request handlers can be used for everything from user authentication, session initialization, request validity checking, and so on. For instance, when the built-in file serving functionality is enabled in the server and pointed at a root folder to serve files from, additional Request Handler logic is run to ensure that the request isn't trying to be sneaky by specifying path ascension segments ".." in the URL and automatically generates an error if that's the case to prevent further request processing and access to files outside of the specified root folder.

The "normal" processing flow typically involves selecting a Controller to service the request and generate the primary response (usually selected based on the requested URL path) however sometimes none of that logic needs to be run. If a user succeeds a login attempt that's handled by the built-in authentication capabilities, then the user should be redirected to the page they were initially trying to access. Anytime a Request Handler sets a Response value the Early Response behavior will be triggered.

### Routing (Route Handler)

Once processing has made it past the Request Handlers without an Early Response, Route Handlers are called to determine which Controller should be run to generate a response to the client. The idea is to "route" a request to a Controller (think an MVC controller) that services the request. Most of the time this will happen based on the resource path sent in the request which is discussed in more detail below. It is also possible to decide which Controller should be invoked by looking at request headers or by using other details available in the request. This means it is possible to have "standard" controllers that use the request path (if important) as an input parameter instead of being written for specific request paths.

If a Controller isn't selected to service the request then an error is generated, typically resulting in a 404 error response being sent to the client by default. See the error handling section for more details.

#### Request Path Matching (Path Regex)
The Route Handler that is always available is a regular expression matching router that uses the pattern specified by each controller to see if it matches the request path. Each Controller implementation is required to supply a regex pattern. Even if the path matching isn't being utilized, the VI must be overridden but the pattern can be left unspecified. There are a couple of common regex components that will come in handy but a website such as https://regexr.com can be used to help create and test regular expressions.

- An empty pattern will match any path.
- All paths being matched will always start with a forward slash '/'.
- Placing a '^' at the front of a pattern anchors the pattern to the beginning of the path.
- Placing a '\$' at the end anchors the pattern to the end of the path. However with the request regex matching, the URL fragment and query are included in the the matching check and so a $ should generally not be used if either fragments (# links to anchor tags) or query parameters are supposed to be used for that path.
- Forward-slashes '/' used when separating path segments need to be escaped with a backslash '\\', e.g. '^/article\\/'
- Submatches, portions of a pattern within parenthesis, can be used to pull portions of the path out as parameters that can be used in the Controller logic. More on this in the Submatch Parameters section below.

Eventually the plan is to provide an API that can be used to build up the regular expression pattern with standard operations such as path segments, parameter sections, and anchoring logic.

#### Submatch Parameters

Submatch parameters enable using portions of a request path to be used as parameters for logic that generates the response. For a first example, let's look at a potential way to pull a user's profile that is handled by the application:

Pattern: "^\/profile\/(\d+)$" <br/>
Regexr: https://regexr.com/7rhte

The regexr link provides a detailed explanation of the various pattern elements but the gist is that this pattern would match a path such as "/profile/5555" with 5555 presumably being the user ID of the user to view. The string value "5555" would be pulled out and specified as a processing value, accessible by a name of "1".

Patterns support multiple submatches and will place each returned submatch (see the LabVIEW documentation for Match Regular Expression) into the processing values collection named by its submatch position. For another example, now with multiple submatches, let's imagine a URL that links to a specific comment of a specific blog article:

Pattern: "^\/blog\/(\d+)\/comment\/(\d+)$" <br/>
Regexr: https://regexr.com/7rhtt

This pattern now pulls out two numbers: an ID for the blog post and an ID for the comment. The first number, the blog ID, would be available using name "1" and the comment ID with a name of "2" corresponding to their positions. Note that nested submatches are also possible but can be tricky to keep track of what the proper index value is so be sure to experiment if attempting anything beyond basic submatches for the first time. The regular expression handling used in this HTTP server is a slightly customized version that returns (sub)matches as an array and index 0 is always the full pattern match which is why the first submatch in the path pattern starts with a position value of "1" instead of an index of "0". With IG HTTP Server installed this can be experiemented with by quick drop searching for *Regex Submatch Array*.

### Controller Initialization (Controller Handler)

Perhaps a bit confusing with the Controller Handler naming but this is not *the* Controller that runs to generate a response. This step is intended to perform any final common initialization before the selected controller. For example, the built-in submatch parameter values are actually stored in the processing values collection at this step even though the match is confirmed in the previous step.

Generally at this point, all request information is available including what code is intended to generate the response to the request. This makes this step a good candidate for performing standard request logging beyond what the built in trace logs capture or setting up any common parameter values in the processing values container. This step is not expected to be useful for most apps though.

Just like Request Handlers can trigger an early response, this step also has an opportunity to set an early response causing the Controller to be skipped. This capability is mostly planned to be used by built-in authentication mechanisms that will be optionally available to Controllers to automate forcing users to be logged in or have specific security roles to gain access to specific Controllers.

### Controller

This is the workhorse of a processing flow and is intended to contain most of the logic that generates a response to a request. If a controller does not set a response to be sent to the client and error is generated that will result in a 500 error response being sent to the client with default error handling.

Before being registered with the server, Controllers (as with all other Handlers) can be initialized with class private data such as actor enqueuers, unnamed queues, notifiers, events, or other mechanisms that allow interfacing with other components in the server application. This ability to directly initialize Controllers and Handlers is in fact the main reason for the existence of this web server, followed closely by replacing functionality that has been dropped for LabVIEW Web Services running on the NI Web Server such as session handling. With this web server, it is unnecessary (though possible) to use globals (FGVs, AEs, named queues, etc) to interface to the rest of the application. Some examples could be sending a DQMH request or an Actor Framework message to a module that interfaces with a piece of hardware, utilizes an NI driver API, or performs some other action to respond to the request.

The sky is the limit with Controllers. Well perhaps the memory and processing power of the system running the server might say otherwise... Examples of functionality could be:
- reading contents from a file to generate an image to send back to the client
- reading data from some hardware
- setting parameters on hardware
- updating file contents on the server machine
- making web requests to other services
- running automated scoring routines for Summer of LabVIEW challenge entries

Keep in mind that Controllers can be invoked an arbitrary number of times depending on how many clients are accessing the application at once or even if someone is being overly aggressive with their F5 key. For this reason it will often be a good idea to have some kind of centralized component to manage things like hardware interfaces, database access, or other shared resource management. This way there will usually be some form of request queueing to that component and interactions with shared resources can't happen in parallel making it less likely to have race conditions with requests.

#### IG Promises

For a library that provides a consistent response mechanism for retrieving responses from asynchronous requests such as with DQMH requests, actor messages, message queues, or events sent to other loops, check out the [IG Promises Package](https://www.vipm.io/package/illuminatedg_lib_ig_promises/) on VIPM. IG Promises is a library for sending a request to an async component and waiting for a response with timeout and error propagation handled by the library along with type checking and more.

### Response Handling (Response Handler)

This is the last step in the normal processing flow before the response is sent to the client. This is the last chance to set any response headers, perform final validation of the request and/or response, and even an opportunity to modify the response if needed. This is the step where the provided session handling behavior ensures that the session cookies are set properly in the response headers, regardless of what response was generated earlier in the processing flow. The Early Response flow also leads here instead of directly to sending the response so that headers can be set even on early responses to enable redirect support for logins and other automation. Another example of built-in handling performed at this step is verifying that a file actually exists on disk when file serving has been enabled and no other controllers matched the request path.

### Send Response

Pretty self explanatory, this is where the generated response is sent back to the client. Depending on the code path that was followed during processing this could be an Early Response, a response generated by a Controller, or a response generated by error handling. For normal web requests the response is sent and the connection is closed though reusing connections when the client specifies the "Connection: keep-alive" header is on the short-list for next features to implement. WebSocket connection upgrades sent the upgrade response and leave the connection open so that the connection can later be handed off to WebSocket handling code and the request processing can be completed.

Some responses can be streamed to the client, that is that the entire data isn't loaded into memory at once and then sent to the client; the data is sent in chunks. This limits the amount of memory that the server needs at once and primarily enables serving large files from disk without overly burdening the server system's memory. If the size of the content is known ahead of time, such as with an existing file on disk, the response is sent as a normal response with the file content sent in 10K segments directly as the response body. If the total response size isn't known ahead of time then chunked transfer-encoding is used.

Utilizing chunked transfers, which enables javascript code running in a browser using the Fetch API to receive and handle each chunk as it's transferred, requires the creation of new Stremable Response children that are aware of the streaming capability. A contrived example might be a specialized Oscilloscope Stream Response class that continuously reads waveforms from hardware and returns the waveform data as chunks for a predetermined amount of time. However this direct interaction with hardware means that only 1 request could be serviced at a time so a better solution would be that anytime one of these streamed responses are active, a centralized actor could make its waveform data available to the streaming response and this would enable support for multiple clients being sent the same data. This is also likely to be simpler to accomplish using WebSocket streaming.

### Complete (Complete Handler)

This is now the last step of a well-behaved request processing session as far as extensible code is concerned. For built-in functionality this step is used to cleanup session handling related to the request, finalize response caching (once fixed at least), and for WebSocket upgrades, passing the connection off to WebSocket connection processing. In the case of WebSocket connections, the WebSocket continues to be managed by a separate async VI allowing the request processing code to finish running and report the processing trace log.

### Finish Request

At the time of writing this document, this is the last step and final cleanup of the entire process. The trace log is finalized and enqueued and the request is cleaned up including deleting any temporary files from multipart form uploads that still remain in the original temporary upload location. 
