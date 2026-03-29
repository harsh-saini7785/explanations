# Explanations

## ❓ What happens when you hit a URL in the browser??

When you enter a URL in a browser like Google Chrome, the browser first **parses the URL** into its components such as protocol, hostname, port, path, and query parameters.

### 1. Cache Check

Before making any network request, the browser checks available caches — memory cache, disk cache, and service worker cache — to determine whether a valid cached response already exists. If a cached response is found and still valid, it is returned immediately; otherwise, the browser proceeds with a network request.

### 2. DNS Resolution

The browser converts the domain name into an IP address. It first checks the browser cache, then the operating system cache, and if necessary, queries a DNS resolver, which recursively contacts root, TLD, and authoritative DNS servers to resolve the domain.

### 3. TCP Handshake

Once the IP address is obtained, the browser establishes a connection with the server using the **TCP three-way handshake** (SYN, SYN-ACK, ACK), ensuring reliable communication.

### 4. TLS Handshake (HTTPS)

If the request is made over HTTPS, a TLS handshake follows, during which the server provides its SSL certificate, the browser verifies it, and both parties agree on encryption keys to secure the communication.

### 5. HTTP Request

After a secure connection is established, the browser sends an HTTP request containing the method (GET/POST), headers, cookies, and the requested resource path. This request may pass through intermediaries such as proxies, load balancers, and CDNs like Cloudflare before reaching the origin server.

### 6. Server Processing

The server processes the request by executing backend logic, which may include authentication, business logic, and database queries, and then sends back an **HTTP response**. The response contains a status code (e.g., `200 OK`), headers, and the response body — typically HTML for the initial page load.

### 7. Critical Rendering Path

Once the browser receives the response, it begins the **Critical Rendering Path**:

- Parses the HTML to construct the **DOM** (Document Object Model)
- Parses CSS to build the **CSSOM** (CSS Object Model)
- Combines them to create the **render tree**, which represents the visible elements on the page

### 8. Layout → Paint → Composite

The browser then performs **layout (reflow)**, calculating the size and position of each element, followed by **painting**, where pixels are drawn on the screen. After painting, **compositing** occurs, where different layers are combined — often using the GPU for better performance.

### 9. Sub-resources & JavaScript

Meanwhile, the browser continues to fetch additional resources such as CSS files, JavaScript files, images, and fonts — often in parallel. JavaScript execution can modify the DOM and CSSOM, which may trigger additional reflows and repaints.

### 10. Page Becomes Interactive

Once all resources are loaded and JavaScript execution is complete, the page becomes fully interactive. The browser's **event loop** then handles user interactions, asynchronous operations, and rendering updates efficiently.

---

### Summary Pipeline

```
URL → Cache → DNS → TCP → TLS → HTTP Request → Server
    → Response → DOM/CSSOM → Render Tree → Layout → Paint → Interactivity
```
