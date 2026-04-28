---
title: Best Practices for Secure Error Handling in Go
tags:
  - go
---

#### [Best Practices for Secure Error Handling in Go][https://blog.jetbrains.com/go/2026/03/02/secure-go-error-handling-best-practices/]

**Error in Go** is **Value**, not **Exception**
- Value - expected result
```go
resp, err := getUser(ctx, in)
if err != nil {
	fmt.Errorf("error: %s", err.Error())
}
```
- Exception - interrupt normal execution
```ts
throw new Error("something went wrong")
// ---
try {  
	riskyFunction();  
} catch (e) {  
	console.log(e.message);  
}
```

>**Error values** in **Go** can be passed around (inspect, compose) like other value could causing **security issues**

#### [[microservices-and-distributed-system-are-having-potential-for-security-breaches]]

Go **errors** leak internal information
- The error-handling `if err != nil {return err}` have some vulnerabilities
- paths, SQL queries, credential, stack traces

#### [[Error-creation-and-wrapping]]
