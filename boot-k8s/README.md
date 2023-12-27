## Spring Boot + Spring Cloud + Kubernetes + Docker + Local Registry + Config Map

## Enable local registry in minikube
[Enabling Insecure Registries](https://minikube.sigs.k8s.io/docs/handbook/registry/)

#### Steps
```console
$ minikube start
$ minikube addons enable registry
```
![minikube message while enabling registry...](minikube-registry-01.jpg)


```console
$ docker run --rm -it --network=host alpine ash -c "apk add socat && socat TCP-LISTEN:5000,reuseaddr,fork TCP:$(minikube ip):5000"
```

#### Test Registry
```console
$ curl http://127.0.0.1:5000/v2/_catalog
{"repositories":[]}

$ docker tag my/image localhost:5000/myimage
$ docker push localhost:5000/myimage

$ curl http://127.0.0.1:5000/v2/_catalog
{"repositories":["localhost:5000/myimage"]}
```
## Setup Application
### application.properties
[application.properties](src%2Fmain%2Fresources%2Fapplication.properties)

[Config file processing in Spring Boot 2.4](https://spring.io/blog/2020/08/14/config-file-processing-in-spring-boot-2-4/)

```console
management.endpoints.web.exposure.include=*
# [Vault] token
#spring.cloud.config.token=vault-plaintext-root-token
# [Config client]
spring.application.name=bar
spring.config.import=optional:configtree:/etc/config/*/
# [Let's fix the port]
server.port=7654
```
## Using Configuration Trees
 When running applications on a cloud platform (such as Kubernetes) you often need to read config values that the platform supplies.
 It is not uncommon to use environment variables for such purposes, but this can have drawbacks, especially if the value is supposed to be kept secret.

 As an alternative to environment variables, many cloud platforms now allow you to map configuration into mounted data volumes. For example, Kubernetes can volume mount both ConfigMaps and Secrets.

 There are two common volume mount patterns that can be used:
    - A single file contains a complete set of properties (usually written as YAML).
    - Multiple files are written to a directory tree, with the filename becoming the ‘key’ and the contents becoming the ‘value’.
 For the first case, you can import the YAML or Properties file directly using spring.config.import as described above. 
 For the second case, you need to use the configtree: prefix so that Spring Boot knows it needs to expose all the files as properties.

 As an example, let’s imagine that Kubernetes has mounted the following volume:
    ```
        etc/
            config/
                myapp/
                    username
                    password
    ```
 The contents of the username file would be a config value, and the contents of password would be a secret.

 To import these properties, you can add the following to your application.properties or application.yaml file:

 Properties
 ```
 spring.config.import=optional:configtree:/etc/config/
 ```

 Yaml
 ```
 spring:
    config:
        import: "optional:configtree:/etc/config/"
 ```
 You can then access or inject myapp.username and myapp.password properties from the Environment in the usual way.

 The folders under the config tree form the property name. In the above example, to access the properties as username and password, 
 you can set spring.config.import to optional:configtree:/etc/config/myapp.
 Filenames with dot notation are also correctly mapped. For example, in the above example, a file named myapp.username 
 in /etc/config would result in a myapp.username property in the Environment.
 Configuration tree values can be bound to both string String and byte[] types depending on the contents expected.
 If you have multiple config trees to import from the same parent folder you can use a wildcard shortcut. 
 Any configtree: location that ends with /*/ will import all immediate children as config trees.

 For example, given the following volume:
```
 etc/
    config/
        dbconfig/
            db/
                username
                password
        mqconfig/
            mq/
                username
                password
 ```
 You can use configtree:/etc/config/*/ as the import location:

 Properties
 ```
 spring.config.import=optional:configtree:/etc/config/*/
 ```

 Yaml
 ```
 spring:
    config:
        import: "optional:configtree:/etc/config/*/"
 ```
 This will add db.username, db.password, mq.username and mq.password properties.

 Directories loaded using a wildcard are sorted alphabetically. If you need a different order, then you should list each location as a separate import.
 Configuration trees can also be used for Docker secrets. When a Docker swarm service is granted access to a secret, 
 the secret gets mounted into the container. 
 For example, if a secret named db.password is mounted at location /run/secrets/, you can make db.password available to the Spring environment using the following:

 Properties
 ```
 spring.config.import=optional:configtree:/run/secrets/
 ```

 Yaml
 ```
 spring:
    config:
        import: "optional:configtree:/run/secrets/"
 ```
## Liveness State
   The “Liveness” state of an application tells whether its internal state allows it to work correctly,
   or recover by itself if it is currently failing. A broken “Liveness” state means that the application is in a state that it cannot recover from, 
   and the infrastructure should restart the application.
   In general, the "Liveness" state should not be based on external checks, such as Health checks. I
   f it did, a failing external system (a database, a Web API, an external cache) would trigger massive restarts and 
   cascading failures across the platform.

   The internal state of Spring Boot applications is mostly represented by the Spring ApplicationContext. 
   If the application context has started successfully, Spring Boot assumes that the application is in a valid state. 
   An application is considered live as soon as the context has been refreshed.

## Readiness State
   The “Readiness” state of an application tells whether the application is ready to handle traffic. 
   A failing “Readiness” state tells the platform that it should not route traffic to the application for now. 
   This typically happens during startup, while CommandLineRunner and ApplicationRunner components are being processed, 
   or at any time if the application decides that it is too busy for additional traffic.

   An application is considered ready as soon as application and command-line runners have been called.

   Tasks expected to run during startup should be executed by CommandLineRunner and ApplicationRunner components instead 
   of using Spring component lifecycle callbacks such as @PostConstruct.

### Creating the config map
```
/spring-microservices/boot-k8s/k8s$ kubectl apply -f config-map.yml
```

### Creating the secret
```
/spring-microservices/boot-k8s/k8s$ kubectl create secret generic config-client-secret --from-literal=encrypted=mysecret
```
### To make the local registry work we will tag the image with localhost:5000
```
/spring-microservices/boot-k8s$ mvn spring-boot:build-image -Dspring-boot.build-image.imageName=localhost:5000/config-client:latest -DskipTests
```

```
$ docker images
REPOSITORY                            TAG       IMAGE ID       CREATED         SIZE
paketobuildpacks/run-jammy-base       latest    cafe9a5d07e5   6 days ago      104MB
alpine                                latest    f8c20f8bbcb6   2 weeks ago     7.38MB
localhost:5000/config-client          latest    02b6534363d5   44 years ago    310MB
paketobuildpacks/builder-jammy-base   latest    ac5f06f79b12   44 years ago    1.55GB
```

### We're pushing the image to the local registry
```
/spring-microservices/boot-k8s$ docker push localhost:5000/config-client:latest
....

/spring-microservices/boot-k8s$ curl http://127.0.0.1:5000/v2/_catalog
{"repositories":["config-client"]}
```
### Deploying the app to Kubernetes
```
/spring-microservices/boot-k8s/k8s$ kubectl apply -f deployment.yml
deployment.apps/config-client created

/spring-microservices/boot-k8s/k8s$ kubectl describe deployment config-client

/spring-microservices/boot-k8s/k8s$ kubectl get deployments
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
config-client   1/1     1            1           2m2s

/spring-microservices/boot-k8s/k8s$ kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
config-client-7f769858bc-5vn55   1/1     Running   0          2m28s
```

### Making the pod accessible locally on the same port 7654
```
$ kubectl port-forward deployment/config-client 7654:7654
Forwarding from 127.0.0.1:7654 -> 7654
```

### Testing if K8s configuration works fine

### Should return <a value from config>"
```
$ curl localhost:7654/refreshed
a value from config
```

### Should return <value for config prop from kubernetes>
```
$ curl localhost:7654/configprop
value for config prop from kubernetes
```

### Should return <mysecret>
```
$ curl localhost:7654/encrypted
mysecret
```

### Updating the config map
```
/spring-microservices/boot-k8s/k8s$ kubectl apply -f config-map-changed.yml
```

### Wait some time ... You can check whether the properties have been updated
```
$ kubectl exec deployment/config-client -- cat /etc/config/default/a
```

### Refresh the application
```
$ curl -X POST localhost:7654/actuator/refresh
```

### Testing if K8s configuration works fine with the updated info
```
$ kubectl exec deployment/config-client -- cat /etc/config/default/a
a value from config REFRESHED
```
### Should return <a value from config REFRESHED>
```
$ curl localhost:7654/refreshed
a value from config REFRESHED
```

### Should return <value for config prop from kubernetes REFRESHED>
```
$ curl localhost:7654/configprop
value for config prop from kubernetes REFRESHED
```

### Should return <mysecret>
```
$ curl localhost:7654/encrypted
mysecret
```
