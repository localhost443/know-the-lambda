# Understand Your Lambda

AWS Lambda is a serverless computing service that allows you to run code in response to various **events** without provisioning or managing servers. To fully understand how Lambda operates, it is essential to explore what an event is and how different events trigger your Lambda function. Lets dig a bit deeper, Shall we?

## So, What is an Event! You say?

An **event** in AWS Lambda is any piece of data that triggers the execution of your Lambda function. Events may come from different sources, including HTTP requests, file uploads, or changes in data streams. Each event provides structured information that the Lambda function processes. The exact **structure** of the event depends on the event source. but please be aware as they needs to be **Structured**

For example, an event from an HTTP GET request might look like this:

```json
{
  "resource": "/",
  "path": "/",
  "httpMethod": "GET",
  "headers": {
    "Accept": "application/json",
    "User-Agent": "curl/7.64.1"
  },
  "requestContext": {
    "identity": {
      "sourceIp": "203.0.113.42"
    }
  }
}
```

This event provides information about the HTTP request, including the method, headers, and the client's IP address.

## Types of Events

Lambda events are categorized into two main types based on how they are invoked:

### Synchronous Events
Synchronous events require an immediate response from the Lambda function. These events are typically triggered by services such as:
- **API Gateway**: Invokes the Lambda function to handle HTTP requests and expects a response within the same connection.
- **Application Load Balancer**: Processes web requests and forwards them to a Lambda function.

### Asynchronous Events
Asynchronous events do not require an immediate response. The event source sends the event to Lambda and continues its operations without waiting for the function to finish execution. Examples include:
- **Amazon S3**: Triggers Lambda when an object is created or deleted.
- **Amazon SNS**: Sends notifications to Lambda functions.

---

## Synchronous Event Flow

### Frontend Load Balancer
![SVG Animation](1_request_receieved_.svg)
1. When a new request (event) arrives, it is first handled by the **Frontend Load Balancer**.
2. The load balancer identifies the appropriate **Frontend Invoke Service** based on the availability zone and routes the event there.

### Frontend Invoke Service

1. The **Frontend Invoke Service** performs initial event processing. it works with **Authentication**, **Authorization** and calls the **Counting Service**

### Counting Service
![SVG Animation](2_counting_service.svg)
It checks things like ***Account Concurrency Limits***, ***Reserved concurrency*** settings on the functions to make sure you are not exceeding any limits. basically it validates if the request exceeds any predefined quotas or limits. and then it calls the ***Assignment Service***

### Assignment Service
![SVG Animation](3_assignment_service.svg)
Its responsible for routing requests to your worker hosts (***Micro Vms***) which runs your codes. and responsible for downloading and executing your codes

If your code has not been loaded to any of the worker nodes yet. or all the worker hosts is busy. then it will create a new execution environment, which is also known as ***Cold Start***

### Placement Service
![SVG Animation](4_placement_serivce_cold_start.svg)
1. The **Placement Service** is invoked when all microVMs are busy or no VM is running. So, need to be initialized.
2. It uses machine learning to:
   - Select the optimal microVM.
   - Prepare a new execution environment (known as a **Cold Start**).
3. Once ready, the microVM is handed back to the **Frontend Invoke Service**.
Now that it transferred to the Frontend Invocation Service, it can run the code while the vm is **worm**
![SVG Animation](4_without_anything.svg)
### Execution in MicroVM

1. The **Assignment Service** triggers the Lambda function within the prepared microVM.
2. After execution, the result is returned to the original requester.
3. The microVM remains **warm** to handle future requests quickly.

---

## Asynchronous Event Flow

### Event Invoke Service (EIFS)

1. For **asynchronous events** (e.g., from S3 or SNS), the **Frontend Load Balancer** sends the event to the **Event Invoke Service (EIFS)**.
2. The EIFS places the event in an **Event Queue**.
3. The Event Queue acknowledges the receipt and immediately responds to the event source.
4. The queued event is later picked up and processed by the Lambda function.
5. EIFS ensures reliability with retries and supports **Dead Letter Queues** for unprocessed events.

### Placement Service

1. The **Placement Service** creates a new microVM environment known as **Cold Start** as we discussed earlier.simplifies application development by handling infrastructure, scaling, and availability. Understanding how these services interact ensures efficient and reliable serverless application design.

2. This involves loading the Lambda function code, initializing dependencies, and setting up the runtime environment.
3. Subsequent requests can reuse this environment, reducing latency.

---

## Example Lambda Function in Go

Below is an example of a Go Lambda function that returns the user's IP address when it receives a GET request.

```go
package main

import (
	"context"
	"encoding/json"
	"github.com/aws/aws-lambda-go/events"
	"github.com/aws/aws-lambda-go/lambda"
)

// Response is the structure for the API Gateway response
type Response struct {
	StatusCode int               `json:"statusCode"`
	Headers    map[string]string `json:"headers"`
	Body       string            `json:"body"`
}

// Handler function for AWS Lambda
func Handler(ctx context.Context, request events.APIGatewayProxyRequest) (Response, error) {
	clientIP := request.RequestContext.Identity.SourceIP
	responseBody, _ := json.Marshal(map[string]string{
		"ip": clientIP,
	})

	return Response{
		StatusCode: 200,
		Headers: map[string]string{
			"Content-Type": "application/json",
		},
		Body: string(responseBody),
	}, nil
}

func main() {
	lambda.Start(Handler)
}
```
