---
title: "HTTP done three ways with Spring Cloud Kubernetes on OpenShift"
date: 2017-05-02
---


A brief look at three ways to achieve synchronous HTTP based communication between microservices in an OpenShift environment using Spring Cloud projects.

Despite the fact that I now believe messaging based asynchronous communication between bounded contexts (in the context of DDD) is a better way for systems to communicate, there are times when a quick RESTful call is the pragmatic choice. This post takes a brief look at three ways to achieve this when deploying to an OpenShift instance.

> For the unacquainted, [OpenShift](https://www.openshift.com/) is a distribution of Kubernetes which adds "enterprise" and other developer centric features and workflows. The examples in this post are based on [OpenShift Origin](https://www.openshift.org/), the upstream community version.

## Muster the cluster

Before we dive in though, let's standup a local OpenShift cluster to deploy to. I'll be using [minishift](https://docs.openshift.org/latest/minishift/index.html) to help with this. Install the latest release by following the documentation here: https://docs.openshift.org/latest/minishift/getting-started/installing.html (I'm using the `1.0.0-rc.2` version at time of writing)

Once installed, spin up an instance with

```
$ minishift addons install --defaults
Default add-ons anyuid, admin-user installed
$ minishift addons enable admin-user
Addon 'admin-user' enabled
$ minishift start \
  --openshift-version v1.5.0
  --cpus 4 # adjust accordingly
Starting local OpenShift cluster using 'xhyve' hypervisor...
Downloading ISO 'https://github.com/minishift/minishift-b2d-iso/releases/download/v1.0.2/minishift-b2d.iso'
 40.00 MB / 40.00 MB [=...=] 100.00% 0s
Downloading OpenShift binary 'oc' version 'v1.5.0'
 18.93 MB / 18.93 MB [=...=] 100.00% 0s
-- Checking OpenShift client ... OK
-- Checking Docker client ... OK
-- Checking Docker version ... OK
-- Checking for existing OpenShift container ... OK
-- Checking for openshift/origin:v1.5.0 image ...
   Pulling image openshift/origin:v1.5.0
   Pulled 0/3 layers, 3% complete
...
   Pulled 3/3 layers, 100% complete
   Extracting
   Image pull complete
-- Checking Docker daemon configuration ... OK
-- Checking for available ports ... OK
-- Checking type of volume mount ...
   Using Docker shared volumes for OpenShift volumes
-- Creating host directories ... OK
-- Finding server IP ...
   Using 192.168.64.12 as the server IP
-- Starting OpenShift container ...
   Creating initial OpenShift configuration
   Starting OpenShift using container 'origin'
   Waiting for API server to start listening
   OpenShift server started
-- Adding default OAuthClient redirect URIs ... OK
-- Installing registry ... OK
-- Installing router ... OK
-- Importing image streams ... OK
-- Importing templates ... OK
-- Login to server ... OK
-- Creating initial project "myproject" ... OK
-- Removing temporary directory ... OK
-- Checking container networking ... OK
-- Server Information ...
   OpenShift server started.
   The server is accessible via web console at:
       https://192.168.64.12:8443

   You are logged in as:
       User:     developer
       Password: developer

   To login as administrator:
       oc login -u system:admin

Applying addon admin-user:.user "admin" created
.cluster role "cluster-admin" added: "admin"
```

Once OpenShift is running, make sure to login with the `oc` CLI tool, as this will shortly be important when using the Fabric8 Maven tooling to deploy our applications.

```
$ oc login -u system:admin
Logged into "https://192.168.64.12:8443" as "system:admin" using existing credentials.

You have access to the following projects and can switch between them with 'oc project <projectname>':

    default
    kube-system
  * myproject
    openshift
    openshift-infra

Using project "myproject".
```

you should also access the OpenShift console at the location indicated after running `minishift start`, which in the example above is https://192.168.64.12:8443. You can login with `admin` / `admin`.

## The echo example... example.. example

The example consists of two microservices:

* `echo` - which accepts a `POST /echo` and using the request body will call
* `chamber` - which simply echoes the request to `/chamber`

The complete source code for this example is available on GitHub here:

https://github.com/donovanmuller/echo-example

## Vanilla RestTemplate

Let's start off with the most basic example, using plain old `RestTemplate`. Note how we reference the `chamber` service in `EchoController`:

```
@PostMapping
String echo(@RequestBody String message) {
  log.info("Sending echo message: {}", message);

  ResponseEntity<String> response = restTemplate
      .postForEntity("http://chamber:8080/chamber", message, String.class);

  return response.getBody();
}
```

The `url` parameter is specified as `http://chamber:8080/chamber`. So how does this work? How and to what does `chamber` resolve?

As you will see, the `chamber` reference is to a OpenShift/Kubernetes [Service](https://docs.openshift.org/latest/architecture/core_concepts/pods_and_services.html#services) which, through the [OpenShift DNS](https://docs.openshift.org/latest/architecture/additional_concepts/networking.html#architecture-additional-concepts-openshift-dns) allows a Pod IP to be queried using the Service name.

### Deploying to OpenShift with the Fabric8 Maven tooling

To better visualise the above and to actually test our vanilla `RestTemplate` based services, we need to deploy them to our local OpenShift instance.

Thankfully, this has been made almost too easy by the awesome tooling provided by the [Fabric8](https://fabric8.io/) team. All it takes is to include the `fabric8-maven-plugin` into our projects and profit. It really is that easy. Here is the complete `pom.xml` from the `echo` service

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.switchbit</groupId>
        <artifactId>echo-parent</artifactId>
        <version>1.0-SNAPSHOT</version>
        <relativePath>..</relativePath>
    </parent>

    <groupId>io.switchbit</groupId>
    <artifactId>echo</artifactId>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.fabric8</groupId>
                <artifactId>fabric8-maven-plugin</artifactId>
                <executions>
                    <execution>
                        <id>fmp</id>
                        <goals>
                            <goal>resource</goal>
                            <goal>build</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

the `build` section is where the magic happens.
Urm, ok. How...? 

* How does it know where our OpenShift instance is running?
* How does it authenticate? 
* OpenShift/Kubernetes schedule Docker containers right, so where does our service get layered into a Docker image and pushed to the OpenShift internal [registry](https://docs.openshift.org/latest/install_config/registry/index.html)?
* OpenShift/Kubernetes needs various Objects that declaratively tell it what and how to run our containers, where does that come from?

[RTFM](https://maven.fabric8.io). However, in a nutshell and using the OpenShift options (standard Kubernetes is also supported)

* It deploys to the OpenShift instance to which you are currently logged into with the `oc` CLI tool (hence the nudge to login earlier...)
* the plugin builds Docker images using the [S2I](https://maven.fabric8.io/#build-openshift) build mechanism. Choosing which base image to use with an abstraction called [Generators](https://maven.fabric8.io/#generators). In our case, the [Spring Boot generator](https://maven.fabric8.io/#generator-spring-boot) is activated based on the dependencies in our project and a Boot optimised build is selected.
* Another abstraction called [Enrichers](https://maven.fabric8.io/#enrichers) are used to optimise the OpenShift resource objects for a Spring Boot application. For instance the `f8-spring-boot-health-check` enricher configures the proper health and readiness probes based on `spring-boot-starter-actuator` settings of your project.

So now that we kinda know how it works, let's see it in action. So we can see what's going on, open the [`My Project`](https://192.168.64.12:8443/console/project/myproject/overview) project in the OpenShift console and kick off the following Maven goal:

```
$ mvn fabric8:deploy
```

First time this runs it might take a while to pull the base S2I images but after a short while you'll see two Builds started and once those have completed, you'll see two Deployments fired up.

Once they're a nice dark blue colour (dark blue indicates the health checks passed) then you can `curl` an echo:

```
$ curl -w '\n' \
    -D POST \
    -H 'Content-Type: text/plain' \
    --data 'Hello' \
    http://echo-myproject.192.168.64.12.nip.io/echo
HELLO... Hello.. hello
```
where `http://echo-myproject.192.168.64.12.nip.io` is the [Route](https://docs.openshift.org/latest/architecture/core_concepts/routes.html) host assigned to your `echo` Service.

That was a good start but since we've broken the [first rule](https://twitter.com/mariofusco/status/612595466593669121) of distributed systems, what happens when the echo `chamber` service is down? Try this by scaling down the `chamber` Pod to 0 pods and repeating the `curl`...

<iframe width="100%" height="730" src="https://www.youtube.com/embed/Zc93FC1HduY" frameborder="0" allowfullscreen></iframe>

😱 504, hmm. This isn't amateur hour, so let's introduce some production ready features.

## Hystrix

Probably one of the biggest issues with synchronous communication patterns, as experienced above when the `chamber` service was down, is that the upstream services are tightly coupled to the health and welfare of the downstream services. If one if them is having a bad day, so are you.

While we cannot get away from this fact, we can make the life of an upstream service slightly more bearable for the end users. To allow for this, we can implement the following patterns:

* *Bulkheading* - so that we can still service traffic reliably by isolating the parts of our application that are affected by a slow or misbehaving collaborator
* *Circuit breaker* - If something's broken, don't call it. If a downstream service is down or throwing errors consistently, then there is not much point in trying to call it repeatedly in quick succession. Rather we'll fail fast and check back every now and then to see if it's recovered.
* *Fallback* - If the downstream service is sinking and we fail fast, let's give our end users something a bit better than a stacktrace.

Fortunately we can lean on the [Spring Cloud Netflix](https://cloud.spring.io/spring-cloud-netflix/) project to provide much of this functionality using Ribbon and Hystrix.

> Switch to the `hystrix` branch of the source repository to see the changes needed to enable Hystrix.
>
> https://github.com/donovanmuller/echo-example/tree/hystrix

First let's bring in two additional dependencies

```
...

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-ribbon</artifactId>
        </dependency>

        <dependency>
            <groupId>io.fabric8</groupId>
            <artifactId>spring-cloud-starter-kubernetes-netflix</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-hystrix</artifactId>
        </dependency>
    </dependencies>

...
```

`spring-cloud-starter-hystrix` is self explanatory and brings in all we need to make use of the Hystrix functionality.

### Spring Cloud Kubernetes

The more interesting dependency is `spring-cloud-starter-kubernetes-netflix`. The Spring Cloud Kubernetes project also [originated](https://github.com/fabric8io/spring-cloud-kubernetes) (the project has now moved to the [Spring Cloud Incubator](https://github.com/spring-cloud-incubator/spring-cloud-kubernetes)) from the Fabric8 team and provides a way for you to use some of the usual Spring Cloud abstractions, like `DiscoveryClient`, in an OpenShift/Kubernetes environment using the native service discovery properties present, without the need for Eureka or Consul. This is great because if you are migrating applications to OpenShift/Kubernetes and are already using the Spring Cloud service discovery abstractions, you only have to swop out the Eureka/Consul starters for the relavent Spring Cloud Kubernetes starter.

Let's change our `EchoController` to use the discovered Service name (`http://chamber/chamber`) and dispense with the port, as `sc-kubernetes` will default it to the first port exposed via the Service.

```
@PostMapping
@HystrixCommand(fallbackMethod = "noOneHome")
String echo(@RequestBody String message) {
  log.info("Sending echo message: {}", message);

  ResponseEntity<String> response = restTemplate
      .postForEntity("http://chamber/chamber", message, String.class);

  return response.getBody();
}

String noOneHome(String message) {
  log.warn("Hmm, looks like no one's home for echoing message [{}] :(", message);

  return message;
}
```

Note that we've also included a fallback method `noOneHome` that will return the message straight back to the caller if the `chamber` service is down or returning errors.

We also need to add `@EnableCircuitBreaker` to our configuration class as well as indicate that our `restTemplate` bean should be enhanced with Ribbon:

```
@Configuration
@EnableCircuitBreaker
public class EchoConfiguration {

	@Bean
	@LoadBalanced
	RestTemplate restTemplate() {
		return new RestTemplate();
	}
}
```

Let's test this by deploying the upgraded echo service:

> At this point you must give the default [Service Account](https://docs.openshift.org/latest/dev_guide/service_accounts.html) the `cluster-reader` role so that `sc-kubernetes` can integrate the OpenShift API. Do this with the following `oc` command: `oc policy add-role-to-user cluster-reader system:serviceaccount:myproject:default`.


```
$ mvn clean fabric8:deploy
```

wait for the deployments to finish rolling and with the `chamber` service with one replica, `curl` another test:

```
$ curl -w '\n' \
    -D POST \
    -H 'Content-Type: text/plain' \
    --data 'Hello' \
    http://echo-myproject.192.168.64.12.nip.io/echo
HELLO... Hello.. hello
```

still working. Now scale down the `chamber` Pod again and retest:

```
$ curl -w '\n' \
    -D POST \
    -H 'Content-Type: text/plain' \
    --data 'Hello' \
    http://echo-myproject.192.168.64.12.nip.io/echo
Hello
```

We got a quick response with our fallback value.
Much better than the vanilla version! 👍

## Feign

Just for completeness, let's look at a third example using a [Feign](http://cloud.spring.io/spring-cloud-netflix/spring-cloud-netflix.html#spring-cloud-feign) client.

Feign allows you to write interfaces annotated with Spring MVC annotations which are then used to build the actual client request. I.e. you call methods on an injected interface which are proxies to actual HTTP requests. For example, this allows you to author an API client library and distribute to your consumers, freeing them of the burden of having to write their own bindings.

This approach sounds good but can further couple your services together and additionally add a healthy dose of distributed dependency management hell to the mix. Tradeoffs 🤔.

> Switch to the `feign` branch of the source repository to see the changes needed to use a declarative Feign interface.
>
> https://github.com/donovanmuller/echo-example/tree/feign


Let's look at our Feign annotated interface:

```
@FeignClient(name = "chamber", fallback = Echo.EchoFallback.class)
public interface Echo {

	@PostMapping
	String send(String message);

	@Component
	class EchoFallback implements Echo {

		private static final Logger log = LoggerFactory.getLogger(EchoFallback.class);

		@Override
		public String send(String message) {
			log.warn("Hmm, looks like no one's home for echoing message [{}] :(",
					message);

			return message;
		}
	}
}
```

This interface essentially mimics the `restTemplate` URL used before. I.e. `http://chamber/chamber`. It also includes an identical fallback method `noOneHome`.

Our `EchoController` has lost allot of fat:

```
@RestController
@RequestMapping("/echo")
public class EchoController {

	private static final Logger log = LoggerFactory.getLogger(EchoController.class);

	private Echo echo;

	public EchoController(Echo echo) {
		this.echo = echo;
	}

	@PostMapping
	String echo(@RequestBody String message) {
		log.info("Sending echo message: {}", message);

		return echo.send(message);
	}
}
```

as well as our configuration:

```
@Configuration
@EnableCircuitBreaker
@EnableFeignClients
public class EchoConfiguration {

}
```

and our `pom.xml` has the `spring-cloud-starter-feign` included

```
...

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-hystrix</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-feign</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.fabric8</groupId>
                <artifactId>fabric8-maven-plugin</artifactId>
                <executions>
                    <execution>
                        <id>fmp</id>
                        <goals>
                            <goal>resource</goal>
                            <goal>build</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <resources>
                        <env>
                            <!-- Must explicitly enable Hystrix support for Feign -->
                            <!-- see http://cloud.spring.io/spring-cloud-netflix/spring-cloud-netflix.html#spring-cloud-feign-hystrix -->
                            <FEIGN_HYSTRIX_ENABLED>true</FEIGN_HYSTRIX_ENABLED>
                        </env>
                    </resources>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```
Note the `<resources>...<env>` section under the `fabric8-maven-plugin` section. This adds an environment variable of `FEIGN_HYSTRIX_ENABLED=true` to the container when run in OpenShift. By default, Hystrix will not be enabled when using Feign, therefore you must [explicitly enable it](http://cloud.spring.io/spring-cloud-netflix/spring-cloud-netflix.html#spring-cloud-feign-hystrix).

Deploy and test

```
$ curl -w '\n' \
    -D POST \
    -H 'Content-Type: text/plain' \
    --data 'Hello' \
    http://echo-myproject.192.168.64.12.nip.io/echo
HELLO... Hello.. hello
```

All good! 👍
You can test the failure case and observe the same result when using `RestTemplate` and Hystrix.

<iframe width="100%" height="730" src="https://www.youtube.com/embed/2WWRarmdxDY" frameborder="0" allowfullscreen></iframe>

## Conclusion

This quick example showed three ways we could achieve synchronous HTTP based communication between microservices running in OpenShift.

1. Using vanilla `RestTemplate` we could invoke our downstream service by reference it's Service name. The service being down (0 Pods running) caused massive delays.
2. Using Hystrix with Spring Cloud Kubernetes gave us improved discovery, where we only had to reference the Service name and we gained effective timeout and fallback capability from Hystrix. Production here we come... come.. come.
3. For those into writing distributable client APIs, we also showed an equivalent Feign based implementation.

Hopefully you also saw how easy the Fabric8 Maven tooling made deploying Spring Boot applications to OpenShift. We've only scratched the surface of what the tooling offers. One feature that is particularly awesome is the ability to watch for local changes (`mvn fabric8:watch` and reload the Spring Boot application running in OpenShift. Perhaps a more in depth post is in order...