# FastAPI Deconstructed

## Anatomy of a Modern ASGI Framework

Recently I had the opportunity to talk about the FastAPI under the hood at PyCon APAC 2024. The title of the talk was “FastAPI Deconstructed: Anatomy of a Modern ASGI Framework”. Then, I thought why not have a written version of the talk. And, I have decided to write. Something like a blog post. So, here it is.

---

FastAPI has quickly become one of the go-to frameworks for Python developers who need high performance and developer-friendly API frameworks. With support for asynchronous programming, dependency injection, and automatic OpenAPI documentation, FastAPI stands out for its speed and ease of use. This post will break down the core components of FastAPI, detailing how each part—from ASGI and Uvicorn to Starlette and Pydantic—works together to create a robust, modern web framework.

### Hello World

Let’s begin with the fundamentals of a FastAPI application. A “Hello World” example in FastAPI is very straightforward.

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def hello():
    return {"message": "Hello, World!"}
```

With a simple setup like this, FastAPI takes care of:

- Defining an asynchronous route.
- Parsing and validating requests.
- Serializing JSON responses.
- Generating automatic API docs.

Here’s how we can run this application.

```bash
uvicorn main:app
```

```bash
hypercorn main:app
```

```bash
granian --interface asgi main:app
```

We can see, there are multiple ways to run our application. The main thing is, we need an ASGI compliant server to run our application. We can use any server that implements ASGI protocol. But for simplicity, in this post I will use `uvicorn` as the example of ASGI server to explain related things. 

---

### Building Blocks

FastAPI’s functionality is layered on top of several powerful components:

1. **ASGI**: The asynchronous protocol layer that handles communication between the server and the application.
2. **Uvicorn**: A high-performance ASGI server that serves FastAPI applications.
3. **Starlette**: An ASGI framework providing routing, middleware, and request/response handling.
4. **Pydantic**: A library for data validation and parsing, used in FastAPI to ensure data consistency and reliability.
5. **Dependency Injection:** A built-in dependency injection system that makes it easy to inject dependencies like database connections, services, or configuration etc.
6. **Automatic API Doc:** Automatically generates an OpenAPI specification for API, which provides detailed documentation and interactive features.

---

### ASGI - The Protocol Layer

ASGI, or the Asynchronous Server Gateway Interface, serves as the foundation of FastAPI, enabling asynchronous programming by providing a standardized interface between the application and server. ASGI evolved from WSGI (Web Server Gateway Interface) to support real-time web features like WebSockets and multiple concurrent connections, allowing Python applications to handle high loads without blocking. Currently ASGI protocol describes HTTP/1.1, HTTP/2 and WebSocket.  

![High Level Diagram](basic-diagram.png)

Here’s how a request flow of ASGI application looks like from a very high level. When client sends a HTTP request, the ASGI server accepts the request and parse & translate it to `scope` and `events` (we will see details of `scope` and `events` a little bit later). Then, the ASGI app receive the `scope` and `events` and process the request. Now let’s see some details about the ASGI protocol itself.

**ASGI Components:**

1. **Scopes**: ASGI defines a `scope` for each connection. This is a dictionary containing the connection’s metadata. For HTTP requests, this includes method, path, query string, headers, etc. Each request or connection is encapsulated in a unique scope.
    
    Example HTTP scope:
    
    ```python
    scope = {
        "type": "http",  # The type of connection ("http", "websocket")
        "http_version": "1.1",  # HTTP version
        "method": "GET",  # HTTP method, like GET, POST
        "path": "/hello",  # URL path requested by the client
        "query_string": b"name=John",  # Query string in the request
        "headers": [  # HTTP/Websocket headers
            (b"host", b"example.com"),
            (b"user-agent", b"Mozilla/5.0"),
            (b"accept", b"text/html"),
        ],
        "client": ("127.0.0.1", 12345),  # Client IP address and port
        "server": ("127.0.0.1", 8000),  # Server IP address and port
    }
    ```
    
2. **Events**: ASGI operates on events for handling requests. Events are async functions used to receive incoming data or send outgoing data:
    - **Receive**: An `awaitable` callable that the application calls to receive events (such as HTTP requests or WebSocket messages).
    - **Send**: An `awaitable` callable that the application uses to send responses back to the server.
3. **Lifespan Events**: ASGI also supports lifespan events, which handle startup and shutdown operations. These events allow setup or cleanup tasks (such as initializing or closing a database connection) to run at the server start or stop.

This is a simple ASGI app looks like. No framework, just a simple Python async function.

```python
async def app(scope, receive, send):
    assert scope['type'] == 'http'

    await send({
        'type': 'http.response.start',
        'status': 200,
        'headers': [
            [b'content-type', b'text/plain'],
        ],
    })
    await send({
        'type': 'http.response.body',
        'body': b'Hello, world!',
    })
```

---

### Uvicorn - The ASGI Server

Uvicorn is the ASGI server that powers FastAPI applications. You could run a FastAPI app with any other ASGI server. Uvicorn is designed for speed and efficiency, making it an ideal choice for applications that require high concurrency. Uvicorn is built on top of `uvloop`, a high-performance implementation of the asyncio event loop, which enhances its ability to handle I/O-bound tasks efficiently.

**Request Lifecycle in Uvicorn**:

1. **Accept Connection**: Uvicorn accepts a connection and creates an ASGI scope for the incoming HTTP request, including metadata like headers and method.
2. **Dispatch Request**: The scope is dispatched to the FastAPI application. Uvicorn uses `uvloop` to asynchronously manage the flow.
3. **Receive Data**: Uvicorn processes incoming request data through ASGI `receive` events.
4. **Send Response**: FastAPI responds with an ASGI `send` event. Uvicorn packages the response (status code, headers, body) and returns it to the client.

---

### Starlette - The ASGI Framework Layer

We can’t talk about FastAPI without Starlette. Starlette is a lightweight ASGI framework that provides FastAPI with its core functionality. Starlette serves as the backbone of FastAPI, handling the low-level routing, middleware, and ASGI compatibility, while FastAPI adds Pydantic validation, dependency injection, and additional tools for building APIs efficiently.

### Lifecycle of a Web Request

Now, let’s visualize the full lifecycle of a http request using a `starlette` hello world example as the ASGI app.

1. **Starlette Application** (`app.py`):

```python
from starlette.applications import Starlette
from starlette.responses import JSONResponse
from starlette.routing import Route

# Route handler for "/hello"
async def hello(request):
    return JSONResponse({'message': 'Hello, World!'})

# Defining the routes
routes = [
    Route('/hello', hello),
]

# Creating the Starlette app
app = Starlette(debug=True, routes=routes)
```

2. **Running the App with Uvicorn**

```bash
uvicorn app:app --host 127.0.0.1 --port 8000
```

3. **Client Request**:

```bash
curl http://127.0.0.1:8000/hello
```

This will return the JSON response `{"message": "Hello, World!"}`.

![ASGI Request Lifecycle](asgi-request-lifecycle.png)

Now, let’s follow the request step-by-step, from the moment the client sends an HTTP request to the response being returned.

**Step 1: Client Sends HTTP Request**

The client sends an HTTP request to the server. For example, a `GET` request to the `/hello` endpoint.

```
GET /hello HTTP/1.1
Host: 127.0.0.1:8000
User-Agent: curl/7.64.1
Accept: */*
```

**Step 2: Uvicorn Accepts the Request**

Uvicorn runs a socket server that listens for incoming TCP connections on the specified host/port (e.g., `127.0.0.1:8000`). When an HTTP request arrives, Uvicorn:

- Accepts the TCP connection.
- Parses the HTTP request from the raw TCP data using `h11` (a pure-Python HTTP/1.1 library) or `httptools` (Python binding for the `nodejs` HTTP parser).

Here, Uvicorn will convert the incoming request into ASGI `scope` and `events`.

**Step 3: Uvicorn Converts HTTP Request to ASGI Scope**

When Uvicorn receives an HTTP request, it converts it into an ASGI `scope` object. 

```python
scope = {
    "type": "http",
    "http_version": "1.1",
    "method": "GET",
    "path": "/hello",
    "query_string": b"",
    "headers": [
        (b"host", b"127.0.0.1:8000"),
        (b"user-agent", b"curl/7.64.1"),
        (b"accept", b"*/*"),
    ],
    "client": ("127.0.0.1", 12345),
    "server": ("127.0.0.1", 8000),
}
```

- **Type**: The type of connection, which is `http` for an HTTP request.
- **HTTP Version**: Version of the HTTP protocol (e.g., `1.1`).
- **Method**: The HTTP method used in the request (`GET`, `POST`, etc.).
- **Path**: The URL path requested (e.g., `/hello`).
- **Headers**: A list of header key-value pairs.
- **Client**: The client’s IP and port.
- **Server**: The server’s IP and port.

**Step 4: Uvicorn Passes the Scope to the ASGI Application**

Once Uvicorn has created the ASGI scope, it will start the ASGI application (in this case, Starlette) by calling the application callable:

```python
async def app(scope, receive, send):
    ...
```

Uvicorn invokes the Starlette app, passing in the `scope` object.

**Step 5: Starlette Processes the Request**

Starlette, being an ASGI-compliant framework, takes over at this point. It matches the route (in this case, `/hello`) and invokes the corresponding route handler.

In this case, the `hello` function is called when the `/hello` route is requested. Starlette internally uses the ASGI `scope` to match the incoming request’s method and path with the defined route.

- **Request Object**: Starlette creates an HTTP request object from the `scope` and ASGI events received from Uvicorn.
- **Receiving Events (`receive`):** Starlette receives events that represent parts of the HTTP request, including the request body.

```python
request_event = {
    "type": "http.request",
    "body": b"",  # Request body
    "more_body": False,  # Indicates whether more data will be sent
}
```

The `body` field contains the request body (in case of a POST request), and `more_body` tells the application whether the request body is complete or more data will follow (useful for streaming large files).

- **Response Handling**: The `hello` route returns a `JSONResponse`, which wraps the response data and sends it back as ASGI events.

**Step 6: Starlette Returns the Response**

After processing the request, Starlette sends back the response to Uvicorn by emitting ASGI events like `http.response.start` and `http.response.body`:

1. **Starting the Response** (`http.response.start`):

```python
response_start_event = {
    "type": "http.response.start",
    "status": 200,  # HTTP status code
    "headers": [
        (b"content-type", b"application/json"),
    ],
}
```

This tells Uvicorn to begin sending the HTTP response headers, with a status code of `200` and a content type of `application/json`.

2. **Sending the Response Body** (`http.response.body`):

```python
response_body_event = {
    "type": "http.response.body",
    "body": b'{"message": "Hello, World!"}',  # JSON response body
    "more_body": False,
}
```

This sends the response body containing the JSON message `{"message": "Hello, World!"}`. The `more_body: False` indicates that this is the final part of the body and that the response is complete.

**Step 7: Uvicorn Sends the HTTP Response Back to the Client**

Uvicorn receives the ASGI events emitted by Starlette and translates them into HTTP responses. Specifically:

- **`http.response.start`** triggers Uvicorn to send the HTTP status line and headers (e.g., `HTTP/1.1 200 OK`).
- **`http.response.body`** sends the response body (e.g., `{"message": "Hello, World!"}`) to the client.

Uvicorn closes the connection when it has sent all parts of the response.

---

### FastAPI - The High-Level Framework

FastAPI builds upon Starlette to create a framework that’s ideal for developing RESTful APIs. FastAPI’s focus on asynchronous programming, Pydantic integration for data validation, and dependency injection make it powerful and developer-friendly.

**Key Features:**

1. Starlette-based routing and request handling.
2. Pydantic-based data validation.
3. Dependency Injection system.
4. Automatic OpenAPI and API documentation generation.

### Pydantic - Data Validation and Serialization

FastAPI’s data validation relies on Pydantic, a library that simplifies the handling of complex data types and validation. Pydantic enables FastAPI to enforce strict data validation rules on incoming request data and outgoing response data.

**Pydantic Model Example**:

```python
from fastapi import FastAPI
from pydantic import BaseModel

# Initialize FastAPI app (this is like initializing Starlette)
app = FastAPI()

# Pydantic model for request data validation
class Item(BaseModel):
    name: str
    price: float
    is_offer: bool = None

# Route with path parameters and Pydantic request body validation
@app.post("/items/")
async def create_item(item: Item):
    return {"item": item}
```

In the example above:

1. Client sends a request:

```bash
curl -X POST "<http://127.0.0.1:8000/items/>" -H "Content-Type: application/json" -d '{"name": "Table", "price": 150.0}'
```

2. FastAPI validates the request body and returns:

```json
{
  "item": {
    "name": "Table",
    "price": 150.0,
    "is_offer": null
  }
}
```

3. If a required field (e.g., `name`) is missing, FastAPI will return an automatic validation error with `422` http status code:

```json
{
  "detail": [
    {
      "loc": ["body", "name"],
      "msg": "field required",
      "type": "value_error.missing"
    }
  ]
}
```

Pydantic also converts data types as needed, making it easier to handle complex data without extensive validation code.

### Dependency Injection in FastAPI

FastAPI’s dependency injection system allows modular, reusable code by injecting resources like database connections, authentication layers, or shared configurations directly into route functions.

**Dependency Injection Example**:

```python
from fastapi import Depends

def get_db():
    db = DatabaseConnection()
    try:
        yield db
    finally:
        db.close()

@app.get("/items/")
async def read_items(db=Depends(get_db)):
    return db.fetch_all_items()
```

With `Depends`, FastAPI manages dependencies automatically, enabling clean, modular, and testable code. Dependency injection is especially useful for managing external services, as it allows centralized control of resource lifecycles.

### OpenAPI and Swagger Documentation in FastAPI

FastAPI’s automatic documentation generation feature provides Swagger and ReDoc interfaces without additional setup. By using route definitions, parameter types, and data models, FastAPI creates real-time OpenAPI documentation, making it easy to test and integrate API endpoints.

With documentation available at `/docs` (Swagger UI) and `/redoc` (ReDoc), FastAPI provides developers with a quick and interactive way to explore API routes, making it easier for teams and external developers to work with the API.

### Request Lifecycle in FastAPI

Here’s a summary of how a request flows through FastAPI:

1. **Client Sends Request**: The client sends an HTTP request to the server.
2. **Uvicorn (ASGI Server)**: Uvicorn receives the request and creates an ASGI scope.
3. **Starlette (Routing)**: Starlette routes the request to the correct endpoint based on path and method.
4. **FastAPI Endpoint**: FastAPI processes any dependencies, validates incoming data with Pydantic, and handles the request.
5. **Response**: Uvicorn receives the response from FastAPI and sends it back to the client.

---

### Conclusion

FastAPI’s architecture combines multiple components to achieve a fast, reliable, and easy-to-use API framework:

- **ASGI** is the backbone of modern Python web frameworks. It enables asynchronous operations.
- **Uvicorn** provides efficient connection handling.
- **Starlette** is the core web framework handling routing and middlewares.
- **FastAPI** extends Starlette with data validation via Pydantic, dependency injection & automatic API docs.