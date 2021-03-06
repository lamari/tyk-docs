---
date: 2017-03-23T18:08:16Z
title: Virtual Endpoints
menu:
  main:
    parent: "Compose APIs"
weight: 0 
---

> **NOTE**: Virtual endpoints are not available in the Tyk Cloud Edition.

Virtual endpoints are unique to Tyk. With a virtual endpoint, you can plug short JavaScript functions at the end of a Tyk route and have them run when the endpoint is called.

> **NOTE**: Virtual endpoints and the JSVM middleware share the same API. See [JavaScript API](/docs/customise-tyk/plugins/javascript-middleware/javascript-api/) for more details. 

A sample use case for this might be aggregate functions that bring together related data from multiple services in your stack into a single object.

Alternatively, you could produce a dynamic response object that transforms or computes data in some way from upstream services.

> **Note**: The JavaScript engine in which these methods run in is a traditional ECMAScript 5 compatible environment and does not offer the more expressive power of something like Node.js. These methods are meant to provide a functional interpreter before complex interactions with your underlying service that cannot be handled by one of the other middleware components.

#### Virtual Endpoint Functions

To create one of these methods, create a file and place it in a subdirectory of the Tyk configuration environment (ideally under the `middleware` folder in your Tyk installation). Here is a sample method:

```{.copyWrapper}
function sampleVirtual (request, session, config) {
    log("Virtual Test running")
    
    log("Request Body: ")
    log(request.Body)
    
    log("Session: ")
    log(session)
    
    log("Config:")
    log(config)
    
    log("param-1:")
    log(request.Params["param1"])
    
    var responseObject = {
        Body: "THIS IS A  VIRTUAL RESPONSE",
        Headers: {
            "test": "virtual",
            "test-2": "virtual"
        },
        Code: 200
    }
    
    return TykJsResponse(responseObject, session.meta_data)   
}
log("Virtual Test initialised")
```


### An Aggregate JS Function

The most common use case for this functionality, as we see it, is to provide some form of aggregate data to your users, here's a snippet that will do just that using the new batch processing API:

```{.copyWrapper}
function batchTest (request, session, config) {
    // Set up a response object
    var response = {
        Body: ""
        Headers: {
            "test": "virtual-header-1",
            "test-2": "virtual-header-2",
            "content-type": "application/json"
        },
        Code: 200
    }
    
    // Batch request
    var batch = {
        "requests": [
            {
                "method": "GET",
                "headers": {
                    "x-tyk-test": "1",
                    "x-tyk-version": "1.2",
                    "authorization": "1dbc83b9c431649d7698faa9797e2900f"
                },
                "body": "",
                "relative_url": "http://httpbin.org/get"
            },
            {
                "method": "GET",
                "headers": {},
                "body": "",
                "relative_url": "http://httpbin.org/user-agent"
            }
        ],
        "suppress_parallel_execution": false
    }
    
    log("[Virtual Test] Making Upstream Batch Request")
    var newBody = TykBatchRequest(JSON.stringify(batch))
    
    // We know that the requests return JSON in their body, lets flatten it
    var asJS = JSON.parse(newBody)
    for (var i in asJS) {
        asJS[i].body = JSON.parse(asJS[i].body)
    }
    
    // We need to send a string object back to Tyk to embed in the response
    response.Body = JSON.stringify(asJS)
    
    return TykJsResponse(response, session.meta_data)
    
}
log("Batch Test initialised")
```

The above code is pretty self explanatory, so we won't go into great detail - the batch object here is the same object that is fed into our batch request method `TykBatchRequest` that is exposed as part of certain API definitions.

### Adding Virtual Endpoints to your API Definition

Virtual endpoints follow the same layout and setup as other elements in the `extended_path` section of the API definition:

```{.copyWrapper}
...
virtual: [
    {
        response_function_name: "batchTest",
        function_source_type: "file",
        function_source_uri: "middleware/testVirtual.js",
        path: "get-batch",
        method: "GET",
        use_session: true
    }
]
```

The parameters are as follows:

*   `response_function_name`: This is the function to run when this virtual endpoint is requested. The function name must match exactly (including casing) the function name in your virtual middleware. We need to know this as it will be the entry point into your code, this will be called first. Make sure it is unique, all plugins run in the same VM, so if there are naming collisions you may end up with unpredictable behaviour.
*   `function_source_type`: This can be `file` or `blob` If set to `file`, then Tyk will pre-load the JS from disk, if set to blob, then Tyk will base64-decode a string from the `function_source_uri` section.
*   `function_source_uri`: This will be the relative path to the source of the functionality (e.g. `myfile.js`), or a blob of base64-encoded data that represents the same information. Blob mode is mainly used by the dashboard to make code injection easier on multiple node deployments.
*   `path`: This is the relative URI path to which the virtual middleware will respond. For example, `http://domain/path`.
*   `method`: This is the HTTP verb (`GET`, `POST` etc.) to which this virtual middleware will respond.
*   `use_session`: If true then the key session data will be provided to the function as the `session` variable. See the plugins documentation for more detail about this object.

### Passing Custom Attributes to Middleware

You can use the `config_data` special field in your API definition to pass custom attributes to middleware via a virtual endpoint.

#### Adding `config_data` to an API Definition

Add the following to the root of your API definition:

```{.copyWrapper}
"config_data": {
    "foo": "bar"
},
```

#### Sample use of `config_data`

```
function testVirtData(request, session, config) {
    var resp = {
        Body: request.Body + " added body",
        Headers: {
            "data-foo": config.config_data.foo
        },
        Code: 202
    }
    return TykJsResponse(resp, session.meta_data)   
}
```

