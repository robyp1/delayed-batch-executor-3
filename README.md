
## ARTICLE IN DZONE

https://dzone.com/articles/delayedbatchexecutor-how-to-optimize-database-usag


## LATEST VERSION

https://search.maven.org/artifact/com.github.victormpcmun/delayed-batch-executor

## Rationale behind of DelayeBatchExecutor

There are several scenarios in which concurrent threads execute many times the same query (with different parameters) to a database at almost the same time. 

For example, a REST endpoint serving tens or hundreds requests per second in which each one requires to retrieve a row from table by a different Id.

In a similar way, another typical scenario is a message listener that consumes a large number of messages per second and requires to execute a query by a different Id to process each one.

In these cases, the database executes many times the same query in a short interval of time (say few milliseconds) like these:
```sql
SELECT * FROM TABLE WHERE ID   = <id1>
SELECT * FROM TABLE WHERE ID   = <id2>
...
SELECT * FROM TABLE WHERE ID   = <idn>
```
DelayedBatchExecutor is a component that allows easily to *convert* these n queries of 1 parameter into just one single query with n parameters, like this one:

```sql
SELECT * FROM TABLE WHERE ID IN (<id1>, <id2>, ..., <idn>)
```

The advantages of executing one query with n parameters instead of n queries of 1 parameter are the following:

* The usage of network resources is reduced dramatically: The number of round-trips to the database is 1 instead of n.

* Optimization of database server resources: you would be surprised how well databases optimize queries of n parameters. Pick any large table of your schema and analyse the execution time and CPU usage a for a single query of n parameters versus n queries of 1 parameter.

* The usage of database connections from the connection pool is reduced: there are more available connections overall, which means less waiting time for a connection on peak times.

In short, it is much more efficient executing 1 query of n parameters than n queries of one parameter, which means that the system as a whole requires less resources.

## DelayedBatchExecutor In Action

It basically works by creating *time windows* where the parameters of the queries executed during the *time window* are collected in a list. 
As soon as the *time window* finishes, the list is passed (via callback) to a method that executes one single query with all the parameters in the list and returns another list with the results. Each thread receives their corresponding result from the result list according to one of the following policies as explained below: blocking , non-blocking (Future) and non-blocking (Reactive).

A DelayedBatchExecutor is defined by three parameters:
 
 * TimeWindow: defined as java.time.Duration
 * max size: it is the max number of items to be collected in the list
 * batchCallback: it receives the parameters list to perform a single query and must return a list with the corresponding results. 
    - It can be implemented as method reference or lambda expression.
    - It is invoked automatically as soon as the TimeWindow is finished OR the collection list is full. 
    - The returned list must have a correspondence in elements with the parameters list, this means that the value of position 0 of the returned list must be the one corresponding to parameter in position 0 of the param list and so on...
    - By default, duplicated parameters (by [hashCode](https://docs.oracle.com/javase/8/docs/api/java/lang/Object.html#hashCode--) and [equals](https://docs.oracle.com/javase/8/docs/api/java/lang/Object.html#equals-java.lang.Object-)) are removed from the parameters list automatically. This is optimal in most cases although there is a way for having all parameters (including duplicates) if it is required (See Advanced Usage)
	
  Now, Let's define a DelayedBatchExecutor to receive an Integer value as parameter and return a String, and having a time window = 50 milliseconds, a max size = 100 elements and having the batchCallback defined as method reference: 
  
  ```java
DelayedBatchExecutor2<String,Integer> dbe = DelayedBatchExecutor2.create(Duration.ofMillis(50), 100, this::myBatchCallBack);
  
...
  
List<String> myBatchCallBack(List<Integer> listOfIntegers) {
	List<String>  resultList = ...// execute query:SELECT * FROM TABKE WHERE ID IN (listOfIntegers.get(0), ..., listOfIntegers.get(n));
                                // use your favourite API: JDBC, JPA, Hibernate,...
  	...
  	return resultList;
}
```

The same DelayedBatchExecutor2 but having the callback defined as lambda expression would be:

```java
DelayedBatchExecutor2<String,Integer> dbe = DelayedBatchExecutor2.create(Duration.ofMillis(50), 100, listOfIntegers-> 
{
  List<String>  resultList = ...// execute query:SELECT * FROM TABLE WHERE ID IN (listOfIntegers.get(0), ..., listOfIntegers.get(n));
                                // use your favourite API: JDBC, JPA, Hibernate,...
  ...
  return resultList;
  
 });
  ``` 
NOTE: the instance `dbe` must be accesible from the code being executed by the threads (it is often declared as instance variable of a singleton DAO).
Once defined the DelayedBatchExecutor instance, it is easy to use it from the code executed in each thread

  ```java
// this code is executed in one of the multiple threads
int param=...;
String result = dbe.execute(param); // all the threads executing this code within a interval of 50 ms will have 
			            // their parameters (an integer value) collected in a list (list of integers) 
				    // that will be passed to the callback method, and from the list returned from this
				    // method, each thread will receive its corresponding value
				    // all this managed behind the scenes by the DelayedBatchExecutor
}
```
NOTE:
- To create a DelayedBatchExecutor for taking more than one argument see FootNote 1
- In the example above, the thread is stopped when the execute(...) method is executed until the result is available (blocking behaviour). This is one of the three execution policies of the DelayedBatchExecutor


### Execution Policies

There are three policies to use a DelayedBatchExecutor from the code being executed from the threads

#### Blocking

The thread is blocked until the result is available, it is implemented by using the method `execute(...)`
 
```java 
    int param = ...
	...
    String result = dbe.execute(param); // this thread will be blocked until the result is available
    // compute with result
```
The following diagram depicts how blocking policy works:

![Blocking image](/src/main/javadoc/doc-files/blocking.svg)


#### Non-blocking (java.util.concurrent.Future)

The thread is not blocked, it is implemented by using the method `executeAsFuture(...)`

```java 
    int param = ...
       ...	
    Future<String> resultFuture = dbe.executeAsFuture(param); // the thread will not  be blocked
    // compute something else
    String result = resultFuture.get();  // Blocks the thread until the result is available (if necessary)
    // compute with result
```

The following diagram depicts how Future policy works:

![Future image](/src/main/javadoc/doc-files/future.svg)


#### Non-blocking (Reactive using Reactor framework):
 
 The thread is not blocked, it is implemented by using the method `executeAsMono(...)`
 
```java 
    int param =...
       ...
    reactor.core.publisher.Mono<String> resultMono = dbe.executeAsMono(param); // the thread will not  be blocked
    // compute something else
    resultMono.subscribe(stringResult -> {
         // compute with stringResult
      });
```
The following diagram depicts how Reactive policy works:

![Reactive image](/src/main/javadoc/doc-files/mono.svg)

### Advanced Usage

There are three parameters of a DelayedBatchExecutor that must be known to get the most of it:

- ExecutorService: the callback method is actually executed in a parallel thread, which is provided by an java.util.concurrent.ExecutorService. By default this Executor is `Executors.newFixedThreadPool(4)`.

- bufferQueueSize: it is the max size of the internal list, by default its value is 8192.

- removeDuplicates: if false, then DelayedBatchExecutor won't removed all duplicated parameters from the parameters list before invoking the batchCallback. By default its value is true.

These parameters can be set by using the following constructor:

```java
... 
int maxSize=20;
ExecutorService executorService= Executors.newFixedThreadPool(10);
int bufferQueueSize= 16384;
boolean removeDuplicates = false;
  
DelayedBatchExecutor2<Integer,String> dbe = DelayedBatchExecutor2.create(
    Duration.ofMillis(200), 
    maxSize,
    executorService,
    bufferQueueSize,
    removeDuplicates,
    this::myBatchCallBack);
```
 At any time, the configuration paramaters can be updated by using this thread safe method
 
```java
...
dbe.updateConfig(
    Duration.ofMillis(200), 
    maxSize,
    executorService,
    bufferQueueSize,
    removeDuplicates,
    this::myBatchCallBack);
 ```

-----
-Foot Note 1:  The example shows a DelayedBatchExecutor for a parameter of type Integer and a return type of String, hence DelayedBatchExecutor2<String,Integer>

For a DelayedBatchExecutor for two parameters (say Integer and Date) and a returning type String, the definition would be:
DelayedBatchExecutor3<String,Integer,Date> and so on...
# delayed-batch-executor-3
