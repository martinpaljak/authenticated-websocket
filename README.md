# Authenticated WebSocket

Modern web applications are [single-page applications](https://en.wikipedia.org/wiki/Single-page_application) that often use (or could use) [WebSockets](https://en.wikipedia.org/wiki/WebSocket) to communicate with the backend system. As an upgrade to HTTP, WebSockets can utilize most of the client authentication methods available to HTTP applications, such as usernames and passwords with authentication cookies or client certificate authentication, but also inherit the same shortcomings, either in security level or usability.

Authenticated WebSocket is not a separate protocol (or a [WebSocket subprotocol](https://tools.ietf.org/html/rfc6455#section-1.9)) _per se_, but a simple implementation convention, based on the [challenge-response](https://en.wikipedia.org/wiki/Challenge%E2%80%93response_authentication) mechanism.

The main purpose of this convention is to support classical PKI X509 client certificate based authentication with a smooth and controllable user experience. While this reference implementation is for [NodeJS](https://nodejs.org/en/download/current/) and utilizes [X509 ID tokens](https://github.com/martinpaljak/x509-webauth/wiki/OpenID-X509-ID-Token) (a profile of [OpenID Connect ID token](http://openid.net/specs/openid-connect-core-1_0.html#IDToken)), the convention does not restrict the type of [JSON Web Tokens (JWT)](https://tools.ietf.org/html/rfc7519) that could be used to transport the authentication claim, as long as the token contains the `nonce` field.

## The convention
- Authenticated WebSocket MUST use a secure WebSocket connection (`wss://`)
- The application SHOULD check the `origin` of the connection to match the acceptance criteria of the application, before processing the authentication messages
- The first message from the server to the client MUST be a text frame with a JSON object with a `nonce` field as a string
- The first message from the client to the server MUST be a text frame with a JSON object with a `token` field as a string
- The first message from the server to the client or from the client to the server MAY contain other fields in the JSON object, than the ones described above
- The `nonce` field of the first message from the server to the client MUST be unique to the session and with enough entropy, e.g. random data fed to an approved hash algorithm or coming from a well-seeded CSPRNG.
- The `token` field of the first message from the client to the server MUST contain a JWT with a `nonce` field in the decoded payload, which value matches the `nonce` field of the first message from the server to the client
- The server application MUST check that
  - the `nonce` field of the JWT payload MUST match the nonce sent by the server
  - the `aud` field of the JWT payload MUST match the origin of the WebSocket connection and that the origin is in the scope of accepted origins for the application
  - the `iat` and `exp` fields of the JWT payload MUST be within the accepted time window for the application
  - the signature of the JWT payload MUST be verified and that the used algorithm MUST be approved for the application
  - if the JWT contains a `x5c` certificate in the header, the signature of the JWT MUST match the certificate and the application MUST perform any necessary certificate validity checks, such as OCSP
- Either the client or the server SHOULD close the WebSocket if any of the conditions above are not fulfilled

After the handshake described above, the WebSocket session is considered as authenticated until either it is closed or an application-defined time has passed.

## Sample messages
- From the server to the client
  - `{"nonce":"993452eb-e39d-4bdb-b6e5-e111edd57348"}`

- From the client to the server
  - `{"token":"
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsIng1YyI6WyJNSUlGb3pDQ0E0dWdBd0lCQWdJUUhGcGRLK3pDUXNGVzRzY09xV1pPYURBTkJna3Foa2lHOXcwQkFRc0ZBREJqTVFzd0NRWURWUVFHRXdKRlJURWlNQ0FHQTFVRUNnd1pRVk1nVTJWeWRHbG1hWFJ6WldWeWFXMXBjMnRsYzJ0MWN6RVhNQlVHQTFVRVlRd09UbFJTUlVVdE1UQTNORGN3TVRNeEZ6QVZCZ05WQkFNTURrVlRWRVZKUkMxVFN5QXlNREUxTUI0WERURTJNRE14TVRFek1qUXpNRm9YRFRFM01URXlNekl4TlRrMU9Wb3dnWk14Q3pBSkJnTlZCQVlUQWtWRk1ROHdEUVlEVlFRS0RBWkZVMVJGU1VReEZ6QVZCZ05WQkFzTURtRjFkR2hsYm5ScFkyRjBhVzl1TVNJd0lBWURWUVFEREJsUVFVeEtRVXNzVFVGU1ZFbE9MRE00TWpBM01UWXlOekl5TVE4d0RRWURWUVFFREFaUVFVeEtRVXN4RHpBTkJnTlZCQ29NQmsxQlVsUkpUakVVTUJJR0ExVUVCUk1MTXpneU1EY3hOakkzTWpJd2dnRWpNQTBHQ1NxR1NJYjNEUUVCQVFVQUE0SUJFQUF3Z2dFTEFvSUJBUUNzQ0djVE92SGI0NGtiT29JSmpibW10ZElMMXFMUFR4ZUJIV3BDakhLWE5WeVc3eHU4NGRSS0ZlQWd1ZTQrYXVON3FKb3JBeTdoRUx0WjFBSE9kQVdLQ0xDTC94RmpLSmcvVHFMa0x3L0N2eGRpQWZhbFhyK3drbjVVRmZUNnRjSFNvL1hmNjMzN0RQSFNncTBuMVlTVTJtNTIyQlhVcjg3RDRIbDBvMlVKS2ZvakJWS0FSTnRrQVVqZkE3OE5ZQnJKL3YxejNZNGszZUxKbVRweE5hR29XRGVPVUhlbUorMERxaS8rUXR6RHllMWgwSzQzS0t2VTAzWXFWeDZ1S0N0dWpQUW55UTZjdGR1UzdJYTJxcDZuWHhBdEhicFMzSnB1Um54c29KbWRBTk5vZm1UeGtucGFId05wNWNjYnpIdmptOTVlSTZhOHJ2RU1sS2huQWplQkFnUW1vQWFybzRJQkh6Q0NBUnN3Q1FZRFZSMFRCQUl3QURBT0JnTlZIUThCQWY4RUJBTUNCTEF3T3dZRFZSMGdCRFF3TWpBd0Jna3JCZ0VFQWM0ZkFRRXdJekFoQmdnckJnRUZCUWNDQVJZVmFIUjBjSE02THk5M2QzY3VjMnN1WldVdlkzQnpNQ0VHQTFVZEVRUWFNQmlCRm0xaGNuUnBiaTV3WVd4cVlXdEFaV1Z6ZEdrdVpXVXdIUVlEVlIwT0JCWUVGTCtsYzNsMWl4QjFaeVBlQU5Bdk1jN05IWE1aTUNBR0ExVWRKUUVCL3dRV01CUUdDQ3NHQVFVRkJ3TUNCZ2dyQmdFRkJRY0RCREFmQmdOVkhTTUVHREFXZ0JTenE0aThtZFZpcElVcUNNMjBIWEk3ZzNKSFVUQThCZ05WSFI4RU5UQXpNREdnTDZBdGhpdG9kSFJ3T2k4dmQzZDNMbk5yTG1WbEwyTnliSE12WlhOMFpXbGtMMlZ6ZEdWcFpESXdNVFV1WTNKc01BMEdDU3FHU0liM0RRRUJDd1VBQTRJQ0FRQnZaMXBHWTB2N2dNQW5GZUVpcmtxRzAvRW42eFVrVkkyY3R5cUxSMk9ZNDlxdCtYOGdObHJmd1l0VlRLUlJNTjRGWllKejhDM0hzZFR5RTlpWUZLSjFOemhnL1hNMStTQlIyRkxBaXZYRjJJRURTUnQvNTkwVkhMYjJ4RVVGSmFDR1ptdWJZTEZFSDRMMzAreHdUWXZ6aUR2NG5jZnAwa3lBS09tVzQxbDhlOFNaM2MzN2lsQ1B3RHk1RUw0ZlJxbUwxSmlxaHRwdXpXeGNVL3Nla04rSnY2ZWkwLzlnSkwwYkJISTNEcHIvUzhyenVwRVlLZktjR3p0VmFzTEJJa1VYcloyZGhJazhObG9OZ3F0Q0VodDRoaElOZWs3aTVEWStvUnZpVjdqUUVLY0dCQXhmQkZIZXRjc1R3LzkvZDRUUEZFQXpHTCtGWUQrTUlKQ2dSV2dzcyt5RVArUzdhQUhRNzhveXFyZE10OTRSUDFneUE4ZEdCYU9DRUFxVU5jdW9MOWg1Um1jWUtzeWJPUEJYZy8xOXEyQUgvc1BpRHBtV1RLZytDUFVWUnAwcGE4aWl5Z1BKVmJJSmEwSHhaZm4rNmwwUW9hZ084SzBpdXZpdFkyUkhGbGRnUjJ3VHZUek01UUNiY0NRSndxaDV1R3l3cEdEcWFLa2VseUovNkhMalZyUS9Ram0zb2R6QThjRHhKWEhUZTJrSE96M3diNUE2RTZQd3FLeUVMb3ZNOXFqb2VUbUJsd280M0NwUUY2bkI2MGJVOEV1TExTWHI3VlJGSk52b1JPOXlGVDY2NExydE1XcGY4RTdwRldWZGo3dnZPd2N3TGJSR2tUdXlabnlDcElaQ0VyanlKdWp4bXVNNUdpRERYMFVqRWdIOWFiRFR1bDVadWNWa2lnPT0iXX0.eyJhdWQiOiJodHRwczovL2V4YW1wbGUuY29tLyIsImV4cCI6MTQ4Nzc3NDIxNiwiaWF0IjoxNDg3NzczOTE2LCJpc3MiOiJFU1RFSUQtU0sgMjAxNSIsIm5vbmNlIjoiZmRkYTZhNjItMWJhNS00NWU3LTk2NjktNWZiODc3ODY0ODYxIiwic3ViIjoiUEFMSkFLLE1BUlRJTiwzODIwNzE2MjcyMiJ9.DBG5RQbmmr9tBZaSOrWrrK9aWtxC5lCG2GPgWKPZkmLGi_BLSNGhNS9Zvi9il24IwwdAYw1yHeyRx5ekomJDi3V47R54EcikTRbyBTUolE8CRbJK97ErJLVlsoC-M1E9XeWuMnP6IE_qO8JJtPN3JOS6FXP-8MlTHwCh1tkFNgcynn7pXVjv-1BmFQ7Vnohl6uMCOx1Z9WfJzfeWdrFaruKrv3k-Hr-qi3d-z-eGf2ryN1RVZJSppq8xop5buoJelroj1Fu5jBxAes0DOGaSSMce96idiKSHr6-75uqCl2V4tbbUhrMB8AgGms5ms7PAIcmJeo1XxjEmC5p79LFAvg"}`

## Using `authenticated-websocket` from NodeJS
- It is expected to be used behind a TLS offloading server, such as [NGINX](https://www.nginx.com/blog/websocket-nginx/).
