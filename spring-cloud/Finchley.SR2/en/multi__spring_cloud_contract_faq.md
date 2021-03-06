## 90. Spring Cloud Contract FAQ

## 90.1 Why use Spring Cloud Contract Verifier and not X ?

For the time being Spring Cloud Contract is a JVM based tool. So it could be your first pick when you’re already creating software for the JVM. This project has a lot of really interesting features but especially quite a few of them definitely make Spring Cloud Contract Verifier stand out on the "market" of Consumer Driven Contract (CDC) tooling. Out of many the most interesting are:

- Possibility to do CDC with messaging

- Clear and easy to use, statically typed DSL

- Possibility to copy paste your current JSON file to the contract and only edit its elements

- Automatic generation of tests from the defined Contract

- Stub Runner functionality - the stubs are automatically downloaded at runtime from Nexus / Artifactory

- Spring Cloud integration - no discovery service is needed for integration tests

- Spring Cloud Contract integrates with Pact out of the box and provides easy hooks to extend its functionality

- Via Docker adds support for any language & framework used

## 90.2 I don’t want to write a contract in Groovy!

No problem. You can write a contract in YAML!

## 90.3 What is this value(consumer(), producer()) ?

One of the biggest challenges related to stubs is their reusability. Only if they can be vastly used, will they serve their purpose. What typically makes that difficult are the hard-coded values of request / response elements. For example dates or ids. Imagine the following JSON request

```java
{
"time" : "2016-10-10 20:10:15",
"id" : "9febab1c-6f36-4a0b-88d6-3b6a6d81cd4a",
"body" : "foo"
}
```

and JSON response

```java
{
"time" : "2016-10-10 21:10:15",
"id" : "c4231e1f-3ca9-48d3-b7e7-567d55f0d051",
"body" : "bar"
}
```

Imagine the pain required to set proper value of the  `time`  field (let’s assume that this content is generated by the database) by changing the clock in the system or providing stub implementations of data providers. The same is related to the field called  `id` . Will you create a stubbed implementation of UUID generator? Makes little sense…

So as a consumer you would like to send a request that matches any form of a time or any UUID. That way your system will work as usual - will generate data and you won’t have to stub anything out. Let’s assume that in case of the aforementioned JSON the most important part is the  `body`  field. You can focus on that and provide matching for other fields. In other words you would like the stub to work like this:

```java
{
"time" : "SOMETHING THAT MATCHES TIME",
"id" : "SOMETHING THAT MATCHES UUID",
"body" : "foo"
}
```

As far as the response goes as a consumer you need a concrete value that you can operate on. So such a JSON is valid

```java
{
"time" : "2016-10-10 21:10:15",
"id" : "c4231e1f-3ca9-48d3-b7e7-567d55f0d051",
"body" : "bar"
}
```

As you could see in the previous sections we generate tests from contracts. So from the producer’s side the situation looks much different. We’re parsing the provided contract and in the test we want to send a real request to your endpoints. So for the case of a producer for the request we can’t have any sort of matching. We need concrete values that the producer’s backend can work on. Such a JSON would be a valid one:

```java
{
"time" : "2016-10-10 20:10:15",
"id" : "9febab1c-6f36-4a0b-88d6-3b6a6d81cd4a",
"body" : "foo"
}
```

On the other hand from the point of view of the validity of the contract the response doesn’t necessarily have to contain concrete values of  `time`  or  `id` . Let’s say that you generate those on the producer side - again, you’d have to do a lot of stubbing to ensure that you always return the same values. That’s why from the producer’s side what you might want is the following response:

```java
{
"time" : "SOMETHING THAT MATCHES TIME",
"id" : "SOMETHING THAT MATCHES UUID",
"body" : "bar"
}
```

How can you then provide one time a matcher for the consumer and a concrete value for the producer and vice versa? In Spring Cloud Contract we’re allowing you to provide a  **dynamic value** . That means that it can differ for both sides of the communication. You can pass the values:

Either via the  `value`  method

```java
value(consumer(...), producer(...))
value(stub(...), test(...))
value(client(...), server(...))
```

or using the  `$()`  method

```java
$(consumer(...), producer(...))
$(stub(...), test(...))
$(client(...), server(...))
```

You can read more about this in the [Chapter 95, Contract DSL](multi_contract-dsl.html) section.

Calling  `value()`  or  `$()`  tells Spring Cloud Contract that you will be passing a dynamic value. Inside the  `consumer()`  method you pass the value that should be used on the consumer side (in the generated stub). Inside the  `producer()`  method you pass the value that should be used on the producer side (in the generated test).

> If on one side you have passed the regular expression and you haven’t passed the other, then the other side will get auto-generated.

Most often you will use that method together with the  `regex`  helper method. E.g.  `consumer(regex('[0-9]{10}'))` .

To sum it up the contract for the aforementioned scenario would look more or less like this (the regular expression for time and UUID are simplified and most likely invalid but we want to keep things very simple in this example):

```java
org.springframework.cloud.contract.spec.Contract.make {
				request {
					method 'GET'
					url '/someUrl'
					body([
					    time : value(consumer(regex('[0-9]{4}-[0-9]{2}-[0-9]{2} [0-2][0-9]-[0-5][0-9]-[0-5][0-9]')),
					    id: value(consumer(regex('[0-9a-zA-z]{8}-[0-9a-zA-z]{4}-[0-9a-zA-z]{4}-[0-9a-zA-z]{12}'))
					    body: "foo"
					])
				}
			response {
				status OK()
				body([
					    time : value(producer(regex('[0-9]{4}-[0-9]{2}-[0-9]{2} [0-2][0-9]-[0-5][0-9]-[0-5][0-9]')),
					    id: value([producer(regex('[0-9a-zA-z]{8}-[0-9a-zA-z]{4}-[0-9a-zA-z]{4}-[0-9a-zA-z]{12}'))
					    body: "bar"
					])
			}
}
```

|images/important.png|Important|
|----|----|
|Please read the [Groovy docs related to JSON](http://groovy-lang.org/json.html) to understand how to properly structure the request / response bodies. |

## 90.4 How to do Stubs versioning?

### 90.4.1 API Versioning

Let’s try to answer a question what versioning really means. If you’re referring to the API version then there are different approaches.

- use Hypermedia, links and do not version your API by any means

- pass versions through headers / urls

I will not try to answer a question which approach is better. Whatever suit your needs and allows you to generate business value should be picked.

Let’s assume that you do version your API. In that case you should provide as many contracts as many versions you support. You can create a subfolder for every version or append it to th contract name - whatever suits you more.

### 90.4.2 JAR versioning

If by versioning you mean the version of the JAR that contains the stubs then there are essentially two main approaches.

Let’s assume that you’re doing Continuous Delivery / Deployment which means that you’re generating a new version of the jar each time you go through the pipeline and that jar can go to production at any time. For example your jar version looks like this (it got built on the 20.10.2016 at 20:15:21) :

```java
1.0.0.20161020-201521-RELEASE
```

In that case your generated stub jar will look like this.

```java
1.0.0.20161020-201521-RELEASE-stubs.jar
```

In this case you should inside your  `application.yml`  or  `@AutoConfigureStubRunner`  when referencing stubs provide the latest version of the stubs. You can do that by passing the  `+`  sign. Example

```java
@AutoConfigureStubRunner(ids = {"com.example:http-server-dsl:+:stubs:8080"})
```

If the versioning however is fixed (e.g.  `1.0.4.RELEASE`  or  `2.1.1` ) then you have to set the concrete value of the jar version. Example for 2.1.1.

```java
@AutoConfigureStubRunner(ids = {"com.example:http-server-dsl:2.1.1:stubs:8080"})
```

### 90.4.3 Dev or prod stubs

You can manipulate the classifier to run the tests against current development version of the stubs of other services or the ones that were deployed to production. If you alter your build to deploy the stubs with the  `prod-stubs`  classifier once you reach production deployment then you can run tests in one case with dev stubs and one with prod stubs.

Example of tests using development version of stubs

```java
@AutoConfigureStubRunner(ids = {"com.example:http-server-dsl:+:stubs:8080"})
```

Example of tests using production version of stubs

```java
@AutoConfigureStubRunner(ids = {"com.example:http-server-dsl:+:prod-stubs:8080"})
```

You can pass those values also via properties from your deployment pipeline.

## 90.5 Common repo with contracts

Another way of storing contracts other than having them with the producer is keeping them in a common place. It can be related to security issues where the consumers can’t clone the producer’s code. Also if you keep contracts in a single place then you, as a producer, will know how many consumers you have and which consumer will you break with your local changes.

### 90.5.1 Repo structure

Let’s assume that we have a producer with coordinates  `com.example:server`  and 3 consumers:  `client1` ,  `client2` ,  `client3` . Then in the repository with common contracts you would have the following setup (which you can checkout [here](https://github.com/spring-cloud/spring-cloud-contract/tree/2.0.x/samples/standalone/contracts)):

```java
├── com
│ └── example
│     └── server
│         ├── client1
│         │ └── expectation.groovy
│         ├── client2
│         │ └── expectation.groovy
│         ├── client3
│         │ └── expectation.groovy
│         └── pom.xml
├── mvnw
├── mvnw.cmd
├── pom.xml
└── src
└── assembly
└── contracts.xml
```

As you can see the under the slash-delimited groupid  `/`  artifact id folder ( `com/example/server` ) you have expectations of the 3 consumers ( `client1` ,  `client2`  and  `client3` ). Expectations are the standard Groovy DSL contract files as described throughout this documentation. This repository has to produce a JAR file that maps one to one to the contents of the repo.

Example of a  `pom.xml`  inside the  `server`  folder.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>com.example</groupId>
	<artifactId>server</artifactId>
	<version>0.0.1-SNAPSHOT</version>

	<name>Server Stubs</name>
	<description>POM used to install locally stubs for consumer side</description>

	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.0.6.RELEASE</version>
		<relativePath />
	</parent>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<java.version>1.8</java.version>
		<spring-cloud-contract.version>2.0.3.BUILD-SNAPSHOT</spring-cloud-contract.version>
		<spring-cloud-release.version>Finchley.BUILD-SNAPSHOT</spring-cloud-release.version>
		<excludeBuildFolders>true</excludeBuildFolders>
	</properties>

	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>${spring-cloud-release.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-contract-maven-plugin</artifactId>
				<version>${spring-cloud-contract.version}</version>
				<extensions>true</extensions>
				<configuration>
					<!-- By default it would search under src/test/resources/ -->
					<contractsDirectory>${project.basedir}</contractsDirectory>
				</configuration>
			</plugin>
		</plugins>
	</build>

	<repositories>
		<repository>
			<id>spring-snapshots</id>
			<name>Spring Snapshots</name>
			<url>https://repo.spring.io/snapshot</url>
			<snapshots>
				<enabled>true</enabled>
			</snapshots>
		</repository>
		<repository>
			<id>spring-milestones</id>
			<name>Spring Milestones</name>
			<url>https://repo.spring.io/milestone</url>
			<snapshots>
				<enabled>false</enabled>
			</snapshots>
		</repository>
		<repository>
			<id>spring-releases</id>
			<name>Spring Releases</name>
			<url>https://repo.spring.io/release</url>
			<snapshots>
				<enabled>false</enabled>
			</snapshots>
		</repository>
	</repositories>
	<pluginRepositories>
		<pluginRepository>
			<id>spring-snapshots</id>
			<name>Spring Snapshots</name>
			<url>https://repo.spring.io/snapshot</url>
			<snapshots>
				<enabled>true</enabled>
			</snapshots>
		</pluginRepository>
		<pluginRepository>
			<id>spring-milestones</id>
			<name>Spring Milestones</name>
			<url>https://repo.spring.io/milestone</url>
			<snapshots>
				<enabled>false</enabled>
			</snapshots>
		</pluginRepository>
		<pluginRepository>
			<id>spring-releases</id>
			<name>Spring Releases</name>
			<url>https://repo.spring.io/release</url>
			<snapshots>
				<enabled>false</enabled>
			</snapshots>
		</pluginRepository>
	</pluginRepositories>

</project>
```

As you can see there are no dependencies other than the Spring Cloud Contract Maven Plugin. Those poms are necessary for the consumer side to run  `mvn clean install -DskipTests`  to locally install stubs of the producer project.

The  `pom.xml`  in the root folder can look like this:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		 xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>com.example.standalone</groupId>
	<artifactId>contracts</artifactId>
	<version>0.0.1-SNAPSHOT</version>

	<name>Contracts</name>
	<description>Contains all the Spring Cloud Contracts, well, contracts. JAR used by the producers to generate tests and stubs</description>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
	</properties>

	<build>
		<plugins>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-assembly-plugin</artifactId>
				<executions>
					<execution>
						<id>contracts</id>
						<phase>prepare-package</phase>
						<goals>
							<goal>single</goal>
						</goals>
						<configuration>
							<attach>true</attach>
							<descriptor>${basedir}/src/assembly/contracts.xml</descriptor>
							<!-- If you want an explicit classifier remove the following line -->
							<appendAssemblyId>false</appendAssemblyId>
						</configuration>
					</execution>
				</executions>
			</plugin>
		</plugins>
	</build>

</project>
```

It’s using the assembly plugin in order to build the JAR with all the contracts. Example of such setup is here:

```xml
<assembly xmlns="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.3"
		  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		  xsi:schemaLocation="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.3 http://maven.apache.org/xsd/assembly-1.1.3.xsd">
	<id>project</id>
	<formats>
		<format>jar</format>
	</formats>
	<includeBaseDirectory>false</includeBaseDirectory>
	<fileSets>
		<fileSet>
			<directory>${project.basedir}</directory>
			<outputDirectory>/</outputDirectory>
			<useDefaultExcludes>true</useDefaultExcludes>
			<excludes>
				<exclude>**/${project.build.directory}/**</exclude>
				<exclude>mvnw</exclude>
				<exclude>mvnw.cmd</exclude>
				<exclude>.mvn/**</exclude>
				<exclude>src/**</exclude>
			</excludes>
		</fileSet>
	</fileSets>
</assembly>
```

### 90.5.2 Workflow

The workflow would look similar to the one presented in the  `Step by step guide to CDC` . The only difference is that the producer doesn’t own the contracts anymore. So the consumer and the producer have to work on common contracts in a common repository.

### 90.5.3 Consumer

When the  **consumer**  wants to work on the contracts offline, instead of cloning the producer code, the consumer team clones the common repository, goes to the required producer’s folder (e.g.  `com/example/server` ) and runs  `mvn clean install -DskipTests`  to install locally the stubs converted from the contracts.

> You need to have [Maven installed locally](https://maven.apache.org/download.cgi)

### 90.5.4 Producer

As a  **producer**  it’s enough to alter the Spring Cloud Contract Verifier to provide the URL and the dependency of the JAR containing the contracts:

```xml
<plugin>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-contract-maven-plugin</artifactId>
	<configuration>
		<contractsMode>REMOTE</contractsMode>
		<contractsRepositoryUrl>http://link/to/your/nexus/or/artifactory/or/sth</contractsRepositoryUrl>
		<contractDependency>
			<groupId>com.example.standalone</groupId>
			<artifactId>contracts</artifactId>
		</contractDependency>
	</configuration>
</plugin>
```

With this setup the JAR with groupid  `com.example.standalone`  and artifactid  `contracts`  will be downloaded from  `http://link/to/your/nexus/or/artifactory/or/sth` . It will be then unpacked in a local temporary folder and contracts present under the  `com/example/server`  will be picked as the ones used to generate the tests and the stubs. Due to this convention the producer team will know which consumer teams will be broken when some incompatible changes are done.

The rest of the flow looks the same.

### 90.5.5 How can I define messaging contracts per topic not per producer?

To avoid messaging contracts duplication in the common repo, when few producers writing messages to one topic, we could create the structure when the rest contracts would be placed in a folder per producer and messaging contracts in the folder per topic.

#### For Maven Project

To make it possible to work on the producer side we could do the following things (all via Maven plugins):

- Add common repo dependency to your classpath:

```xml
<dependency>
<groupId>com.example</groupId>
<artifactId>common-repo</artifactId>
<version>${common-repo.version}</version>
</dependency>
```

- Download the JAR with the contracts and unpack the JAR to target:

```xml
<plugin>
<groupId>org.apache.maven.plugins</groupId>
<artifactId>maven-dependency-plugin</artifactId>
<version>3.0.0</version>
<executions>
<execution>
<id>unpack-dependencies</id>
<phase>process-resources</phase>
<goals>
<goal>unpack</goal>
</goals>
<configuration>
<artifactItems>
<artifactItem>
<groupId>com.example</groupId>
<artifactId>common-repo</artifactId>
<type>jar</type>
<overWrite>false</overWrite>
<outputDirectory>${project.build.directory}/contracts</outputDirectory>
</artifactItem>
</artifactItems>
</configuration>
</execution>
</executions>
</plugin>
```

- Rip out all the folders we’re not interested in:

```xml
<plugin>
<groupId>org.apache.maven.plugins</groupId>
<artifactId>maven-antrun-plugin</artifactId>
<version>1.8</version>
<executions>
<execution>
<phase>process-resources</phase>
<goals>
<goal>run</goal>
</goals>
<configuration>
<tasks>
<delete includeemptydirs="true">
<fileset dir="${project.build.directory}/contracts">
<include name="**/*" />
<!--Producer artifactId-->
<exclude name="**/${project.artifactId}/**" />
<!--List of the supported topics-->
<exclude name="**/${first-topic}/**" />
<exclude name="**/${second-topic}/**" />
</fileset>
</delete>
</tasks>
</configuration>
</execution>
</executions>
</plugin>
```

- Run the contract plugin by pointing to the contracts to the folder under target:

```xml
<plugin>
<groupId>org.springframework.cloud</groupId>
<artifactId>spring-cloud-contract-maven-plugin</artifactId>
<version>${spring-cloud-contract.version}</version>
<extensions>true</extensions>
<configuration>
<packageWithBaseClasses>com.example</packageWithBaseClasses>
<baseClassMappings>
<baseClassMapping>
<contractPackageRegex>.*intoxication.*</contractPackageRegex>
<baseClassFQN>com.example.intoxication.BeerIntoxicationBase</baseClassFQN>
</baseClassMapping>
</baseClassMappings>
<contractsDirectory>${project.build.directory}/contracts</contractsDirectory>
</configuration>
</plugin>
```

#### For Gradle Project

- Add a custom configuration for the common-repo dependency:

```java
ext {
conractsGroupId = "com.example"
contractsArtifactId = "common-repo"
contractsVersion = "1.2.3"
}

configurations {
contracts {
transitive = false
}
}
```

- Add the common-repo dependency to your classpath:

```java
dependencies {
contracts "${conractsGroupId}:${contractsArtifactId}:${contractsVersion}"
testCompile "${conractsGroupId}:${contractsArtifactId}:${contractsVersion}"
}
```

- Download the dependency to an appropriate folder:

```java
task getContracts(type: Copy) {
from configurations.contracts
into new File(project.buildDir, "downloadedContracts")
}
```

- Unzip JAR:

```java
task unzipContracts(type: Copy) {
def zipFile = new File(project.buildDir, "downloadedContracts/${contractsArtifactId}-${contractsVersion}.jar")
def outputDir = file("${buildDir}/unpackedContracts")

from zipTree(zipFile)
into outputDir
}
```

- Cleanup unused contracts:

```java
task deleteUnwantedContracts(type: Delete) {
delete fileTree(dir: "${buildDir}/unpackedContracts",
include: "**/*",
excludes: [
"**/${project.name}/**"",
"**/${first-topic}/**",
"**/${second-topic}/**"])
}
```

- Create task dependencies:

```java
unzipContracts.dependsOn("getContracts")
deleteUnwantedContracts.dependsOn("unzipContracts")
build.dependsOn("deleteUnwantedContracts")
```

- Configure plugin by specifying the directory containing contracts using  `contractsDslDir`  property

```java
contracts {
contractsDslDir = new File("${buildDir}/unpackedContracts")
}
```

## 90.6 Do I need a Binary Storage? Can’t I use Git?

In the polyglot world, there are languages that don’t use binary storages like Artifactory or Nexus. Starting from Spring Cloud Contract version 2.0.0 we provide mechanisms to store contracts and stubs in a SCM repository. Currently the only supported SCM is Git.

The repository would have to the following setup (which you can checkout [here](https://github.com/spring-cloud-samples/spring-cloud-contract-samples/tree/master/contracts_git/)):

```java
.
└── META-INF
└── com.example
└── beer-api-producer-git
└── 0.0.1-SNAPSHOT
├── contracts
│ └── beer-api-consumer
│     ├── messaging
│     │ ├── shouldSendAcceptedVerification.groovy
│     │ └── shouldSendRejectedVerification.groovy
│     └── rest
│         ├── shouldGrantABeerIfOldEnough.groovy
│         └── shouldRejectABeerIfTooYoung.groovy
└── mappings
└── beer-api-consumer
└── rest
├── shouldGrantABeerIfOldEnough.json
└── shouldRejectABeerIfTooYoung.json
```

Under  `META-INF`  folder:

- we group applications via  `groupId`  (e.g.  `com.example` )

- then each application is represented via the  `artifactId`  (e.g.  `beer-api-producer-git` )

- next, the version of the application. The version is mandatory! (e.g.  `0.0.1-SNAPSHOT` )

- finally, there are two folders:

-  `contracts`  - the good practice is to store the contracts required by each consumer in the folder with the consumer name (e.g.  `beer-api-consumer` ). That way you can use the  `stubs-per-consumer`  feature. Further directory structure is arbitrary.

-  `mappings`  - in this folder the Maven / Gradle Spring Cloud Contract plugins will push the stub server mappings. On the consumer side, Stub Runner will scan this folder to start stub servers with stub definitions. The folder structure will be a copy of the one created in the  `contracts`  subfolder.

### 90.6.1 Protocol convention

In order to control the type and location of the source of contracts (whether it’s a binary storage or an SCM repository), you can use the protocol in the URL of the repository. Spring Cloud Contract iterates over registered protocol resolvers and tries to fetch the contracts (via a plugin) or stubs (via Stub Runner).

For the SCM functionality, currently, we support the Git repository. To use it, in the property, where the repository URL needs to be placed you just have to prefix the connection URL with  `git://` . Here you can find a couple of examples:

```java
git://file:///foo/bar
git://https://github.com/spring-cloud-samples/spring-cloud-contract-nodejs-contracts-git.git
git://[emailprotected]:spring-cloud-samples/spring-cloud-contract-nodejs-contracts-git.git
```

### 90.6.2 Producer

For the producer, to use the SCM approach, we can reuse the same mechanism we use for external contracts. We route Spring Cloud Contract to use the SCM implementation via the URL that contains the  `git://`  protocol.

|images/important.png|Important|
|----|----|
|You have to manually add the  `pushStubsToScm`  goal in Maven or execute (bind) the  `pushStubsToScm`  task in Gradle. We don’t push stubs to  `origin`  of your git repository out of the box. |

**Maven.**  

```xml
<plugin>
<groupId>org.springframework.cloud</groupId>
<artifactId>spring-cloud-contract-maven-plugin</artifactId>
<version>${spring-cloud-contract.version}</version>
<extensions>true</extensions>
<configuration>
<!-- Base class mappings etc. -->

<!-- We want to pick contracts from a Git repository -->
<contractsRepositoryUrl>git://https://github.com/spring-cloud-samples/spring-cloud-contract-nodejs-contracts-git.git</contractsRepositoryUrl>

<!-- We reuse the contract dependency section to set up the path
to the folder that contains the contract definitions. In our case the
path will be /groupId/artifactId/version/contracts -->
<contractDependency>
<groupId>${project.groupId}</groupId>
<artifactId>${project.artifactId}</artifactId>
<version>${project.version}</version>
</contractDependency>

<!-- The contracts mode can't be classpath -->
<contractsMode>REMOTE</contractsMode>
</configuration>
<executions>
<execution>
<phase>package</phase>
<goals>
<!-- By default we will not push the stubs back to SCM,
you have to explicitly add it as a goal -->
<goal>pushStubsToScm</goal>
</goals>
</execution>
</executions>
</plugin>
```

**Gradle.**  

```java
contracts {
	// We want to pick contracts from a Git repository
	contractDependency {
		stringNotation = "${project.group}:${project.name}:${project.version}"
	}
	/*
	We reuse the contract dependency section to set up the path
	to the folder that contains the contract definitions. In our case the
	path will be /groupId/artifactId/version/contracts
	 */
	contractRepository {
		repositoryUrl = "git://https://github.com/spring-cloud-samples/spring-cloud-contract-nodejs-contracts-git.git"
	}
	// The mode can't be classpath
	contractsMode = "REMOTE"
	// Base class mappings etc.
}

/*
In this scenario we want to publish stubs to SCM whenever
the `publish` task is executed
*/
publish.dependsOn("publishStubsToScm")
```

With such a setup:

- Git project will be cloned to a temporary directory

- The SCM stub downloader will go to  `META-INF/groupId/artifactId/version/contracts`  folder to find contracts. E.g. for  `com.example:foo:1.0.0`  the path would be  `META-INF/com.example/foo/1.0.0/contracts` 

- Tests will be generated from the contracts

- Stubs will be created from the contracts

- Once the tests pass, the stubs will be committed in the cloned repository

- Finally, a push will be done to that repo’s  `origin` 

#### Keeping contracts with the producer and stubs in an external repository

It is also possible to keep the contracts in the producer repository, but keep the stubs in an external git repo. This is most useful when you want to use the base consumer-producer collaboration flow, but do not have a possibility to use an artifact repository for storing the stubs.

In order to do that, use the usual producer setup, and then add the  `pushStubsToScm`  goal and set  `contractsRepositoryUrl`  to the repository where you want to keep the stubs.

### 90.6.3 Consumer

On the consumer side when passing the  `repositoryRoot`  parameter, either from the  `@AutoConfigureStubRunner`  annotation, the JUnit rule or properties, it’s enough to pass the URL of the SCM repository, prefixed with the protocol. For example

```java
@AutoConfigureStubRunner(
stubsMode="REMOTE",
repositoryRoot="git://https://github.com/spring-cloud-samples/spring-cloud-contract-nodejs-contracts-git.git",
ids="com.example:bookstore:0.0.1.RELEASE"
)
```

With such a setup:

- Git project will be cloned to a temporary directory

- The SCM stub downloader will go to  `META-INF/groupId/artifactId/version/`  folder to find stub definitions and contracts. E.g. for  `com.example:foo:1.0.0`  the path would be  `META-INF/com.example/foo/1.0.0/` 

- Stub servers will be started and fed with mappings

- Messaging definitions will be read and used in the messaging tests

## 90.7 Can I use the Pact Broker?

When using [Pact](http://pact.io/) you can use the [Pact Broker](https://github.com/pact-foundation/pact_broker) to store and share Pact definitions. Starting from Spring Cloud Contract 2.0.0 one can fetch Pact files from the Pact Broker to generate tests and stubs.

As a prerequisite the Pact Converter and Pact Stub Downloader are required. You have to add it via the  `spring-cloud-contract-pact`  dependency. You can read more about it in the [Section 97.1.1, “Pact Converter”](multi__using_the_pluggable_architecture.html#pact-converter) section.

|images/important.png|Important|
|----|----|
|Pact follows the Consumer Contract convention. That means that the Consumer creates the Pact definitions first, then shares the files with the Producer. Those expectations are generated from the Consumer’s code and can break the Producer if the expectation is not met. |

### 90.7.1 Pact Consumer

The consumer uses Pact framework to generate Pact files. The Pact files are sent to the Pact Broker. An example of such setup can be found [here](https://github.com/spring-cloud-samples/spring-cloud-contract-samples/tree/master/consumer_pact).

### 90.7.2 Producer

For the producer, to use the Pact files from the Pact Broker, we can reuse the same mechanism we use for external contracts. We route Spring Cloud Contract to use the Pact implementation via the URL that contains the  `pact://`  protocol. It’s enough to pass the URL to the Pact Broker. An example of such setup can be found [here](https://github.com/spring-cloud-samples/spring-cloud-contract-samples/tree/master/producer_pact).

**Maven.**  

```xml
<plugin>
<groupId>org.springframework.cloud</groupId>
<artifactId>spring-cloud-contract-maven-plugin</artifactId>
<version>${spring-cloud-contract.version}</version>
<extensions>true</extensions>
<configuration>
<!-- Base class mappings etc. -->

<!-- We want to pick contracts from a Git repository -->
<contractsRepositoryUrl>pact://http://localhost:8085</contractsRepositoryUrl>

<!-- We reuse the contract dependency section to set up the path
to the folder that contains the contract definitions. In our case the
path will be /groupId/artifactId/version/contracts -->
<contractDependency>
<groupId>${project.groupId}</groupId>
<artifactId>${project.artifactId}</artifactId>
<!-- When + is passed, a latest tag will be applied when fetching pacts -->
<version>+</version>
</contractDependency>

<!-- The contracts mode can't be classpath -->
<contractsMode>REMOTE</contractsMode>
</configuration>
<!-- Don't forget to add spring-cloud-contract-pact to the classpath! -->
<dependencies>
<dependency>
<groupId>org.springframework.cloud</groupId>
<artifactId>spring-cloud-contract-pact</artifactId>
<version>${spring-cloud-contract.version}</version>
</dependency>
</dependencies>
</plugin>
```

**Gradle.**  

```java
buildscript {
	repositories {
		//...
	}

	dependencies {
		// ...
		// Don't forget to add spring-cloud-contract-pact to the classpath!
		classpath "org.springframework.cloud:spring-cloud-contract-pact:${contractVersion}"
	}
}

contracts {
	// When + is passed, a latest tag will be applied when fetching pacts
	contractDependency {
		stringNotation = "${project.group}:${project.name}:+"
	}
	contractRepository {
		repositoryUrl = "pact://http://localhost:8085"
	}
	// The mode can't be classpath
	contractsMode = "REMOTE"
	// Base class mappings etc.
}
```

With such a setup:

- Pact files will be downloaded from the Pact Broker

- Spring Cloud Contract will convert the Pact files into tests and stubs

- The JAR with the stubs gets automatically created as usual

### 90.7.3 Pact Consumer (Producer Contract approach)

In the scenario where you don’t want to do Consumer Contract approach (for every single consumer define the expectations) but you’d prefer to do Producer Contracts (the producer provides the contracts and publishes stubs), it’s enough to use Spring Cloud Contract with Stub Runner option. An example of such setup can be found [here](https://github.com/spring-cloud-samples/spring-cloud-contract-samples/tree/master/consumer_pact_stubrunner).

First, remember to add Stub Runner and Spring Cloud Contract Pact module as test dependencies.

**Maven.**  

```xml
<dependencyManagement>
<dependencies>
<dependency>
<groupId>org.springframework.cloud</groupId>
<artifactId>spring-cloud-dependencies</artifactId>
<version>${spring-cloud.version}</version>
<type>pom</type>
<scope>import</scope>
</dependency>
</dependencies>
</dependencyManagement>

<!-- Don't forget to add spring-cloud-contract-pact to the classpath! -->
<dependencies>
<!-- ... -->
<dependency>
<groupId>org.springframework.cloud</groupId>
<artifactId>spring-cloud-starter-contract-stub-runner</artifactId>
<scope>test</scope>
</dependency>
<dependency>
<groupId>org.springframework.cloud</groupId>
<artifactId>spring-cloud-contract-pact</artifactId>
<scope>test</scope>
</dependency>
</dependencies>
```

**Gradle.**  

```java
dependencyManagement {
imports {
mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
}
}

dependencies {
//...
testCompile("org.springframework.cloud:spring-cloud-starter-contract-stub-runner")
// Don't forget to add spring-cloud-contract-pact to the classpath!
testCompile("org.springframework.cloud:spring-cloud-contract-pact")
}
```

Next, just pass the URL of the Pact Broker to  `repositoryRoot` , prefixed with  `pact://`  protocol. E.g.  `pact://http://localhost:8085` 

```java
@RunWith(SpringRunner.class)
@SpringBootTest
@AutoConfigureStubRunner(stubsMode = StubRunnerProperties.StubsMode.REMOTE,
		ids = "com.example:beer-api-producer-pact",
		repositoryRoot = "pact://http://localhost:8085")
public class BeerControllerTest {
//Inject the port of the running stub
@StubRunnerPort("beer-api-producer-pact") int producerPort;
//...
}
```

With such a setup:

- Pact files will be downloaded from the Pact Broker

- Spring Cloud Contract will convert the Pact files into stub definitions

- The stub servers will be started and fed with stubs

For more information about Pact support you can go to the [Section 97.7, “Using the Pact Stub Downloader”](multi__using_the_pluggable_architecture.html#pact-stub-downloader) section.

## 90.8 How can I debug the request/response being sent by the generated tests client?

The generated tests all boil down to RestAssured in some form or fashion which relies on [Apache HttpClient](https://hc.apache.org/httpcomponents-client-ga/). HttpClient has a facility called [wire logging](https://hc.apache.org/httpcomponents-client-ga/logging.html#Wire_Logging) which logs the entire request and response to HttpClient. Spring Boot has a logging [common application property](https://docs.spring.io/spring-boot/docs/current/reference/html/common-application-properties.html) for doing this sort of thing, just add this to your application properties

```java
logging.level.org.apache.http.wire=DEBUG
```

### 90.8.1 How can I debug the mapping/request/response being sent by WireMock?

Starting from version  `1.2.0`  we turn on WireMock logging to info and the WireMock notifier to being verbose. Now you will exactly know what request was received by WireMock server and which matching response definition was picked.

To turn off this feature just bump WireMock logging to  `ERROR` 

```java
logging.level.com.github.tomakehurst.wiremock=ERROR
```

### 90.8.2 How can I see what got registered in the HTTP server stub?

You can use the  `mappingsOutputFolder`  property on  `@AutoConfigureStubRunner`  or  `StubRunnerRule`  to dump all mappings per artifact id. Also the port at which the given stub server was started will be attached.

### 90.8.3 Can I reference text from file?

Yes! With version 1.2.0 we’ve added such a possibility. It’s enough to call  `file(…)`  method in the DSL and provide a path relative to where the contract lays. If you’re using YAML just use the  `bodyFromFile`  property.

