# spring-chaos

![image](https://github.com/user-attachments/assets/1caecaf0-fcf3-4f4f-839b-585b1d28d38e)

Process
Integrating and executing experiments with Chaos Monkey for Spring Boot-Java applications is a 3 step process as described below:

![image](https://github.com/user-attachments/assets/c7439f96-3370-4de0-a3ae-061999680a8b)

* Add chaos monkey to the spring boot application
* Create the experiments
* Execute the experiments

1. Add Chaos Monkey to SpringBoot application

   
   #pom dependency

    <dependency>
    <groupId>de.codecentric</groupId>
        <artifactId>chaos-monkey-spring-boot</artifactId>
        <version>{version}</version>
    </dependency>

    #application.properties

      spring.profiles.active=chaos-monkey
      chaos.monkey.enabled=true
      management.endpoint.chaosmonkey.enabled=true
      management.endpoints.web.exposure.include=health,info,chaosmonkey
      management.endpoints.web.base-path=/actuator

3. Create Chaos experiments
   
    Spring Boot Chaos Monkey uses reflections to inject failures in the application. 
    These experiments are defined with the conventions of Watchers and Assaults.

![image](https://github.com/user-attachments/assets/ea70ed05-480f-4cbf-91b2-7473eb5e8776)

Watchers

A Watcher is a module that injects the failure in the Application modules decorated with specific Annotations. Version used in this post supports the following stereotypes.

@Controller
@RestController
@Service
@Repository
@Component

Assaults

Assault is the attack that is injected in the application. The following Assaults are supported in the version used in this post.

* Latency Assault
* Exception Assault
* AppKiller Assault
* Memory Assault
* CPU Assault

Experiment 1 — Experiment to simulate database failure

The following experiment file defines an experiment to test application when a failure is injected at database level for every request. The application tries to connect to the database and get a connection error during test.

---------------------------------------------------------------------------------------
{
  "version": "1.0.0",
  "title": "Order Service when database is down",
  "description": "N/A",
  "tags": [],
  "steady-state-hypothesis": {
    "title": "Order database is available",
    "probes": [
      {
        "type": "probe",
        "name": "we-can-retrieve-all-orders-from-DB",
        "tolerance": 200,
        "provider": {
          "type": "http",
          "timeout": 1,
          "url": "http://localhost:8080/order-svc/all-orders"
        }
      }
    ]
  },
  "method": [
    {
      "name": "enable_chaosmonkey",
      "provider": {
        "arguments": {
          "base_url": "http://localhost:8080/actuator"
        },
        "func": "enable_chaosmonkey",
        "module": "chaosspring.actions",
        "type": "python"
      },
      "type": "action"
    },
    {
      "name": "configure_assaults",
      "provider": {
        "arguments": {
          "base_url": "http://localhost:8080/actuator",
          "assaults_configuration": {
            "level": 1,
            "latencyActive": false,
            "exceptionsActive": true,
            "exception": {
              "type": "java.net.ConnectException",
              "arguments": [
                {
                  "className": "java.lang.String",
                  "value": "Connection timed out"
                }
              ]
            }
          }
        },
        "func": "change_assaults_configuration",
        "module": "chaosspring.actions",
        "type": "python"
      },
      "type": "action"
    },
    {
      "name": "configure_repository_watcher",
      "tolerance": 200,
      "provider": {
        "type": "http",
        "url": "http://localhost:8080/actuator/chaosmonkey/watchers",
        "method": "POST",
        "headers": {
          "Content-Type": "application/json"
        },
        "arguments": {
          "controller": false,
          "restController": false,
          "service": false,
          "repository": true,
          "component": false
        }
      },
      "type": "action"
    }
  ],
  "rollbacks": [
    {
      "name": "disable_chaosmonkey",
      "provider": {
        "arguments": {
          "base_url": "http://localhost:8080/actuator"
        },
        "func": "disable_chaosmonkey",
        "module": "chaosspring.actions",
        "type": "python"
      },
      "type": "action"
    }
  ]
}
---------------------------------------------------------------------------------------
The steady state hypothesis means the experiment will be considered successful on receiving HTTP(probes[0].provider.type) HTTP 200 response (probes[0].tolerance) from http://localhost:8080/order-svc/all-orders (probes[0].provider.url) in 1 second (probes[0].provider.timeout).

The method configure_repository_watcher sets the repository assaults in the configure_repository_watcher.arguments.repository:true

The method configure_assaults sets an exception assault using exceptionsAssault:true.

The method configure_assaults injects the failure using level:1. If you need to inject the failure on random request the please choose any number from 2–5. Level > 1 would require one to fire the tests from a load test tool. Example: level:2 means inject failure every alternative request, or level 5 will inject failure every fifth request to the test.

The rollbacks defines the rollbacks that can be applied after the test is successful. The rollback section in this test disable the chaos monkey from the application. As per the declaration above, the rollback section will not execute after the failure happens, so the chaos monkey will continue to inject failures on every successive request in the application.

Pre-Test set-up

* Order-service is running on port 8080 http://localhost:8080/actuator/health should return {“status”:”UP”}
* Billing-service is running on port 8081 http://localhost:8081/actuator/health should return {“status”:”UP”}
* Verify the H2 in-memory database, sql and data is working http://localhost:8080/h2-console/
* Create the virtual environment to run the experiment from toolkit.
---------------------------------------------------------------------------------------

chaos run <<path of the experiment file>>

$ chaos run {root-path}/experiment-database-failure-injector.json

The test will fail the steady state hypothesis with the following log

***************************************************************************************
[2021–09–13 17:28:11 INFO] Validating the experiment’s syntax
[2021–09–13 17:28:11 INFO] Experiment looks valid
[2021–09–13 17:28:11 INFO] Running experiment: Order Service when database is down
[2021–09–13 17:28:11 INFO] Steady-state strategy: default
[2021–09–13 17:28:11 INFO] Rollbacks strategy: default
[2021–09–13 17:28:11 INFO] Steady state hypothesis: Order database is available
[2021–09–13 17:28:11 INFO] Probe: we-can-retrieve-all-orders-from-DB
[2021–09–13 17:28:11 INFO] Steady state hypothesis is met!
[2021–09–13 17:28:11 INFO] Playing your experiment’s method now…
[2021–09–13 17:28:11 INFO] Action: enable_chaosmonkey
[2021–09–13 17:28:11 INFO] Action: configure_assaults
[2021–09–13 17:28:11 INFO] Action: configure_repository_watcher
[2021–09–13 17:28:11 INFO] Steady state hypothesis: Order database is available
[2021–09–13 17:28:11 INFO] Probe: we-can-retrieve-all-orders-from-DB
[2021–09–13 17:28:11 CRITICAL] Steady state probe ‘we-can-retrieve-all-orders-from-DB’ is not in the given tolerance so failing this experiment
[2021–09–13 17:28:11 INFO] Experiment ended with status: deviated
[2021–09–13 17:28:11 INFO] The steady-state has deviated, a weakness may have been discovered

**************************************************************************************

This experiment have discovered a possible weakness within the application.

Fix Issue:

There are multiple ways to fix this issue in the application code. Uncomment the try catch block at OrderComponent.getAllOrders().

After deploying the fix, the same experiment will successfully pass with the following console output.

***************************************************************************************
[2021–09–13 17:44:20 INFO] Validating the experiment’s syntax
[2021–09–13 17:44:20 INFO] Experiment looks valid
[2021–09–13 17:44:20 INFO] Running experiment: Order Service when database is down
[2021–09–13 17:44:20 INFO] Steady-state strategy: default
[2021–09–13 17:44:20 INFO] Rollbacks strategy: default
[2021–09–13 17:44:20 INFO] Steady state hypothesis: Order database is available
[2021–09–13 17:44:20 INFO] Probe: we-can-retrieve-all-orders-from-DB
[2021–09–13 17:44:20 INFO] Steady state hypothesis is met!
[2021–09–13 17:44:20 INFO] Playing your experiment’s method now…[2021–09–13 17:44:20 INFO] Action: enable_chaosmonkey
[2021–09–13 17:44:20 INFO] Action: configure_assaults
[2021–09–13 17:44:20 INFO] Action: configure_repository_watcher
[2021–09–13 17:44:20 INFO] Steady state hypothesis: Order database is available
[2021–09–13 17:44:20 INFO] Probe: we-can-retrieve-all-orders-from-DB
[2021–09–13 17:44:20 INFO] Steady state hypothesis is met!
[2021–09–13 17:44:20 INFO] Let’s rollback…
[2021–09–13 17:44:20 INFO] Rollback: disable_chaosmonkey
[2021–09–13 17:44:20 INFO] Action: disable_chaosmonkey
[2021–09–13 17:44:20 INFO] Experiment ended with status: completed
***************************************************************************************
Verification from application logs

It can still be verified from the application or console logs that the failure was injected and the application handled it in the graceful manner.
***************************************************************************************
...
2021-09-14 17:10:36.034 ERROR 8221 --- [nio-8080-exec-9] xxx.xx.xx.ordersvc.order.OrderComponent  : Exception when fetching DB records {}
...  
xxx.xxx.xxx.ordersvc.order.OrderComponent.getAllOrders(OrderComponent.java:30) ~[classes/:na]
...
***************************************************************************************
***************************************************************************************

Experiment 2— Experiment to inject service failures

The following experiment file defines an experiment to test application when a failure is injected at service connection for every request. The application tries to connect to a downstream serve and get a connection error
---------------------------------------------------------------------------------------
{
  "version": "1.0.0",
  "title": "Order Service when billing service is required",
  "description": "N/A",
  "tags": [],
  "steady-state-hypothesis": {
    "title": "Billing service is available",
    "probes": [
      {
        "type": "probe",
        "name": "we-can-process-order-with-billing-service",
        "tolerance": 200,
        "provider": {
          "type": "http",
          "timeout": 1,
          "url": "http://localhost:8080/order-svc/order-billing-status/1"
        }
      }
    ]
  },
  "method": [
    {
      "name": "enable_chaosmonkey",
      "provider": {
        "arguments": {
          "base_url": "http://localhost:8080/actuator"
        },
        "func": "enable_chaosmonkey",
        "module": "chaosspring.actions",
        "type": "python"
      },
      "type": "action"
    },
    {
      "name": "configure_assaults",
      "provider": {
        "arguments": {
          "base_url": "http://localhost:8080/actuator",
          "assaults_configuration": {
            "level": 1,
            "latencyActive": false,
            "exceptionsActive": true,
            "exception": {
              "type": "java.lang.RuntimeException",
              "arguments": [
                {
                  "className": "java.lang.String",
                  "value": "just an exception"
                }
              ]
            }
          }
        },
        "func": "change_assaults_configuration",
        "module": "chaosspring.actions",
        "type": "python"
      },
      "type": "action"
    },
    {
      "name": "configure_service_watcher",
      "tolerance": 200,
      "provider": {
        "type": "http",
        "url": "http://localhost:8080/actuator/chaosmonkey/watchers",
        "method": "POST",
        "headers": {
          "Content-Type": "application/json"
        },
        "arguments": {
          "controller": false,
          "restController": false,
          "service": true,
          "repository": false,
          "component": false
        }
      },
      "type": "action"
    }
  ],
  "rollbacks": [
    {
      "name": "disable_chaosmonkey",
      "provider": {
        "arguments": {
          "base_url": "http://localhost:8080/actuator"
        },
        "func": "disable_chaosmonkey",
        "module": "chaosspring.actions",
        "type": "python"
      },
      "type": "action"
    }
  ]
}
--------------------------------------------------------------------------------------
configure_argument_watcher is configured to add assault to the service

configure_assaults injects a Runtime Exception

steady_state_hypothesis will be fulfilled when the test receives HTTP 200 response under 1 seconds from http://localhost:8080/process-billing-for-order
---------------------------------------------------------------------------------------
Running Experiment 2 — Service failure experiment

Ensure the test set up is correct as per instruction

Check if order service API is up and running: http://localhost:8080/order-svc/order-billing-status/1

Case 1: When billing-service is NOT up and running then #2, should return HTTP 500 error response

Case 2: When billing-service is up and running, then #2, should return HTTP 200 response with body { “id”: 1, “status”: “ON_HOLD”, “billingStatus”: true }

The success criteria of this test depends on Case 2.

3. Execute the chaos run command from the chaos toolkit virtual environment to run experiment from experiment-service-failure-injector.json

$ chaos run {root-path}/experiment-service-failure-injector.json

This test exposes a weakness in the system.
---------------------------------------------------------------------------------------
[2021-09-14 13:33:05 INFO] Validating the experiment's syntax
[2021-09-14 13:33:05 INFO] Experiment looks valid
[2021-09-14 13:33:05 INFO] Running experiment: Order Service when billing service is required
[2021-09-14 13:33:05 INFO] Steady-state strategy: default
[2021-09-14 13:33:05 INFO] Rollbacks strategy: default
[2021-09-14 13:33:05 INFO] Steady state hypothesis: Billing service is available
[2021-09-14 13:33:05 INFO] Probe: we-can-process-order-with-billing-service
[2021-09-14 13:33:05 INFO] Steady state hypothesis is met!
[2021-09-14 13:33:05 INFO] Playing your experiment's method now...
[2021-09-14 13:33:05 INFO] Action: enable_chaosmonkey
[2021-09-14 13:33:05 INFO] Action: configure_assaults
[2021-09-14 13:33:05 INFO] Action: configure_service_watcher
[2021-09-14 13:33:05 INFO] Steady state hypothesis: Billing service is available
[2021-09-14 13:33:05 INFO] Probe: we-can-process-order-with-billing-service
[2021-09-14 13:33:05 CRITICAL] Steady state probe 'we-can-process-order-with-billing-service' is not in the given tolerance so failing this experiment
[2021-09-14 13:33:05 INFO] Experiment ended with status: deviated[2021-09-14 13:33:05 INFO] The steady-state has deviated, a weakness may have been discovered
---------------------------------------------------------------------------------------
Fix Issue:

There are multiple ways to fix this issue in the application code. For demo, I will uncomment the try catch block at OrderComponent.getOrderBillingStatus(int id);

After deploying the fix, the same experiment will successfully pass with the following console output.

---------------------------------------------------------------------------------------
[2021-09-14 17:04:33 INFO] Validating the experiment's syntax
[2021-09-14 17:04:33 INFO] Experiment looks valid
[2021-09-14 17:04:33 INFO] Running experiment: Order Service when billing service is required
[2021-09-14 17:04:33 INFO] Steady-state strategy: default
[2021-09-14 17:04:33 INFO] Rollbacks strategy: default
[2021-09-14 17:04:33 INFO] Steady state hypothesis: Billing service is available
[2021-09-14 17:04:33 INFO] Probe: we-can-process-order-with-billing-service
[2021-09-14 17:04:33 INFO] Steady state hypothesis is met!
[2021-09-14 17:04:33 INFO] Playing your experiment's method now...[2021-09-14 17:04:33 INFO] Action: enable_chaosmonkey
[2021-09-14 17:04:33 INFO] Action: configure_assaults
[2021-09-14 17:04:33 INFO] Action: configure_service_watcher
[2021-09-14 17:04:33 INFO] Steady state hypothesis: Billing service is available
[2021-09-14 17:04:33 INFO] Probe: we-can-process-order-with-billing-service
[2021-09-14 17:04:33 INFO] Steady state hypothesis is met!
[2021-09-14 17:04:33 INFO] Let's rollback...
[2021-09-14 17:04:33 INFO] Rollback: disable_chaosmonkey
[2021-09-14 17:04:33 INFO] Action: disable_chaosmonkey
[2021-09-14 17:04:33 INFO] Experiment ended with status: completed
---------------------------------------------------------------------------------------
Verification from application logs

It can be verified from the application or console logs that the failure was injected at desired location, and the application handled it in the graceful manner.
***************************************************************************************
....
 at com.xxx.xxx.ordersvc.order.OrderComponent$$EnhancerBySpringCGLIB$$e2c96859.getOrderBillingStatus(<generated>) ~[classes/:na]
.... 
at xxx.xxx.xxx.ordersvc.order.OrderController.getOrderWithBillingStatus(OrderController.java:44) ~[classes/:na]
 at com.sdb.poc.ordersvc.order.OrderController$$FastClassBySpringCGLIB$$237d5b02.invoke(<generated>) ~[classes/:na]
***************************************************************************************
