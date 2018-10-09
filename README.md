# Circuit Breaker pattern
### Context
In a Microservice architecture, services sometimes collaborate when handling requests. When one service synchronously invokes another there is always the possibility that the other service is unavailable or is exhibiting such high latency it is essentially unusable. Previous resources such as threads might be consumed in the caller while waiting for the other service to respond. This might lead to resource exhaustion, which would make the calling service unable to handle other requests. The failure of one service can potentially cascade to other services throughout the application.

### Problem
How to prevent a network or service failure from cascading to other services?

### Solution
A service client should invoke a remote service via a proxy that functions in a similar fashion to an electrical circuit breaker. When the number of consecutive failures crosses a threshold, the circuit breaker trips, and for the duration of a timeout period all attempts to invoke the remote service will fail immediately. After the timeout expires the circuit breaker allows a limited number of test requests to pass through. If those requests succeed the circuit breaker resumes normal operation. Otherwise, if there is a failure the timeout period begins again.

# Demo
To show the Circuit Breaker pattern in action, the [Hystrix](https://github.com/Netflix/hystrix) implementation of the pattern is used. For the purposes of the demo, the following architecture will be deployed and run on Docker:

<img src="https://github.com/mrusanov/spring-cb-hystrix/blob/master/spring-cb-hystrix-demo.png"/>

where:
 - **spring-cb-server** - a server application which exposes two ednpoints:
   - **GET /server/delay/{delay_in_seconds}** - an endpoint to simulate service latency. The client specifies the delay after which the response will be returned from the server;
   - **GET /server/error/{error_code}** - an endpoint to which the client passes an error code (4xx or 5xx) and the server responses with this code. It is allowed to pass code 200-OK, in order to close the circuit.
 
 - **spring-cb-delay-client** - a client application to the delay endpoint of the server. This application uses the Hystrix implementation of the Circuit Breaker pattern. When a reliable read is made to the server, it is executed as Hystrix command, which means that if the read request fails, the corresponding fallback method will be called. Also, on every reliable read (Hystrix command) a statistic metrics are pushed to the **rabbitmq**, which will be used by the Hystrix dashboard. 
 One endpoint is exposed:
   - **GET /invoke-server/delay/{delay_in_seconds}?reliable={true/false}** - an endpoint to invoke the server's delay endpoint. The `reliable` parameter specifies if the read from the server will be done as Hystrix command or not.
     
 - **spring-cb-error-client** - a client application to the error endpoint of the server, which also uses Hystrix. The following endpoint is exposed:
   - **GET /invoke-server/error/{error_code}?reliable={true/false}** - an endpoint to invoke the server's error endpoint.
   
 - **rabbitmq** - a message queue used by the two clients to push their Hystrix metrics.
 
 - **spring-cb-turbine** - a Turbine application to combine the two Hystrix metrics streams from the two clients into a single Turbine stream. 

 - **spring-cb-dashboard** - a Hystrix Dashboard application to visualize the metrics from the Hystrix commands, pushed by the two clients into the Turbine stream.
 
## Installation: 
1) Clone each of the 'spring-cb-' projects repositories:
   - [spring-cb-server](https://github.com/mrusanov/spring-cb-server)
   - [spring-cb-delay-client](https://github.com/mrusanov/spring-cb-delay-client)
   - [spring-cb-error-client](https://github.com/mrusanov/spring-cb-error-client)
   - [spring-cb-turbine](https://github.com/mrusanov/spring-cb-turbine)
   - [spring-cb-dashboard](https://github.com/mrusanov/spring-cb-dashboard)
 
2) To build a Docker image for every of the projects, run the `buildDockerImage` Gradle task.
   
   ### Environment Variables:
   As explained, the two clients of the server use the Hystrix implementation of the Circuit Breaker pattern. There are some basic Hystrix parameters, which should be configured appropriately. These basic parameters are exposed as Docker environment variables (for each Docker variable, the corresponding Hystrix property is mentioned in brackets):
   - `HYSTRIX_REQUEST_VOLUME_THRESHOLD` (**hystrix.command.default.circuitBreaker.requestVolumeThreshold**) - this property sets the minimum number of requests in a rolling window that will trip the circuit.
      For example, if the value is 20, then if only 19 requests are received in the rolling window (say a window of 10 seconds) the circuit will not trip open even if all 19 failed.
   - `HYSTRIX_SLEEP_WINDOW_MILLIS` (**hystrix.command.default.circuitBreaker.sleepWindowInMilliseconds**) - this property sets the amount of time, after tripping the circuit, to reject requests before allowing attempts again to determine if the circuit should again be closed.
   - `HYSTRIX_ERROR_THRESHOLD_PERCENTAGE` (**hystrix.command.default.circuitBreaker.errorThresholdPercentage**) - this property sets the error percentage at or above which the circuit should trip open and start short-circuiting requests to fallback logic.
   
   *Example:* If **HYSTRIX_REQUEST_VOLUME_THRESHOLD_ARG=20**, **HYSTRIX_SLEEP_WINDOW_MILLIS_ARG=5000** (5 seconds) and **HYSTRIX_ERROR_THRESHOLD_PERCENTAGE_ARG=50**, then if in 5 seconds are received at least 20 requests (volume threshold) and the certain percentage of them will fail (error threshold) circuit will be opened for consecutive 5 seconds. After that time, the first request will be served to a downstream resource, and in case of success, the circuit will be closed again.
   
   - `HYSTRIX_TIMEOUT_ENABLED` (**hystrix.command.default.execution.timeout.enabled**) - this property indicates whether the Hystrix command execution should have a timeout.
   - `HYSTRIX_TIMEOUT_MILLIS` (**execution.isolation.thread.timeoutInMilliseconds**) - this property sets the time in milliseconds after which the caller will observe a timeout and walk away from the command execution. Hystrix marks the HystrixCommand as a TIMEOUT, and performs fallback logic. Note that there is configuration for turning off timeouts per-command, if that is desired.
   
   Information about all Hystrix parameters is available [here](https://github.com/Netflix/Hystrix/wiki/Configuration).
   
   - `CLIENT_SERVER_READ_TIMEOUT_MILLIS` - this property sets the timeout when a client reads from the server. Its value is used by the `RestTemplateBuilder.setReadTimeout(readTimeoutInMilliseconds)`.
  
  ## Running the demo:
  To run the demo, download the **docker-compose.yaml** file and execute:
  
  `$docker-compose up -d`
  
  ## Testing
  If all applications have been started successfully, the server application should be running on port 8090 on the Docker machine, and the endpoints for testing are:
   - _http://{docker-machine-ip}:8091/invoke-server/delay/{delay_in_seconds}?reliable={true/false}_;
   - _http://{docker-machine-ip}:8092/invoke-server/error/{error_code}?reliable={true/false}_;
   
  To monitor the Hystrix metrics, pushed by the two clients, open the Hystrix Dashboard:
   - _http://{docker-machine-ip}:9080/hystrix_
   
  On the home page of the Hystrix dashboard, the Turbine stream should be specified. In the text input for the Turbine stream, paste this:
   - _http://{docker-machine-ip}:8989/hystrix.stream_
   
#Sources
 - [Circuit Breaker pattern](https://microservices.io/patterns/reliability/circuit-breaker.html)
 - [Netflix Hystrix](https://github.com/Netflix/hystrix)
 - [Circuit Breaker Spring Getting Started guide](https://spring.io/guides/gs/circuit-breaker/)
 - [Spring Cloud with Turbine AMQP](http://www.java-allandsundry.com/2016/05/spring-cloud-with-turbine-amqp.html)
