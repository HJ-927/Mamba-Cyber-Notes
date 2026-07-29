The user interacts with the input device such as the keyboard. The keyboard sends signals to the operating system (OS). The OS then delivers the keyboard events to the active application, for example, Chrome.

Chrome interprets the URL and identifies information such as the scheme, hostname, port, path, query string, and fragment.

Chrome then asks the OS name resolution facilities to resolve the hostname into the correct IP address. The OS name resolution facilities may look up local sources such as /etc/hosts, the DNS cache, or query a DNS server, depending on the system configuration.

After obtaining the correct IP address, the OS networking stack establishes a TCP connection between the client and the server using the destination IP address and port.

For HTTPS requests, the client performs a TLS handshake with the server. TLS protects the communication channel by providing confidentiality, integrity, server authentication, and optionally client authentication.

Chrome then constructs the HTTP request according to the HTTP protocol and serializes it into bytes.

Chrome asks the OS networking stack to transmit those bytes to the server through the established TCP connection.

Once the application data reaches the server, there may be several components in front of the application, such as a CDN, load balancer, reverse proxy, or web server. For HTTPS, TLS termination may occur at one of these components.

The HTTP parser reconstructs the received bytes into structured HTTP request information according to the HTTP protocol and validates that the request follows the protocol syntax. For example, an invalid request line such as POST/login (without a space) would fail HTTP parsing.

Based on its configuration, the web server or reverse proxy may perform routing decisions or preliminary processing, such as checking the hostname, path, request headers (including the Authorization header for HTTP Basic Authentication), or selecting the correct backend application.

For static resources, the web server usually handles the request directly because no application logic is required to generate the response. It simply reads the requested resource and returns it.

For dynamic requests, application logic is required to generate the response. Therefore, the reverse proxy or web server forwards the request to the appropriate backend application.

Inside the application, there may be global middleware. Global middleware is a chain of functions that can inspect, modify, add, or remove information from the request. Examples include session handling, logging, error handling, security headers, or request tracing.

The request then reaches the router. The router matches the request based on information such as the HTTP method, path, hostname, route constraints, and other routing rules. For example, if the route is /dashboard/{id} and {id} only accepts numeric values, a request containing a non-numeric value will not match that route.

Route groups may be used when multiple routes share a common prefix or configuration. Route-specific or route-group middleware may also exist. Like global middleware, these are chains of functions that can inspect or modify the request before it reaches the handler. Examples include authentication, authorization, rate limiting, or CSRF protection.

If all routing and middleware requirements are satisfied, the router passes the request to the appropriate handler.

Inside the selected handler, there may be several validation steps. For example, checking file extensions, validating file size, verifying that parameter values fall within an allowed range, or ensuring required fields are present.

There may also be normalization. Normalization converts accepted input into a consistent or preferred representation before it reaches the core business logic. Examples include trimming whitespace, converting text to lowercase or uppercase, URL decoding, or canonicalizing file paths. Depending on the application, normalization may occur before or after validation, and not every application performs explicit normalization.

The input then reaches the business logic. Business logic enforces the application's rules and interacts with one or more backend destinations, such as databases, filesystems, caches, message queues, or external APIs. A single request may interact with multiple destinations.

Some destinations have one or more consumers that interpret or process the input. A destination and a consumer are not the same thing. A destination is where the data goes, while a consumer is the component that later reads, interprets, or processes that data. For example, a PDF stored on a filesystem may later be processed by a PDF parser, search indexer, OCR engine, or simply served back to the browser. Some destinations simply store data without immediately invoking another consumer.

Business logic may also produce side effects. Side effects are backend state changes caused by processing the request. Examples include inserting new database records, writing files to disk, sending emails, writing logs, updating caches, or queuing background jobs. These side effects are often not directly observable from the immediate HTTP response, although they may later be inferred or verified by an attacker through subsequent observations, such as reading database contents via SQL injection, locating uploaded files, or successfully logging in with newly created credentials.

After the business logic finishes, the application constructs the HTTP response.

The response may then pass through response middleware, which can inspect, modify, add, or remove response information before it is sent back to the client.

Finally, the response is serialized into bytes and sent through the reverse proxy or web server over the existing TCP connection. On the client side, the OS networking stack delivers the response to Chrome, which parses the HTTP response and renders the content for the user.
