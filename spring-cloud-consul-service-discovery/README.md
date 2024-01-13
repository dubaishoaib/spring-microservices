# Spring Boot + Spring Cloud Consul and Service Discovery using Load Balanced RestTemplate, OpenFeign and Spring’s declarative HTTP client

## Building a RESTful Web Service
https://spring.io/guides/gs/rest-service/

## Registering with Consul
https://docs.spring.io/spring-cloud-consul/docs/current/reference/html/#registering-with-consul

## Appendix A: Common application properties (Very helpfull)
https://cloud.spring.io/spring-cloud-consul/reference/html/appendix.html

## Running Consul Server as Docker Instance 
#### Follow the tutorial on HashCorp site and run the Consul docker container
Consul with containers
https://developer.hashicorp.com/consul/tutorials/day-0/docker-container-agents
## OR
### Run the container as Docker Compose service
https://github.com/hashicorp/learn-consul-docker

Goto /spring-microservices/spring-cloud-consul-service-discovery/datacenter-deploy/ folder
Run the following command.

```
spring-microservices/spring-cloud-consul-service-discovery/datacenter-deploy$ echo "Starting Consul"
spring-microservices/spring-cloud-consul-service-discovery/datacenter-deploy$ docker-compose kill || echo "Consul wasn't running previously"
spring-microservices/spring-cloud-consul-service-discovery/datacenter-deploy$ docker-compose rm -v -f || echo "Failed to remove the volume"
spring-microservices/spring-cloud-consul-service-discovery/datacenter-deploy$ ls
README.md  client.json  docker-compose.yml  server.json
spring-microservices/spring-cloud-consul-service-discovery/datacenter-deploy$ docker-compose up -d
```

### Register a service with consul
Goto /spring-microservices/spring-cloud-consul-service-discovery

```
/spring-microservices/spring-cloud-consul-service-discovery/fraud-detection$ mvn clean install -DskipTests
/spring-microservices/spring-cloud-consul-service-discovery/fraud-detection$ mvn spring-boot:run
```

### Discover the service 

```java

@Configuration(proxyBeanMethods = false)
class Config {

	@Bean
	@LoadBalanced
	RestTemplate restTemplate(RestTemplateBuilder builder) {
		return builder.build();
	}

	@Bean
	@LoadBalanced
	WebClient.Builder webClient() {
		return WebClient.builder();
	}

	@Bean
	HttpServiceProxyFactory proxyFactory(WebClient.Builder webClientBuilder) {
		return HttpServiceProxyFactory.builder()
				.clientAdapter(WebClientAdapter.forClient(webClientBuilder
								.baseUrl("http://frauddetection")
						.build()))
				.build();
	}

	@Bean
	DeclarativeFrauds declarativeFrauds(HttpServiceProxyFactory httpServiceProxyFactory) {
		return httpServiceProxyFactory.createClient(DeclarativeFrauds.class);
	}

}

```
Goto /spring-microservices/spring-cloud-consul-service-discovery/loanissuance-loadbalancer

```
/spring-microservices/spring-cloud-consul-service-discovery/loanissuance-loadbalancer$ mvn clean install -DskipTests

```

### Now Test the Service Discovery

Using Load Balanced RestTemplate
```
~$ curl localhost:9081/resttemplate
["josh","marcin"]
```

Using OpenFeign
```
~$ curl localhost:9081/feign
["josh","marcin"]
```

Spring’s declarative HTTP client
```
~$ curl localhost:9081/declarative
["josh","marcin"]
```

### Clean up environment
```
~$ docker-compose down --rmi all
```
