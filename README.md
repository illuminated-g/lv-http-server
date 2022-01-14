# lv-http-server

HTTP Server implementation in LabVIEW. Utilizes IG TCP Streams and provides mechanisms akin to Web Services.

This server provides flexibility and extensibility via a handful of specific request processing steps that can have additional functionality plugged in. The following flowchart maps out the processing flow.

<details>
    <summary>Expand for Processing Flowchart</summary>

![Request Processing Flowchart](https://raw.githubusercontent.com/illuminated-g/lv-http-server/main/Documentation/images/Request%20Flow.drawio.png "Request Processing Flowchart")
</details>

<br/>
Aside from Controllers, which can certainly grow quite large, the expectation is that the other handlers should be small and run quickly. In the flowchart above, each of the blue shapes represents an interface point and except for the Controller step, is a loop to run through all registered handlers for that step.

<br/>
For instance, the following code of a Response Handler is nice and small and serves a single purpose:

![FileExistChecker Code](https://raw.githubusercontent.com/illuminated-g/lv-http-server/main/Documentation/images/fileexistchecker.png "FileExistChecker Code")
<br/>
The only point of this handler is to verify that the specified file exists when the Response is a FileResponse. It doesn't generate a replacement Response itself, instead it returns an error 7 to allow the exception handling functionality to generate the 404 error. This also lets the functionality for whether the full error string will be sent to the client or not based on whether debugging is enabled or not to live in one place and not have processing code strongly coupled to error handling and output code.