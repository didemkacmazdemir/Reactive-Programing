# Reactive-Programing

Java Reactive Programming

Reactive programming is a declarative programming paradigm / an asynchronous programming style in which we use an event based model to push the data streams to the consumers / observers as and when the data is available / updated. It is completely asynchronous and non-blocking.
In reactive programming, threads are not blocked or waiting for a request to complete. Instead they are notified when the request is complete / the data changes. Till then they can do other tasks. This makes us to use less resources to serve more requests.

Big monolithic applications are split into easily deployable, self-containing microservices. It has been a while since Microservices have become the trend! Microservices do have some disadvantages as well! 

When we have multiple services, One service (A) might send a request to another service (B) and wait for the response from the microservice (B) to proceed further. This is synchronous / blocking the thread from doing the actual work.

One possible solution is to create a thread and send the request via the newly created thread. So that the main thread can do other tasks. Even though it sounds like a good solution, this is what all the old frameworks have been doing all these years. Threads consume memory. If the threads are going to wait for the response, it is wasting of resources

Lets consider we would like to perform the below task.
1. Execute a DB query based on the given input parameters
2. Process the DB query result  (say lowercase to uppercase)
3. Write the processed result into a file.

Solution : Event-Driven Programming:

Pros

we split the above 1 big synchronous task into 3 synchronous simple tasks (ie each step becomes a task). All these tasks are queued. All these tasks are read from a queue one by one and executed using dedicated worker threads from a thread-pool.  When there are no tasks in queue, worker threads would simply wait for the tasks to arrive.
Usually number of worker threads would be small enough to match the number of available processors. This way, You can have 10000 tasks in the queue and process them all efficiently – but we can not create 10000 threads in a single machine.

Cons

So the event-driven programming model is NOT going to increase the performance (up to certain limit) or overall the response time could be still same. But it can process more number of concurrent requests efficiently!

Solution2 : Reactive Programming:

Reactive programming is event-driven programming or special case of event-driven programming.
Lets consider a simple example using Microsoft Excel. Lets take the first three cells A1, B1 and C1 in a spreadsheet. Lets assume that we have the formula for C1 which is = A1 + B1. Now whenever you enter / change the data in A1 or B1 cells, the C1 cell value gets updated immediately. You do not press any button to recalculate. It happens asynchronously.

Reactive programming is based on Observer design pattern. It has following interfaces / components.
* Publisher / Observable : Publisher is observable whom we are interested in listening to! These are the data sources or streams.
* Observer / Subscriber : Observer subscribes to Observable/Publisher. Observer reacts to the data emitted by the Publisher. Publisher pushes the data to the Observers. Publishers are read-only whereas Observers are write-only.
* Subscription : Observer subscribes to Observable/Publisher via an object called Subscription. Publisher’s emitting rate might be higher than Observer’s processing rate. So in that case, Observer might send some feedback to the Publisher how it wants the data / what needs to be done when the publisher’s emitting rate is high via Subscription object.
* Processor / Operators : Processor acts as both Publisher and Observer. They stay in between a Publisher and an Observer.  It consumes the messages from a publisher, manipulates it and sends the processed message to its subscribers. They can be used to chain multiple processors in between a Publisher or an Observer.

NOTE : 
Reactive Stream is a specification which specifies the above 4 interfaces to standardize the programming libraries for Java. Some of the implementations are
* Project Reactor
* RxJava2
* Akka

We would be using Project Reactor in our examples going forward as this is what spring boot uses internally.

	
Publisher Types:

There are 2 types of Observables / Publishers

* Cold Publisher
    * This is lazy
    * Starts producing/emitting only when a subscriber subscribes to this publisher
    * Publisher creates a data producer and generates new sets of values for each new subscription
    * When there are multiple observers, each observer might get different values
    * Example: Netflix. Movie will start streaming only if the subscriber wants to watch. Each subscriber can watch a movie any time from the beginning
* Hot Publisher
    * Values are generated outside the publisher even when there are no observers.
    * There will be only one data producer
    * All the observers get the value from the single data producer irrespective of the time they started subscribing to the publisher. It means any new observer might not see the old value emitted by the publisher.
    * Example: Radio Stream. Listeners will start listening to the song currently playing. It might not be from the beginning.

We have 2 different types of implementations for the Publisher interface.
* Flux
* Mono

For more information : http://www.vinsguru.com/reactive-programming-a-simple-introduction/

The producer/consumer problem - Backpressure
Due to the non-blocking nature of Reactive Programming, the server doesn't send the complete stream at once. It can push the data concurrently as soon as it is available.
Thus, the client waits less time to receive and process the events. 
Let's use an example to clearly describe what it is:
* The system contains three services: the Publisher, the Consumer, and the Graphical User Interface (GUI)
* The Publisher sends 10000 events per second to the Consumer
* The Consumer processes them and sends the result to the GUI
* The GUI displays the results to the users
* The Consumer can only handle 7500 events per second

1-) Request
The first option is to give the consumer control over the events it can process. Thus, the publisher waits until the receiver requests new events. In summary, the client subscribes to the Flux and then process the events based on its demand.
This is a pull strategy to gather elements at the emitter request


With this approach, the emitter never overwhelms the receiver. In other words, the client is under control to process the events it needs.
We'll test the producer behavior with respect to backpressure with StepVerifier. We'll expect the next n items only when the thenRequest(n) is called.


Flux request = Flux.range(1, 50);

    request.subscribe(
      System.out::println,
      err -> err.printStackTrace(),
      () -> System.out.println("All 50 items have been successfully processed!!!"),
      subscription -> {
          for (int i = 0; i < 5; i++) {
              System.out.println("Requesting the next 10 elements!!!");
              subscription.request(10);
          }
      }
    );

    StepVerifier.create(request)
      .expectSubscription()
      .thenRequest(10)
      .expectNext(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
      .thenRequest(10)
      .expectNext(11, 12, 13, 14, 15, 16, 17, 18, 19, 20)
      .thenRequest(10)
      .expectNext(21, 22, 23, 24, 25, 26, 27 , 28, 29 ,30)
      .thenRequest(10)
      .expectNext(31, 32, 33, 34, 35, 36, 37 , 38, 39 ,40)
      .thenRequest(10)
      .expectNext(41, 42, 43, 44, 45, 46, 47 , 48, 49 ,50)
      .verifyComplete();

2-) Limit
Working as a limited push strategy the publisher only can send a maximum amount of items to the client at once

Flux<Integer> limit = Flux.range(1, 25);

    limit.limitRate(10);
    limit.subscribe(
      value -> System.out.println(value),
      err -> err.printStackTrace(),
      () -> System.out.println("Finished!!"),
      subscription -> subscription.request(15)
    );

    StepVerifier.create(limit)
      .expectSubscription()
      .thenRequest(15)
      .expectNext(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
      .expectNext(11, 12, 13, 14, 15)
      .thenRequest(10)
      .expectNext(16, 17, 18, 19, 20, 21, 22, 23, 24, 25)
      .verifyComplete();

3-) Cancel
Finally, the consumer can cancel the events to receive at any moment.
In this case, the receiver can abort the transmission at any given time and subscribe to the stream later again

Flux<Integer> cancel = Flux.range(1, 10).log();

    cancel.subscribe(new BaseSubscriber<Integer>() {
        @Override
        protected void hookOnNext(Integer value) {
            request(3);
            System.out.println(value);
            cancel();
        }
    });

    StepVerifier.create(cancel)
      .expectNext(1, 2, 3)
      .thenCancel()
      .verify();

WebClient

WebClient is an interface representing the main entry point for performing web requests. Which introduced in spring 5.
It was created as part of the Spring Web Reactive module and will be replacing the classic RestTemplate.  And the new client is a reactive, non-blocking solution that works over the HTTP/1.1 protocol.
If we want to use web client we should add spring-boot-starter-webflux dependency which supports for both synchronous and asynchronous operations, making it suitable also for applications running on a Servlet Stack.

spring-boot-starter-webflux includes the below dependencies
* 		spring-webflux framework
* 		reactor-core that we need for reactive streams
* 		reactor-netty (the default server that supports reactive streams). Any other servlet 3.1+ containers like Tomcat, Jetty or non-servlet containers like Undertow can be used as well
* 		spring-boot and spring-boot-starter for basic Spring Boot application setup

Spring Webflux publishers
Webflux provides 2 types of publishers namely
* 		Mono (0…1)
* 		Flux (0…N)
The framework accepts a plain publisher as input and maps it to a reactor type internally, processes it, and returns either a Mono or Flux as the output

Working with WebClient
* create an instance

	public static final int TIMEOUT = 1000;

    final var  httpClient = HttpClient.create()
        .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, TIMEOUT) //set the connection timeout, the default HTTP timeouts of 30 seconds are too slow
        .responseTimeout(Duration.ofMillis(TIMEOUT)) //configure a response timeout
        .doOnConnected(conn -> {
            conn.addHandlerLast(new ReadTimeoutHandler(TIMEOUT, TimeUnit.MILLISECONDS)); //set the read and write timeouts
            conn.addHandlerLast(new WriteTimeoutHandler(TIMEOUT, TimeUnit.MILLISECONDS));
    });

* make a request
	Get And Post Request

Mono<String> monoResponse = webClient.get()
  .uri(uriBuilder - > uriBuilder
    .path("/products/")
    .queryParam("name", "AndroidPhone")
    .queryParam("color", "black")
    .queryParam("deliveryDate", "13/04/2019")
    .build())
  .retrieve()
  .bodyToMono(String.class)
  .block();

Employee createdEmployee = webClient.post()
    .uri("/employees")
    .header(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
    .body(Mono.just(empl), Employee.class)
    .retrieve()
    .bodyToMono(Employee.class);

For PUT, DELETE examples
https://howtodoinjava.com/spring-webflux/webclient-get-post-example/

For Error Handling
https://medium.com/a-developers-odyssey/spring-web-client-exception-handling-cd93cf05b76

* handle the response and retrieve data

Retrieving a Single Resource

Mono<Employee> employeeMono = client.get()
  .uri("/employees/{id}", "1")
  .retrieve()
  .bodyToMono(Employee.class);

employeeMono.subscribe(
    i -> log.info("Received {}", i),
    ex -> log.error("Opps!", ex),
    () -> log.info("Mono completed")
);

Employee employee = employeeMono.block();

Retrieving a Collection Resource

Flux<Employee> employeeFlux = client.get()
  .uri("/employees")
  .retrieve()
  .bodyToFlux(Employee.class);	
        
employeeFlux.subscribe(System.out::println);
List<Employee> list = employeeFlux.collectList().block();

Example of retrieving data 

https://msayag.github.io/WebFlux/
https://www.programcreek.com/java-api-examples/?class=reactor.core.publisher.Mono&method=subscribe



