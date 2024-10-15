---
title: 'Building the Neo4j Matrix: Spring Boot, Reactive APIs, and Graph Databases'
description: 'Lorem ipsum dolor sit amet'
pubDate: 'Sep 24 2024'
heroImage: '/springneo4j.jpg'
---

Imagine the following: tech lead asks you to come up ASAP with a project boilerplate including:
1. Latest Spring boot version
2. Cutting-edge graph database support
3. reactive CRUD API
4. Database seeder for your dev environment

> Whatcha gonna do?

---

## 1. *Spring* into action

With the power of our favourite http client we swiftly fetch a ready-to-go project structure from the beautiful Spring starter API just like that:

```bash
curl https://start.spring.io/starter.tgz \
  -d dependencies=webflux,data-neo4j,lombok \
  -d type=gradle-project \
  -d JavaVersion=21 \
  -d bootVersion=3.3.4 \
  -d baseDir=matrix \
  -d groupId=org.thematrix \
  -d artifactId=library \
  -d name=Library | tar -xzvf -
```

Pretty neat, huh? Let's enter the matrix:

```bash
cd matrix
```

---

## 2. "I know why you're here, Neo" 😎

We'll use a neo4j service with data volume binding and expose the database server default ports to our host network.

```yml
services:
	neo4j:
		image: "neo4j:5.23"
		environment:
			NEO4J_AUTH: $DB_USER/$DB_PWD # `none` to disable
		volumes:
			- $HOME/.neo4j/data:/data
			- $HOME/.neo4j/conf:/var/lib/neo4j/conf
		ports:
			- 7474:7474 # ui
			- 7687:7687 # bolt
```


**Bonus**: Having the database configuration files (`conf/neo4j.conf`) at hand is mighty useful; it lets us e.g. optimize for read/write resp., configure transaction behaviour, disable unwanted background services etc.

```ini
// Incoming defaults
server.memory.pagecache.size=512M

server.default_listen_address=0.0.0.0
server.directories.logs=/logs

// other defaults
db.transaction.concurrent.maximum=1000
db.transaction.timeout=0

// add this
dbms.usage_report.enabled=false
```

---

### Bridging the gap

Nice, all up and running! Now we'll need to tell our Spring boot application in `src/main/resources/application.properties` how to connect to the database system:

```ini
spring.neo4j.uri=bolt://${DB_URL}:7687
spring.neo4j.authentication.username=${DB_USER}
spring.neo4j.authentication.password=${DB_PWD}
```

Make sure we have auth data to work with.

```bash
echo "DB_URL=localhost\nDB_USER=neo4j\nDB_PWD=password" > .env
```

#### Spilling the Beans

Before we get to play with our DTOs, we'll need to define a couple of database config beans (injected singleton dependencies) to the [Spring IoC Container](https://docs.spring.io/spring-framework/reference/core/beans/introduction.html). We'll have a separate `AppConfig` class to hold 'em:

```java
@org.springframework.context.annotation.Configuration
@EnableReactiveNeo4jRepositories(basePackages = "repos")
public class AppConfig {
	// all our beans (see next 2 sections)
}
```

##### Transaction Manager

First, we register the `TransactionManager` database middleware. *Spring Data Neo4J* offers a tidy abstraction that takes care of all [transaction management methods](https://docs.spring.io/spring-data/neo4j/docs/current/api/org/springframework/data/neo4j/core/transaction/ReactiveNeo4jTransactionManager.html#method-summary); imperative to pass the `reactiveTransactionManager`  name for our future reactive CRUD service to find it.

```java
	@Bean(name = "reactiveTransactionManager")
	public ReactiveNeo4jTransactionManager reactiveTransactionManager(Driver driver,
	ReactiveDatabaseSelectionProvider dbNameProvider) {
		return new ReactiveNeo4jTransactionManager(driver, dbNameProvider);
	}
```

We'll come back to the reactivity concept later on in this post.

##### Putting Cypher in place

> “You know, I know this steak doesn't exist. I know that when I put it in my mouth, the Matrix is telling my brain that it is juicy and delicious.” -- Cypher

```java
	@Bean
	public Configuration cypherDslConfiguration() {
		return org.neo4j.cypherdsl.core.renderer.Configuration.newConfig()
									.withDialect(Dialect.NEO4J_5).build();
	}
```

With the current *Spring Data Neo4J* version (`7.3.4`), [CypherDSL will fall back to an older dialect that will, among other things, use the deprecated `Long Id` field instead of the superseding `String elementId`](https://community.neo4j.com/t/unable-to-create-a-relationship-with-save-method-of-neo4jrepository/62773/3). We'll need to tell SPN here explicitly to [use the latest dialect](https://github.com/spring-projects/spring-data-neo4j/blob/039ee2694326adfa2a880b762de13a3749d702f6/src/main/java/org/springframework/data/neo4j/core/mapping/SpringDataCypherDsl.java#L39) compatible with the Neo4J version `>5` that we're running.

---

### Models

Alright, now that we should have a running database service with a working application interface, we can start getting creative with our data layer primitives.

Let's have e.g. a `Book` and an `Author` model in `src/main/java/db/model/*`

```java
@Data
@Node
public class Book {
	@Id
	@GeneratedValue
	private String id;

	private final String title;
	private final String description;

	@Relationship
	private Author author;
}
```

`@Node` binds our `Book` POJO to a corresponding entity (aka node) in our graph data model. `@Node` is conceptually the same as `@Entity` if you're coming from the [JPA paradigm](https://jakarta.ee/learn/docs/jakartaee-tutorial/current/persist/persistence-intro/persistence-intro.html#_overview_of_jakarta_persistence).

`@Relationship` lets us access connected nodes in our NoSQL graph model via relation properties.

```java
@Data
@Node
public class Author {
	@Id
	@GeneratedValue
	private String id;

	private final String fullName;
}
```

`@Data` comes with [Lombok](https://projectlombok.org/). Lombok is awesome: it delves under the hood for us and creates a constructor, getter & setter, hashCode functions etc. for our DTOs. All it takes is literally: `@Data`


![hunlimited powah](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/n0lj74701urgr0fp6544.gif)

Note that if we didn't define our fields as `final` vars, Lombok 'd create a constructor for ALL fields by default, which in case of a database-managed `ID` field wouldn't necessarily be a desirable thing...

___

## 3. Reactive CRUD Repo

We'll use a repository to manage all our `Book` model interaction methods (i.e. `find(All)?`, `save(All)?`, `delete(All)?` etc.) for us. Since we're mainly concerned with authors in terms of `Book` properties, we'll spare ourselves a separate `Author` repository.

```java
public interface BookRepository extends ReactiveNeo4jRepository<Book, String> {
	// add your custom CRUD methods
}
```

---

### A word about reactivity

A reactive API relies on non-blocking callback-event-based IO-multiplexing. Now why would we want to have that?
Optimized resource consumption: not to waste a whole thread on handling a single, potentially unreliable (e.g. high latency) connection synchronously (with a lot of requests queueing up).

[Netty NIO framework](https://netty.io/) is a popular implementation that forms -- along with the [Reactive Streams](https://github.com/reactive-streams/reactive-streams-jvm) publish-subscribe callback specification -- the basis for the [Reactor](https://github.com/reactor/reactor-core/) high-level abstraction that comes with the Spring Framework.

With Reactor's [`Flux`](https://github.com/reactor/reactor-core/blob/c910883bdf22726b7aeedcb6a831ca838de02114/reactor-core/src/main/java/reactor/core/publisher/Flux.java#L126) and [`Mono`](https://github.com/reactor/reactor-core/blob/c910883bdf22726b7aeedcb6a831ca838de02114/reactor-core/src/main/java/reactor/core/publisher/Mono.java#L121) types as [`Publisher`](https://github.com/reactive-streams/reactive-streams-jvm/blob/master/api/src/main/java/org/reactivestreams/Publisher.java) implementations, sticking together asynchronous data processing pipelines is plug and play. But this comes at the expense that we cannot use it in the conventional imperative way as we would with synchronous blocking IO op procedure calls.

---

## 4. Sticking everything together

Great! Our `src/main/java/*` packages so far:

```sh
.
├── db
│   ├── DatabaseSeeder.java❗
│   └── model
|	    ├── Author.java ✅
│       └── Book.java   ✅
├── org
│   └── thematrix
|	    └── library
|		    ├── AppConfig.java ✅
|		    └── LibraryApplication.java ✅
└── repos
	└── BookRepository.java ✅
```


Let's register our DatabaseSeeder class as a service provider to the Spring Container and autowire the BookRepository property to be automatically injected upon service start.

We'll have two methods `down` and `seed` attached to a context refresh event listener. `down` and `seed` will fire in the order of their definition once the application context starts or refreshes. Note that, to satisfy reactivity, `stock.deleteAll()` shall not be used imperatively since it merely registers itself as a callback event publisher to our reactive `BookRepository`.

```java
@Profile("dev")
@Service
public class DatabaseSeeder {

	@Autowired
	private final BookRepository stock = null;

	@EventListener
	public Mono<Void> down(ContextRefreshedEvent event) {
		return stock.deleteAll();
	}

	@EventListener
	public Flux<Book> seed(ContextRefreshedEvent event) {
		Faker fake = new Faker();

		List<Book> shelf = new ArrayList<Book>();

		for (int i = 0; i < 5; i++) {
			Book book = new Book(fake.book.title(), fake.lorem.paragraph());
			book.setAuthor(new Author(fake.artist.name()));
			shelf.add(book);
		}

		return stock.saveAll(shelf);

	}
}
```

Make sure we only seed in `dev` environment.

```ini
spring.active.profile=dev
```


Now to add the necessary `AppConfig` and package paths to the application entrypoint in our main method:

```java
@SpringBootApplication(scanBasePackages = { "db", "db.model" })
@Import(AppConfig.class)
public class LibraryApplication {

	public static void main(String[] args) {
		SpringApplication.run(LibraryApplication.class, args);
	}
}
```


We're all set! If we now run our app, the database seeder service will automatically write dummy data to the database. Mission accomplished. Tech Lead's proud of you. And this is merely where the fun begins.

---

## Further Reading

### Spring boot boilerplate
https://docs.spring.io/spring-data/neo4j/reference/getting-started.html#create-spring-boot-project
https://start.spring.io/
https://github.com/spring-projects/spring-data-neo4j/blob/main/README.adoc#getting-started
https://docs.spring.io/spring-framework/docs/4.2.x/spring-framework-reference/html/transaction.html

### Neo4J
https://neo4j.com/docs/operations-manual/current/docker/introduction/
https://hub.docker.com/_/neo4j
https://neo4j.com/docs/operations-manual/current/configuration/configuration-settings/#config_dbms.usage_report.enabled
https://neo4j.com/docs/operations-manual/current/database-internals/transaction-management/
https://community.neo4j.com/t/unable-to-create-a-relationship-with-save-method-of-neo4jrepository/62773/3
https://github.com/spring-projects/spring-data-neo4j/issues/2848

### Reactivity
https://developer.okta.com/blog/2018/09/21/reactive-programming-with-spring
https://www.reactive-streams.org/
https://github.com/reactor/reactor-core/
https://github.com/reactor/reactor-netty/
https://www.reactivemanifesto.org/
