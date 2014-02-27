# Gatling semantics

## Introduction

Gatling can be used for two use cases:

- Throughput scenarios: Measurements of how many requests per second is a 
server able to deliver.
- Constant load scenarios: Measurements of an exact number of (long running) 
requests per second to examine long term behavior of a remote server.

The following overview gives recipes how to configure Gatling for throughput as 
well as constant load scenarios.

The atomic execution unit is an http request or an arbitrary Java invocation 
using the SessionHookBuilder. The following example declares a single http 
requests towards a remote server:

    val req = http("some id")
    .post("/app/service")
    .body("some body")

The concrete URL is encapsulated in a separate companion object:

    object Protocols {
      val local = http.baseURL("http://localhost:9200")
    }

The supported atomic executions can be seen in the Exec trait. Let's denote one 
execution unit with a simple star:

    +

## Throughput scenarios

Multiple atomic executions can be executed in the context of a scenario. A 
scenario is a series of executions for one user. They can be constructed i.e. 
using the available methods in the Loops trait:

- repeat(n times): executes a unit exactly n times (regardless of the needed 
time)
- during(t seconds): executes a unit exactly for at least t seconds (regardless 
of the amount of requests fitting in this time window)

Examples:

    val timedScn = scenario("1").during(5 seconds) { exec(req) }
    val iteratedScn = scenario("1").repeat(1000) { exec(req) }

The timedScn scenario executes the declared request req during 5 seconds in a 
loop as fast as possible and then stops. The iteratedScn scenario executes the 
declared request exactly 1000 times and then stops. One can visualize timedScn 
scenario using the above denoted lines having a start and end like so:

    |
    | |+++++++|
    |
    +-|-------|-----> time (seconds)
      1       5

Since these scenarios are being executed in an iteration or time bounded loop 
without any delay they will measure the maximum executed requests per second 
and thus are suited for throughput measurements. The measured (successful) 
requests per second are visible as green lines in Gatling.

### Injection

Until now we considered only one user. In a multiuser scenario we have to 
inject multiple parallel users. The amount of concurrent users at a given time 
sample will be visible as yellow lines in Gatling.

### atOnce injection

The atOnce injection works like a 'pulse'. It executes the units in parallel 
with the declared amount of parallel users:

    val atOnceInj = atOnce(3 users)
    setUp(timedScn.inject(atOnceInj)).protocols(Protocols.local)

    User
      |
    3 | |+++++++|
    2 | |+++++++|
    1 | |+++++++|
      |
      +-|-------|-----> time (seconds)
        1       5

Note that starting from second 1 until second 5 we have a constant number of 
concurrently running users. The following ascii graph represents the concurrent 
users vs. time:

    Amount of concurrent users
      |
    3 | * * * * *
    2 |
    1 |
      |
      +-|-------|-----> time (seconds)
        1       5

The following measurement reflects the above facts in an empirical measurement:

![atOnce 3 users over 5 seconds](atOnce3.png)

### constantRate injection

The constantRate injection works like a stretched atOnce injection over time. 
It adds n new users/sec for a given time duration. One can declare a constant 
rate of one user per second for 3 seconds as follows (resulting in 3 concurrent 
users):

    val constInj = constantRate(1 usersPerSec) during (3 seconds)
    setUp(timedScn.inject(constInj)).protocols(Protocols.local)

    User
      |
    3 |     |+++++++|
    2 |   |+++++++|
    1 | |+++++++|
      |
      +-|-------|---|-> time (seconds)
        1 2 3 4 5   7

The following ascii graph represents the concurrent users vs. time:

    Amount of concurrent users
      |
    3 |     * * *
    2 |   *       *
    1 | *           *
      |
      +-|-------|---|-> time (seconds)
        1       5   7

The following measurement reflects the above facts in an empirical measurement:

![constantRate 1 user/sec during 3 seconds](constantRate3.png)

### ramp injection

The ramp injection is equivalent to constantRate injection but allows for a 
different declaration. One can define to ramp 3 users over 3 seconds (resulting 
in the same graph as constantRate) as follows:

    val rampInj = ramp(3 users) over (3 seconds)
    setUp(timedScn.inject(rampInj)).protocols(Protocols.local)

    User
      |
    3 |     |+++++++|
    2 |   |+++++++|
    1 | |+++++++|
      |
      +-|-------|-----> time (seconds)
        1 2 3 4 5

The following measurement reflects the above facts in an empirical measurement:

![ramp 3 users over 3 seconds](ramp3.png)

### rampRate injection

The rampRate allows a progression on the ramp injection. Starting with adding x 
users/sec it adds new users until y new users/sec over a duration. Example: If 
one starts adding 1 users/sec to 3 users/sec then at second 1 the simulation 
executes 1 concurrent user, at second 2 it adds 2 users, at second 3 it adds 3 
users resulting in a peak of 1+2+3=6 users.

    val rampRateInj = rampRate(1 usersPerSec) to (3 usersPerSec) during (3 
seconds)
    setUp(timedScn.inject(rampRateInj)).protocols(Protocols.local)

    User
      |
    6 |     |+++++++|
    5 |     |+++++++|
    4 |     |+++++++|
      |
    3 |   |+++++++|
    2 |   |+++++++|
      |
    1 | |+++++++|
      |
      +-|-------|---|-> time (seconds)
        1 2 3 4 5   7


The following ascii graph represents the concurrent users vs. time:

    Amount of concurrent users
    6 |     * * *
    5 |           *
    4 |
    3 |   *         *
    2 |
    1 | *
      |
      +-|-------|---|-> time (seconds)
        1       5   7

The following measurement reflects the above facts in an empirical measurement:

![ramp rate starting from 1 users/sec to 3 users/sec during 3 
seconds](rampRate3.png)

## Constant load scenarios

The above scenarios reflect measurements with requests which are supposed to 
take longer that one second or take a considerable amount of iterations. If and 
only single execution units (requests) take less than a second then the gatling 
DSL can be used to simulate a constant load on the server. This can be achieved 
by leaving out the methods from the Loop trait:

    val onceScn = scenario("once").exec(req)

Using atOnce doesn't make much sense in these scenarios because here they 
really behave like pulses. But if executing single requests (without iteration 
or duration and which take less than a second) a constantRate injection 
produces a constant load on the server. Each second x users are injected 
resulting always in x requests/second:

    val onceScn = scenario("once").exec(req)
    val constInj = constantRate(3 usersPerSec) during (5 seconds)
    setUp(onceScn.inject(constInj)).protocols(Protocols.local)

    User
      |
    3 | + + + + +
    2 | + + + + +
    1 | + + + + +
      |
      +-|-------|-----> time (seconds)
        1 2 3 4 5

The following ascii graph represents the concurrent users vs. time:

    Amount of concurrent users
      |
    3 | * * * * *
    2 | 
    1 |
      |
      +-|-------|-----> time (seconds)
        1       5

By using this technique we would generate a load of 3 req/sec which is 
reflected in the following empirical measurements:

![](constantRate3_constant.png)

Similarly the ramp injection can be used to generate the same sort of constant 
load simulation. In this case one has to ramp 15 users over 5 seconds to 
generate a constant load of 3 requests/sec, which is reflected in the following 
measurements:

    val onceScn = scenario("once").exec(req)
    val rampInj = ramp(15 users) over (5 seconds)
    setUp(onceScn.inject(rampInj)).protocols(Protocols.local)

![](ramp3_constant.png)

One can use the rampRate injection to ramp up the load, i.e. starting from 1 
user/sec up to 3 users/sec over 3 seconds. Afterwards you can chain an 
additional constantRate injection to create a constant load of 3 users/sec:

    val onceScn = scenario("once").exec(req)
    val rampRateInj = rampRate(1 usersPerSec) to (3 usersPerSec) during (3 
seconds)
    val constInj = constantRate(3 usersPerSec) during (5 seconds)
    setUp(onceScn.inject(rampRateInj, constInj)).protocols(Protocols.local)

    User
      |
    3 |     + + + + + +
    2 |   + + + + + + +
    1 | + + + + + + + +
      |
      +-|-------|-----> time (seconds)
        1 2 3 4 5

The following ascii graph represents the concurrent users vs. time:

    Amount of concurrent users
      |
    3 |     * * * * * *
    2 |   *
    1 | *
      |
      +-|-------|-----> time (seconds)
        1       5

![](rampRate3_constant.png)

## Annex

The following source code was used to generate the above simulations:

    class TestSimulation extends Simulation {
      val req = http("_search")
        .post("/attributes/attribute/_search")
        .body(StringBody(MedsSimulations.randomQueryLowPExpression))

      val timedScn = scenario("timed").during(5 seconds) { exec(req) }
      val iteratedScn = scenario("iterated").repeat(1000) { exec(req) }
      val onceScn = scenario("once").exec(req)

      val rampInj = ramp(15 users) over (5 seconds)
      val atOnceInj = atOnce(1 users)
      val rampRateInj = rampRate(1 usersPerSec) to (3 usersPerSec) during (3 seconds)
      val constInj = constantRate(3 usersPerSec) during (5 seconds)

      setUp(onceScn.inject(constInj)).protocols(Protocols.local)
    }

    object Protocols {
      val local = http.baseURL("http://localhost:9200")
    }
