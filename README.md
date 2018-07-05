# water-spring-config-server
Water (WMA) Spring Config Server

## Server Usage
The Water Spring Config Server is a simple application that implements of the Spring Cloud Config Server, and as such the in-depth usage details are available from that project's documentation.

See: https://cloud.spring.io/spring-cloud-config/

This guide will only cover basic usage and how to get an application up and running using this configuration server.

### 1. Providing configuration
This implementation of the Spring Config Server has been configured to load configuration via two Git repos, one for secrets and one for non-secrets. This style of configuration management is still a work-in-progress and may change in the future. In order to link the config server to the Git repos, the following ENV variables should be set:

- `gitSecretRepoUri` - The URI to the Git repo for secrets
- `gitSecretRepoUser` - The username to use when checking out the secrets Git repo.
- `gitSecretRepoPassword` - The password to use when checking out the secrets Git repo.
- `gitConfigRepoUri` - The URI to the Git repo for non-secret configuration
- `gitConfigRepoUser` - The username to use when checking out the non-secret config Git repo.
- `gitConfigRepoPassword` - The password to use when checking out the non-secret config Git repo.

Note that these Git repos do not have to be remote, and can also be accessed from the local filesystem using the `file://` uri syntax. 

Also note that the secret Git repo is given higher precedence than the non-secret repo, meaning that if a configuration property exist for an application in both repos the value in the secret repo will be used.

### 2. Creating a configuration Git repo
Configuration Git repos should contain a sub-directory from the root git repo directory for each application, as well as any configuration files at the root git repo directory level that should apply to all applications. Note that configuration properties loaded from application-specific sub-directories will have higher precedence than cross-application configuration provided at the root level of the repo. 

Application sub-directories **must** be named so that they match the `spring.application.name` value configured for the application that should be accessing them. I.E: An application with a `spring.application.name` value of `my-app` will only be able to load configuration from sub-directories in the configured git repos that are named `my-app`.

Each project sub-directory should contain configuration files just as if they were being loaded by the application locally. This means that the directory should likely have an `application.yml` file containing most of the default application configuration, and then also any other configuration files that the application might be configured to load configuration from.

### 3. Auth
In order to protect secrets client applications that want to connect to the configuration server must provide credentials. The credentials for the configuration server are set via the following ENV variables:

- `configServerUser` - The username client applications should connect with.
- `configServerPassword` - The password client applications should connect with.

It is important to note that when an application gets access to the configuration server using the username and password it can gain access to **ALL** of the secrets and configuration stored on the server. At this time there is no application-specific auth, and as a result access to the config server should be tightly controlled. Apps with highly sensitive secrets should consider hosting their own instance of this server, rather than using any shared, cross-application instances that may be stood up.

### 4. Additional configuration properties
As with other Springboot-based applications there are several additional deployment related configuration properties available:

- `serverPort - [default: 8888]` - The port on which the config server should listen. The default value 8888 is the default value used by the Spring Cloud Config server and its client implementation, so if you change this value away from 8888 you must remember to explicitly set the port in your client application's config server connection configuration.
- `keystoreLocation` - The file path to the keystore file to be used by this application.
- `keystorePassword` - The password to access the configured keystore.
- `keystoreSSLKey` - The alias of the key within the keystore to serve out as the application's SSL cert. This key should have no password or a password equal to the password of they keystore itself.

## Client Usage
In order for a client application to use the config server it must include the Spring Cloud Config starter dependency:

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```
If this is the application's first dependency from the Spring Cloud project then the following will also be necessary (where the `version` value is changed to the version compatible with your application's version of Spring Boot):
```
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Edgware.SR2</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```
Once the dependency is included there should be no additional code changes required to use the config server, only configuration changes which are described in detail below.

### 1. `application.yml` vs `bootstrap.yml`
Using an implementation of the Spring Config Server with a client application introduces the concept of a `bootstrap.yml` file into each client. The `bootstrap.yml` file is required for each client application as it is loaded prior to the loading of all other configuration (including `application.yml`) and is used to confiugre the client's connection to a Spring Config Server instance.

Any configuration placed into the `bootstrap.yml` file is considered to have the lowest level of precedence when it comes to loading in additional configuration, which means that when the `application.yml` file is loaded or any configuration from the config server is loaded any values present in `bootstrap.yml` that are also from another source will be overridden by the loaded values.

### 2. Configuring `bootstrap.yml`
As a result of the behavior described above it is considered a best practice to include in the `bootstrap.yml` file _only_ the configuration that is necessary to startup the application and connect it to the Spring Config Server and access the proper configuration. This process occurs before any other spring boot auto configuration, meaning that you do not need any values for application-specific components in the file. An example `bootstrap.yml` file is shown below:
```
spring:
  profiles:
    active: default, swagger
  application:
    name: aqcu-tss-report
  jmx:
    default-domain: aqcu-tss-report
  cloud:
    config:
      uri: https://localhost:8888
      username: admin
      password: changeMe
```

### 3. Configuration precedence and fallback
When an application that pulls configuration from the Spring Config Server boots, there may be competing sources of configuration from multiple spources: local `application.yml`, local ENV variables, local `bootstrap.yml`, and configuration loaded from the config server. In order to decide what configuration values to use, the application uses the following hierarchy:
```
[bootstrap.yml]

   is overridden by

[default spring boot configuration hierarchy]

   is overridden by

[configuration loaded from the config server]
```
In the event that the Spring Config Server is unreachable, the hierarchy falls back to the next-highest precedence source of configuration.