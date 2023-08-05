# API Gateway

Expose your serverless application as a REST API.
...or many others! GW architectures support placing a unified facade over a collection of many backend services.

AWS API Gateway is a great reference implementation of an API GW and their core features.

* authentication
* development plans/environments: dev, prod, test...
* web socket protocol support
* API Versioning: v1, v2, ...
* Security (authentication and authorization)
* Create api keys and request throttling
* Swagger/OpenAPI import to quickly define API
* Transform and validate requests
* Generate SDK and API specs
* Cache API responses

Integrations

* Lambda, to expose a serverless REST api
* HTTP: expose http endpoints, internal/on-prem or other, add rate-limiting, etc.
* AWS Service: exposes any AWS service to utilize the features of API GW
  * Example: client -> API GW -> kinesis DS -> Firehose -> S3

Endpoint Types:

* edge-optimized (default): for global clients
  * requests are routed through the CloudFront Edge
  * The API GW still lives in only one region
* region: for clients within the same region as GW
  * manually combine with CloudFront
* private: accessible only from within your vpc and ENI endpoints

Security:

* User authentication through:
  * IAM Roles (useful for internal apps)
  * Cognito (mobile and external users)
  * Custom authorizer (your own logic)
* Custom domain name: HTTPS security through ACM
  * If Edge-optimized, then the cert must be from us-east-1
  * If using regional endpoint, the cert must be in GW region
  * Must setup CNAME or A-alias record in Route 53

### Example

Build one of the following:

* REST
* Web socket
* some others....

REST:

* name the API and select endpoint type (regional, edge optimimzed, or private)
* define REST methods: GET /, DELETE /thing, etc
* integrate methods with services or endpoints: ie, create a backend lambda to serve.
* permissions to invoke lambda will be configured automatically via a Resource-Based Policy: API GW is allowed to access the lambda func
* configure stages (optional): effectively allows you to set up a dev API GW

### Deployment Stages

Making changes to GW does NOT mean they are in effect; you must make a Deployment. Changes are deployed in stages; each stage has its own parameters.

Example:

* V1 clients use <https://api.example.com/v1>, backed by V1 stage within GW.
* We create V2 stage, for V2 clients using <https://api.example.com/v2>, backed by V2 GW stage and V2 code

Stages ensure that we can implement this pattern, migrate clients from v1 to v2, and then shutdown the V1 api gracefully.

Stage Variables: just like env vars, but for stages.
These variables abstract away the volatile/variable values that you want to represent with stages:

* lambda function ARN
* HTTP endpoint
* parameter mapping templates

Usecases:

* configure HTTP endpoints your stages talk to (dev, test, prod)
* pass configuration params to AWS Lambda

Stage variables are passed to the "context" parameter in Lambda

* format: "${stageVariables.variableName}

Example: create a **stage variable** to indicate the corresponding Lambda func alias. Our API GW will automatically invoke the right lambda function.

```
  prod stage  ->  Lambda_PROD  ->  V1  (95% V1; 5% V2)
  test stage  ->  Lambda_TEST  ->  V2
  dev stage   ->  Lambda_DEV   ->  $LATEST
```

NOTE: the aliases are setup in LAMBDA. We then point to those aliases (versioned lambda funcs) from GW using the variable '${stageVariables.lambdaAlias}'

* ensure that you have the right Function Policy for all aliased lambda funcs: each aliased lambda has-a Function Policy

Deployments (aka Stages) have stage-variables; thus their is a compositional relationship of deployments to stages, and stages have a references-a relationship to aliased lambdas.

### Canary Deployments

Choose the % of traffic the canary channel receives, for any stage.
Metrics and logs are separate, such that the health of each deployment can be evaluated clearly before swapping over.

### API GW Integration Types

* MOCK: API GW returns a response without sending anything to any backend (ie, 200-OK with no data).
* HTTP / AWS: you must configure both the integration request and the integration response
  * setup data mapping using **mapping templates** for the request and response
* AWS_PROXY (lambda proxy): incoming request from the client is the input to the backend lambda. No mapping template, headers, query string parameters... are passed
as arguments.
* HTTP_PROXY: your own implementation; no mapping template, http request is passed directly to backend, response is forwarded via the GW.
  * possible to add HTTP header if needed (API key)

### Mapping Templates

Used to modify the request/response; not used/available for proxy-based methods.

* rename/modify query string parameters
* modify body content, add headers
* user the Velocity Template Language (VTL): for loop, if, etc...
* Filter output results (remove unnecessary data)
* Content-type can be set to application/json or xml

Exam: how to integrate a SOAP api with a REST frontend.
Answer: SOAP are xml based, whereas REST is json-based. Use a mapping template to transform REST json data to SOAP xml backend request.

Query string example: rename query string variables in any manner desired.

### Open API Spec

A common way of defining REST apis, using API definition as code.
You can import OpenAPI 3.0 spec to API gateway (1:1):

* method, method request
* integration request
* method response

You can export current API as OpenAPI spec.
OpenAPI spec can be written in YAML or JSON.
Using OpenAPI we can generate SDK for our apps.

Also implements api validation: if a request does not adhere to schema, reducing unnecessary calls to backend. Checks:

* the required request parameters in the URI, query string, and headers are included and non-blank
* the application request payload adheres to the configured json schema request model

This is setup by importing OpenAPI definitions file. Define the validators and conditions.

### Caching

Reduce number of backend calls, per Stage.

* default TTL is 300s (min 0, max 3600s)
* possible to override cache settings **per method**
* cache capacity 0.5GB to 237GB
* cache is expensive and only makes sense for prod, not dev/test

Invalidation: possible to invalidate entries immediately from UI, or using a header **Cache-Control:max-age=0** (with proper IAM authorization). Note that invalidation is per-entry; clients can't invalidate the whole cache for an endpoint.

* NOTE: if you don't impose an InvalidateCache policy (or choose the Require authroization checkbox in console), then ANY client can invalidate api cache.

### Usage Plans and API Keys

Usage plans:

* who can access one or more deployed API stages and methods
* how much and how fast they can access them
* uses API keys to identify API clients and meter access
* configure throttling limits and quota limits enforced per client

API keys: alphanum string to distirbutes to your customers. Can use with usage plans to control access. Throttling limits are applied to the api keys.

Correct order of creation for API keys:

1) Create one or more apis, configure the methods to require an API key, and deploy the APIs to stages.
2) Generate or import API keys to distribute to ap devs who will be using your api.
3) Create Usage Plan with throttles
4) Associate API stages and API keys with the usage plan.

* Callers of the API must supply an assigned API key in the **x-api-key** headers to API requests.

### Logging and Monitoring

CloudWatch logs:

* log contains information about request/response body
* enable CW at the Stage level (with Log Level: dbg, error, info)
* can override per api

X-Ray: tracing information through the API GW, with full path through backends.

#### Metrics

Metrics are per Stage, and it is possible to enable detailed metrics.

* CacheHitCount, CacheMissCount
* Count: api requests in a given period
* IntegrationLatency: time it takes to forward a request and receive response from backend.
* Latency: total latency from the time API GW receives request to the time it sends response
  * Max is 29s: otherwise clients will receive timeouts
* 4xx/5xx errors: client and server side error count, respectively.

#### Throttling

Account limit: 10000 reqs/s; can be increased on request.
Throttling: over request limit, clients will receive http-429 (retriable)
Stage and Method limits: these can be configured to ensure quotas are not exceeded.
Usage plan: define to limit per customer.

#### Errors

4xx: client errors, 400=bad request, 429 too many requests, 403 WAF filtered or not authed.
5xx: server errors, 502 = lambda proxy did not respond well, 503 service unavailabe exception, 504 integration failure, API GW timed out (>29s)

### CORS

Origin = scheme, host, port

[Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) is an HTTP-header based mechanism that allows a server to indicate any origins (domain, scheme, or port) other than its own from which a browser should permit loading resources. CORS also relies on a mechanism by which browsers make a "preflight" request to the server hosting the cross-origin resource, in order to check that the server will permit the actual request. In that preflight, the browser sends headers that indicate the HTTP method and headers that will be used in the actual request.

Example: host a static website from an S3 bucket on origin A, whose links/requests call into your API GW.

Must be enabled if you want to receive API requests from another domain.

Preflight request must contain headers:

* Access-Control-Allow-Headers
* Access-Control-Allow-Method
* Access-Control-Allow-Origin

CORS can be enabled through the console.
To enable for a server endpoint in the GW, define the accepted methods, endpoint, and headers it will accept, and the allow-origin variable.

* NOTE: if using a lambda proxy, the configuration is on the lambda response.

### Security

1) IAM Permissions:
    * create an IAM policy authorization and attach to user/role.
    * Authentication: IAM
    * Authorization: IAM policy
        * Leverages the SigV4 method to pass signed credentials to the GW from client

    Resource Policies: much like lambda resource policies. These allow for cross-account access.

    * allow specific IP addresses or VPC endpoints
    * ALLOW -> Principal (some IAM user) -> Resource = your endpoint

2) Cognito Pools
    Fully manages user lifecycle, tokens expire.
    No custom implementation required.

    * Authentication: Cognito User Pools
    * Authorization: API GW methods

3) Lambda Authorizer: custom authorization

    * token based authorizer (JWT)
    * request parameters and query string are passed to lambda authorizer, which returns an IAM user, result policy is cached.
    * authentication: external via lambda authorizer
    * authorization: lambda function

    Used primarily with 3rd party authentication mechanisms.

When to use each solution:

* IAM: users already within AWS account, handle authentication and authorizaiton, uses SigV4
* Custom authorizer: when we want to handle authentication and authorization in our own implementation. Very flexible, but pay per lambda invocation.
* Cognito User Pool: manage your own user pool, no need for custom code, must implement authorization backend.

### HTTP and WebSocket api

HTTP: low latency, cost-effective lambda proxy, with no mappings available. Supports CORS, but only OIDC and OAuth 2.0. Same as trad API GW, but simpler and cheaper. No support for Resource Policies.

REST: all features except Open Id Connect.

WebSocket: two way interactive communication (server push option). Enables stateful app use-cases for realtime apps (chat, trading apps, games).
One persistent connection.
Backend (behind API GW) is separated via lambdas:

* Lambda on-connect: invoked on new websocket connection
* Lambda message: send message

Connecting to the API: connecting is done via a special url, formatted as:

* wss://[unique id].execute-api.[region].amazonaws.com/[stage]

The handshake:

1) invokes backend connection lambda, which is passed connection-id
2) client then sends-messages (frames): these invoke new lambdas
with the same connection id.

Server to client messaging:

* using the same special url...
* BUT, lambdas can invoke an extension of the "wss://..." url to send info back
  * wss://[unique id].execute-api.[region].amazonaws.com/[stage]/@connections/[connection id]
    * This url supports verbs:
      * GET: get conn status
      * POST: send data to client
      * DELETE: disconnect the connection from the client

Routing: websockets use routing to direct messages along socket paths to backend lambdas.

* "route key table" in the GW maps socket data "action" fields to lambdas; thus they mapped the protocol to backend lambda routing.

FUN: use 'wscat' tool to test websocket apis!
