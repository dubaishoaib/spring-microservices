## Setting up Spring Cloud Config Server with Vault and Git repository.

### Set up the vault 
https://docs.spring.io/spring-cloud-config/docs/current/reference/html/#_spring_cloud_config_server

```console
docker compose up
```
**OR**
```console
docker run --cap-add=IPC_LOCK --rm --name vault-server -p 8200:8200 -e 'VAULT_DEV_ROOT_TOKEN_ID=vault-plaintext-root-token' -e 'VAULT_DEV_LISTEN_ADDRESS=0.0.0.0:8200' hashicorp/vault
```

### Connect to the vault
```console
$ docker exec -it vault-server sh
```

```console
echo "Add entries to vault"
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='vault-plaintext-root-token'

echo "Adding a property for all applications (same as application.yml)"
vault kv put secret/application only.in.vault="Hello"
echo "Adding a property for app bar with profile foo"
vault kv put secret/bar,foo a="bar with foo profile value from Vault"
echo "Reading the properties back"
vault kv get secret/application
vault kv get secret/bar,foo
```

### Query the vault
```
curl -H "X-Vault-Request: true" -H "X-Vault-Token: vault-plaintext-root-token" -X GET http://127.0.0.1:8200/v1/secret/da
ta/bar%2Cfoo | jq

curl -H "X-Vault-Request: true" -H "X-Vault-Token: vault-plaintext-root-token" -X GET http://127.0.0.1:8200/v1/secret/data/application | jq
```
```json
{
  "request_id": "2862d63f-33ec-b49a-0719-60464c5166db",
  "lease_id": "",
  "renewable": false,
  "lease_duration": 0,
  "data": {
    "data": {
      "baz": "bam",
      "foo": "bar"
    },
    "metadata": {
      "created_time": "xxxx-xx-20T14:51:13.790737Z",
      "custom_metadata": null,
      "deletion_time": "",
      "destroyed": false,
      "version": 1
    }
  },
  "wrap_info": null,
  "warnings": null,
  "auth": null
}
```

```
curl -H "X-Vault-Request: true" -H "X-Vault-Token: vault-plaintext-root-token" -X GET http://127.0.0.1:8200/v1/secret/data/secret/application | jq
```

### Build Config Server Spring Boot Application
/spring-microservices/spring-cloud-config-server$ mvn clean install -DskipTests

### Run Config Server Spring Boot Application
/spring-microservices/spring-cloud-config-server$ java -jar target/*.jar

### Encrypt the secret
$ curl localhost:8888/encrypt -d "my secret"
9c2d820795ad96ddcea4bcb6c04bd1b7a61c92b866dce16996e45b985a069128
$ curl localhost:8888/decrypt -d "9c2d820795ad96ddcea4bcb6c04bd1b7a61c92b866dce16996e45b985a069128"
my secret

### Push the encrypted secret in property file on GIT
echo "Appending ciphered entry to bar.properties"
echo "encrypted={cipher}9c2d820795ad96ddcea4bcb6c04bd1b7a61c92b866dce16996e45b985a069128" >> bar.properties
git commit -am "Encryption"
git push origin main

### Afert restart the Config Server
$ curl http://localhost:8888/bar-foo.properties -H "X-Config-Token: vault-plaintext-root-token"
config.property: bar with foo profile value for config prop from git
only.in.git: hello
logging.level.com.example: TRACE
encrypted: my secret
only.in.vault: Hello
a: bar with foo profile value from Vault
