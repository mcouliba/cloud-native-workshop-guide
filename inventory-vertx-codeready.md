## Enterprise Microservices with Eclipse Vert.x

In this lab you will learn about building microservices using Vert.x.

![CoolStore Architecture]({% image_path coolstore-arch-inventory-vertx.png %}){:width="200px"}

#### What is Eclipse Vert.x?

[Eclipse Vert.x](http://vertx.io) is a toolkit for building reactive applications on the Java Virtual Machine (JVM). Vert.x does not 
impose a specific framework or packaging model and can be used within your existing applications and frameworks 
in order to add reactive functionality by just adding the Vert.x jar files to the application classpath.

Vert.x enables building reactive systems as defined by [The Reactive Manifesto](http://www.reactivemanifesto.org) and build 
services that are:

* *Responsive*: to handle requests in a reasonable time
* *Resilient*: to stay responsive in the face of failures
* *Elastic*: to stay responsive under various loads and be able to scale up and down
* *Message driven*: components interact using asynchronous message-passing

Vert.x is designed to be event-driven and non-blocking. Events are delivered in an event loop that must never be blocked. Unlike traditional applications, Vert.x uses a very small number of threads responsible for dispatching the events to event handlers. If the event loop is blocked, the events won’t be delivered anymore and therefore the code needs to be mindful of this execution model.

#### Vert.x Maven Project 

The `inventory-vertx` project has the following structure which shows the components of 
the Vert.x project laid out in different subdirectories according to Maven best practices:

***TODO: REPLACE THIS SCREENSHOT WITH* `inventory-vertx` *SCREENSHOT***

![Inventory Project]({% image_path thorntail-inventory-project.png %}){:width="200px"}

This is a minimal Vert.x project with support for RESTful services and relational database access.
This project currently contains no code other than the main class, ***InventoryVerticle.java*** which is there to bootstrap
the Vert.x application. Verticles are encapsulated parts of the application that can run completely independently and
communicate with each other using HTTP. Verticles get deployed and run by Vert.x in an event loop and therefore it 
is important that the code in a Verticle does not block. This asynchronous architecture allows Vert.x applications 
to easily scale and handle large amounts of throughput with few threads. All API calls in Vert.x by default are non-blocking 
and support this concurrency model.

![Vert.x Event Loop]({% image_path vertx-event-loop.png %}){:width="600px"}

Although you can have multiple, there is currently only one Verticle created in the `inventory-vertx` project. 

Examine `com.redhat.cloudnative.inventory.InventoryVerticle` in the `src/main` directory:

~~~java
package com.redhat.cloudnative.inventory;

import io.reactivex.Completable;
import io.reactivex.Single;
import io.vertx.reactivex.config.ConfigRetriever;
import io.vertx.config.ConfigRetrieverOptions;
import io.vertx.config.ConfigStoreOptions;
import io.vertx.core.json.JsonObject;
import io.vertx.reactivex.core.http.HttpServer;
import io.vertx.reactivex.ext.jdbc.JDBCClient;
import io.vertx.ext.sql.ResultSet;
import io.vertx.reactivex.ext.web.Router;
import io.vertx.reactivex.core.AbstractVerticle;
import io.vertx.reactivex.ext.web.handler.StaticHandler;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class InventoryVerticle extends AbstractVerticle {

  private static final Logger LOG = LoggerFactory.getLogger(InventoryVerticle.class);

  private JDBCClient inventoryClient;

  @Override
  public Completable rxStart() {

    Router router = Router.router(vertx);
    router.get("/*").handler(StaticHandler.create("assets"));
    router.get("/health").handler(ctx -> ctx.response().end(new JsonObject().put("status", "UP").toString()));

    Single<HttpServer> serverSingle = vertx.createHttpServer()
        .requestHandler(router)
        .rxListen(Integer.getInteger("http.port", 8080));

    ConfigRetrieverOptions configOptions = new ConfigRetrieverOptions()
        .addStore(new ConfigStoreOptions()
            .setType("file")
            .setFormat("yaml")
            .setConfig(new JsonObject()
                .put("path", "config/app-config.yml")));
    ConfigRetriever retriever = ConfigRetriever.create(vertx, configOptions);
    Single<JsonObject> s = retriever.rxGetConfig();

    return s
        .flatMap(this::populateDatabase)
        .flatMap(rs -> serverSingle)
        .ignoreElement();
  }

  private Single<ResultSet> populateDatabase(JsonObject config) {
    LOG.info("Will use database " + config.getValue("jdbcUrl"));
    inventoryClient = JDBCClient.createNonShared(vertx, config);
    String sql = "" +
        "drop table if exists INVENTORY;" +
        "create table \"INVENTORY\" (\"ITEMID\" varchar(32) PRIMARY KEY, \"QUANTITY\" int);" +
        "insert into \"INVENTORY\" (\"ITEMID\", \"QUANTITY\") values (329299, 35);" +
        "insert into \"INVENTORY\" (\"ITEMID\", \"QUANTITY\") values (329199, 12);" +
        "insert into \"INVENTORY\" (\"ITEMID\", \"QUANTITY\") values (165613, 45);" +
        "insert into \"INVENTORY\" (\"ITEMID\", \"QUANTITY\") values (165614, 87);" +
        "insert into \"INVENTORY\" (\"ITEMID\", \"QUANTITY\") values (165954, 43);" +
        "insert into \"INVENTORY\" (\"ITEMID\", \"QUANTITY\") values (444434, 32);" +
        "insert into \"INVENTORY\" (\"ITEMID\", \"QUANTITY\") values (444435, 53);";
    return inventoryClient.rxQuery(sql);
  }
}
~~~

Here is what happens in the above code:

1. A Verticle is created by extending from ***AbstractVerticle*** class
2. ***Router*** is retrieved for mapping the REST endpoints
3. A REST endpoint is created for **/** to return a static HTML page **assets/index.html**
4. The database client configuration is loaded from the file `config/app-config.yml`
5. The database is populated with arbitrary data by ***populateDatabase()***
6. An HTTP Server is created which listens on port **8080**
7. These operations get actually sequentially executed using an RxJava `Single` composition `flatMap`

Vert.x provides [built-in service configuration](https://vertx.io/docs/vertx-config/java/) 
for configured services. Vert.x service config can be seamlessly integrated with external 
service configuration mechanisms provided by OpenShift, Kubernetes, Consul, Redis, etc.

In this case we will simply load the configuration file from the file system.

You can use Maven to make sure the skeleton project builds successfully. You should get a **BUILD SUCCESS** message 
in the build logs, otherwise the build has failed.

In CodeReady Workspaces, `right click on inventory-vertx` project in the project explorer then, `click on Commands > Build > build`

![Maven Build]({% image_path codeready-commands-build.png %}){:width="600px"}

Once successfully built, the resulting ***jar*** is located in the **target/** directory:

~~~shell
$ ls labs/inventory-vertx/target/*.jar

labs/inventory-vertx/target/inventory-1.0-SNAPSHOT.jar
~~~

This is an uber-jar with all the dependencies required packaged in the *jar* to enable running the 
application with `java -jar`.

Now let's write some code and expose the databae as RESTful endpoint to create the Inventory service:

#### Exposing the database as a RESTful Service

Replace the content of ***src/main/java/com/redhat/cloudnative/inventory/InventoryVerticle.java*** class with the following:

~~~java
package com.redhat.cloudnative.inventory;

import io.reactivex.Completable;
import io.reactivex.Single;
import io.vertx.reactivex.config.ConfigRetriever;
import io.vertx.config.ConfigRetrieverOptions;
import io.vertx.config.ConfigStoreOptions;
import io.vertx.core.json.JsonArray;
import io.vertx.core.json.JsonObject;
import io.vertx.reactivex.core.http.HttpServer;
import io.vertx.reactivex.ext.jdbc.JDBCClient;
import io.vertx.ext.sql.ResultSet;
import io.vertx.reactivex.ext.web.Router;
import io.vertx.reactivex.core.AbstractVerticle;
import io.vertx.reactivex.ext.web.RoutingContext;
import io.vertx.reactivex.ext.web.handler.StaticHandler;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.List;

public class InventoryVerticle extends AbstractVerticle {

  private static final Logger LOG = LoggerFactory.getLogger(InventoryVerticle.class);

  private JDBCClient inventoryClient;

  @Override
  public Completable rxStart() {

    Router router = Router.router(vertx);
    router.get("/*").handler(StaticHandler.create("assets"));
    router.get("/health").handler(ctx -> ctx.response().end(new JsonObject().put("status", "UP").toString()));
    router.get("/api/inventory/:itemId").handler(this::findQuantity);

    Single<HttpServer> serverSingle = vertx.createHttpServer()
        .requestHandler(router)
        .rxListen(Integer.getInteger("http.port", 8080));

    ConfigRetrieverOptions configOptions = new ConfigRetrieverOptions()
        .addStore(new ConfigStoreOptions()
            .setType("file")
            .setFormat("yaml")
            .setConfig(new JsonObject()
                .put("path", "config/app-config.yml")));
    ConfigRetriever retriever = ConfigRetriever.create(vertx, configOptions);
    Single<JsonObject> s = retriever.rxGetConfig();

    return s
        .flatMap(this::populateDatabase)
        .flatMap(rs -> serverSingle)
        .ignoreElement();
  }

  private void findQuantity(RoutingContext rc) {
    String itemId = rc.pathParam("itemId");
    inventoryClient.queryWithParams(
        "select \"QUANTITY\" from \"INVENTORY\" where \"ITEMID\"=?",
        new JsonArray().add(itemId),
        ar -> {
          if (ar.succeeded()) {
            ResultSet resultSet = ar.result();
            List<JsonObject> rows = resultSet.getRows();
            if (rows.size() == 1) {
              int quantity = rows.get(0).getInteger("QUANTITY");
              JsonObject body = new JsonObject()
                  .put("quantity", quantity)
                  .put("itemId", itemId);
              rc.response()
                  .putHeader("content-type", "application/json")
                  .end(body.encodePrettily());
            } else {
              rc.response().setStatusCode(404).end("Product " + itemId + " not found");
            }
          } else {
            LOG.error("Could not access database", ar);
            rc.fail(500, ar.cause());
          }
        });
  }

  private Single<ResultSet> populateDatabase(JsonObject config) {
    LOG.info("Will use database " + config.getValue("jdbcUrl"));
    inventoryClient = JDBCClient.createNonShared(vertx, config);
    String sql = "" +
        "drop table if exists INVENTORY;" +
        "create table \"INVENTORY\" (\"ITEMID\" varchar(32) PRIMARY KEY, \"QUANTITY\" int);" +
        "insert into \"INVENTORY\" (\"ITEMID\", \"QUANTITY\") values (329299, 35);" +
        "insert into \"INVENTORY\" (\"ITEMID\", \"QUANTITY\") values (329199, 12);" +
        "insert into \"INVENTORY\" (\"ITEMID\", \"QUANTITY\") values (165613, 45);" +
        "insert into \"INVENTORY\" (\"ITEMID\", \"QUANTITY\") values (165614, 87);" +
        "insert into \"INVENTORY\" (\"ITEMID\", \"QUANTITY\") values (165954, 43);" +
        "insert into \"INVENTORY\" (\"ITEMID\", \"QUANTITY\") values (444434, 32);" +
        "insert into \"INVENTORY\" (\"ITEMID\", \"QUANTITY\") values (444435, 53);";
    return inventoryClient.rxQuery(sql);
  }
}
~~~

You don't need to press a save button! CodeReady Workspaces automatically saves the changes made to the files.

Let's break down what happens in the above code.

~~~java
router.get("/api/inventory/:itemId").handler(this::findQuantity);
~~~

Vert.x uses Vert.x Web for building REST services. The HTTP router gets an extra route that adds a REST mapping to
map **/api/inventory** to ***findQuantity()***. 

The above REST service defines an endpoint that is accessible via `HTTP GET` at  for example `/api/inventory/329299` with 
the last path param being the product id which we want to check its inventory status.

Review ***findQuantity()*** and note how the database is accessed asynchronously and the result is actually
sent back to the client using the routing context provided by Vert.x Web.

Examine `inventory-vertx/src/main/resources/config/app-config.yml` to see the database connection details. 
An in-memory H2 database is used in this lab for local development and in the following 
labs will be replaced with a PostgreSQL database. Be patient! More on that later.

Build and package the ***Inventory Service*** using Maven by `right clicking on inventory-vertx` project in the project explorer then, `click on Commands > Build > build`

![Maven Build]({% image_path codeready-commands-build.png %}){:width="600px"}

> Make sure **inventory-vertx** project is highlighted in the project explorer

Using CodeReady Workspaces, you can conveniently run the application
directly in the IDE and test it before deploying it on OpenShift.

In CodeReady Workspaces, click on the run icon and then on **run vertx**.

> You can also run the inventory service in CodeReady Workspaces using the commands palette and then **run > run vertx**

![Run Palette]({% image_path thorntail-inventory-codeready-run-palette.png %}){:width="800px"}

Once you see `INFO: Succeeded in deploying verticle` in the logs, the Inventory service is up and running and you can access the
inventory REST API. Let’s test it out using `curl` in the **Terminal** window:

~~~shell
$ curl http://localhost:9001/api/inventory/329299

{"itemId":"329299","quantity":35}
~~~

You can also use the preview url that CodeReady Workspaces has generated for you to be able to test service 
directly in the browser. Append the path `/api/inventory/329299` at the end of the preview url and try 
it in your browser in a new tab.

![Preview URL]({% image_path thorntail-inventory-codeready-preview-url.png %}){:width="900px"}

![Preview URL]({% image_path wfswarm-inventory-che-preview-browser.png %}){:width="900px"}

The REST API returned a JSON object representing the inventory count for this product. Congratulations!

#### Deploy Vert.x on OpenShift

It’s time to build and deploy our service on OpenShift. 

OpenShift [Source-to-Image (S2I)]({{OPENSHIFT_DOCS_BASE}}/architecture/core_concepts/builds_and_image_streams.html#source-build) 
feature can be used to build a container image from your project. OpenShift 
S2I uses the [supported OpenJDK container image](https://access.redhat.com/documentation/en-us/red_hat_jboss_middleware_for_openshift/3/html/red_hat_java_s2i_for_openshift) to build the final container 
image of the Inventory service by uploading the Vert.x uber-jar from 
the **target** folder to the OpenShift platform. 

Maven projects can use the [Fabric8 Maven Plugin](https://maven.fabric8.io) in order to use OpenShift S2I for building 
the container image of the application from within the project. This maven plugin is a Kubernetes/OpenShift client 
able to communicate with the OpenShift platform using the REST endpoints in order to issue the commands 
allowing to build a project, deploy it and finally launch a docker process as a pod.

To build and deploy the **Inventory Service** on OpenShift using the *fabric8* maven plugin, 
which is already configured in CodeReady Workspaces, `right click on inventory-vertx` project in the project explorer then, `click on Commands > Deploy > fabric8:deploy`

![Fabric8 Deploy]({% image_path codeready-commands-deploy.png %}){:width="600px"}

`fabric8:deploy` will cause the following to happen:

* The Inventory uber-jar is built using Vert.x
* A container image is built on OpenShift containing the Inventory uber-jar and JDK
* All necessary objects are created within the OpenShift project to deploy the Inventory service

Once this completes, your project should be up and running. OpenShift runs the different components of 
the project in one or more pods which are the unit of runtime deployment and consists of the running 
containers for the project. 

Let's take a moment and review the OpenShift resources that are created for the Inventory REST API:

* **Build Config**: `inventory-s2i` build config is the configuration for building the Inventory 
container image from the inventory source code or JAR archive
* **Image Stream**: `inventory` image stream is the virtual view of all inventory container 
images built and pushed to the OpenShift integrated registry.
* **Deployment Config**: `inventory` deployment config deploys and redeploys the Inventory container 
image whenever a new Inventory container image becomes available
* **Service**: `inventory` service is an internal load balancer which identifies a set of 
pods (containers) in order to proxy the connections it receives to them. Backing pods can be 
added to or removed from a service arbitrarily while the service remains consistently available, 
enabling anything that depends on the service to refer to it at a consistent address (service name 
or IP).
* **Route**: `inventory` route registers the service on the built-in external load-balancer 
and assigns a public DNS name to it so that it can be reached from outside OpenShift cluster.

You can review the above resources in the OpenShift Web Console or using `oc describe` command:

> `bc` is the short-form of `buildconfig` and can be interchangeably used 
> instead of it with the OpenShift CLI. The same goes for `is` instead 
> of `imagestream`, `dc` instead of `deploymentconfig` and `svc` instead of `service`.

~~~shell
$ oc describe bc inventory-s2i
$ oc describe is inventory
$ oc describe dc inventory
$ oc describe svc inventory
$ oc describe route inventory
~~~

You can see the exposed DNS url for the Inventory service in the OpenShift Web Console or using 
OpenShift CLI:

~~~shell
$ oc get routes

NAME        HOST/PORT                                        PATH       SERVICES  PORT  TERMINATION   
inventory   inventory-{{COOLSTORE_PROJECT}}.{{APPS_HOSTNAME_SUFFIX}}   inventory  8080            None
~~~

Copy the route url for the Inventory service and verify the API Gateway service works using `curl`:

> The route urls in your project would be different from the ones in this lab guide! Use the one from yor project.

~~~shell
$ curl http://{{INVENTORY_ROUTE_HOST}}/api/inventory/329299

{"itemId":"329299","quantity":35}
~~~

Well done! You are ready to move on to the next lab.
