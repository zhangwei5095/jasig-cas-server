---
layout: default
title: CAS - JPA Ticket Registry
---


# JPA Ticket Registry
The JPA Ticket Registry allows CAS to store client authenticated state data (tickets) in a database back-end such as MySQL. 

<div class="alert alert-warning"><strong>Usage Warning!</strong><p>Using a RDBMS as the back-end persistence choice for Ticket Registry state management is a fairly unnecessary and complicated process. Ticket registries generally do not need the durability that comes with RDBMS and unless you are already outfitted with clustered RDBMS technology and the resources to manage it, the complexity is likely not worth the trouble. Given the proliferation of hardware virtualization and the redundancy and vertical scaling they often provide, more suitable recommendation would be the default in-memory ticket registry for a single node CAS deployment and distributed cache-based registries for higher availability.</p></div>



# Configuration

- Adjust the `src/main/webapp/WEB-INF/spring-configuration/ticketRegistry.xml` with the following:

{% highlight xml %}
<bean id="ticketRegistry" class="org.jasig.cas.ticket.registry.JpaTicketRegistry" />

<bean class="org.springframework.orm.jpa.support.PersistenceAnnotationBeanPostProcessor"/>

<bean id="entityManagerFactory" class="org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean">
	<property name="dataSource" ref="dataSource"/>
	<property name="jpaVendorAdapter">
	  <bean class="org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter">
	    <property name="generateDdl" value="true"/>
	    <property name="showSql" value="true" />
	  </bean>
	</property>
	<property name="jpaProperties">
	  <props>
	    <prop key="hibernate.hbm2ddl.auto">update</prop>
	  </props>
	</property>
</bean>

<bean id="transactionManager" class="org.springframework.orm.jpa.JpaTransactionManager"
		p:entityManagerFactory-ref="entityManagerFactory" />

<tx:annotation-driven transaction-manager="transactionManager" />

<bean id="ticketRegistryCleaner" class="org.jasig.cas.ticket.registry.support.DefaultTicketRegistryCleaner"
		p:ticketRegistry-ref="ticketRegistry"
		p:lock-ref="cleanerLock" />

<bean id="cleanerLock" class="org.jasig.cas.ticket.registry.support.JpaLockingStrategy"
		p:uniqueId="${host.name}"
		p:applicationId="cas-ticket-registry-cleaner" />
 
{% endhighlight %}
The above snippet assumes that data source information and connection details are defined.

- Configure other JPA dependencies:

In the `pom.xml` file of the Maven overlay, adjust for the following dependencies:

{% highlight xml %}
...
<dependency>
	<groupId>org.hibernate</groupId>
	<artifactId>hibernate-core</artifactId>
	<version>${hibernate.core.version}</version>
	<scope>runtime</scope>
</dependency>

<dependency>
	<groupId>org.hibernate</groupId>
	<artifactId>hibernate-entitymanager</artifactId>
	<version>${hibernate.core.version}</version>
	<scope>runtime</scope>
</dependency>
...
{% endhighlight %}


# Database Configuration


## JDBC Driver
CAS must have access to the appropriate JDBC driver for the database. Once you have obtained the appropriate driver and configured the data source, place the JAR inside the lib directory of your web server environment (i.e. `$TOMCAT_HOME/lib`)


## Schema
If the user has sufficient privileges on start up, the database tables should be created. The database user MUST have `CREATE/ALTER` privileges to take advantage of automatic schema generation and schema updates.


## Deadlocks
The Hibernate SchemaExport DDL creation tool *may* fail to create two very import indices when generating the ticket tables. The absence of these indices dramatically increases the potential for database deadlocks under load.
If the indices were not created you should manually create them before placing your CAS configuration into a production environment.

To review indices, you may use the following MYSQL-based sample code below:

{% highlight sql %}
show index from SERVICETICKET where column_name='ticketGrantingTicket_ID';
show index from TICKETGRANTINGTICKET where column_name='ticketGrantingTicket_ID';
{% endhighlight %}

To create indices that are missing, you may use the following sample code below:


### MYSQL
{% highlight sql %}
CREATE INDEX ST_TGT_FK_I ON SERVICETICKET (ticketGrantingTicket_ID);
CREATE INDEX ST_TGT_FK_I ON TICKETGRANTINGTICKET (ticketGrantingTicket_ID);
{% endhighlight %}


###ORACLE
{% highlight sql %}
CREATE INDEX "ST_TGT_FK_I"
  ON SERVICETICKET ("TICKETGRANTINGTICKET_ID")
  COMPUTE STATISTICS;
 
/** Create index on TGT self-referential foreign-key constraint */
CREATE INDEX "TGT_TGT_FK_I"
  ON TICKETGRANTINGTICKET ("TICKETGRANTINGTICKET_ID")
  COMPUTE STATISTICS;
{% endhighlight %}


## Ticket Cleanup

The use `JpaLockingStrategy` is strongly recommended for HA environments where multiple nodes are attempting ticket cleanup on a shared database. `JpaLockingStrategy` can auto-generate the schema for the target platform.  A representative schema is provided below that applies to PostgreSQL:


{% highlight sql %}
CREATE TABLE locks (
 application_id VARCHAR(50) NOT NULL,
 unique_id VARCHAR(50) NULL,
 expiration_date TIMESTAMP NULL
);

ALTER TABLE locks ADD CONSTRAINT pk_locks
 PRIMARY KEY (application_id);
{% endhighlight %}

<div class="alert alert-warning"><strong>Platform-Specific Issues</strong><p>The exact DDL to create the LOCKS table may differ from the above. For example, on Oracle platforms the `expiration_date` column must be of type `DAT`E.  Use the `JpaLockingStrategy` which can create and update the schema automatically to avoid platform-specific schema issues.</p></div>


## Connection Pooling

It is ***strongly*** recommended that database connection pooling be used in a production environment. The following configuration makes use of the c3p0 connection pooling library, which would replace the `dataSource` bean found in the `ticketRegistry.xml` example above:

{% highlight xml %}
...
<bean
  id="dataSource"
  class="com.mchange.v2.c3p0.ComboPooledDataSource"
  p:driverClass="${database.driverClass}"
  p:jdbcUrl="${database.url}"
  p:user="${database.user}"
  p:password="${database.password}"
  p:initialPoolSize="${database.pool.minSize}"
  p:minPoolSize="${database.pool.minSize}"
  p:maxPoolSize="${database.pool.maxSize}"
  p:maxIdleTimeExcessConnections="${database.pool.maxIdleTime}"
  p:checkoutTimeout="${database.pool.maxWait}"
  p:acquireIncrement="${database.pool.acquireIncrement}"
  p:acquireRetryAttempts="${database.pool.acquireRetryAttempts}"
  p:acquireRetryDelay="${database.pool.acquireRetryDelay}"
  p:idleConnectionTestPeriod="${database.pool.idleConnectionTestPeriod}"
  p:preferredTestQuery="${database.pool.connectionHealthQuery}"
  />
...
{% endhighlight %}

The following pool configuration parameters are provided for information only and may serve as a reasonable starting point for configuring a production database connection pool. 

<div class="alert alert-info"><strong>Usage Tip</strong><p>Note the health check query is specific to PostgreSQL.</p></div>

{% highlight bash %}
# == Basic database connection pool configuration ==
database.dialect=org.hibernate.dialect.PostgreSQLDialect
database.driverClass=org.postgresql.Driver
database.url=jdbc:postgresql://somehost.vt.edu/cas?ssl=true
database.user=somebody
database.password=meaningless
database.pool.minSize=6
database.pool.maxSize=18
 
# Maximum amount of time to wait in ms for a connection to become
# available when the pool is exhausted
database.pool.maxWait=10000
 
# Amount of time in seconds after which idle connections
# in excess of minimum size are pruned.
database.pool.maxIdleTime=120
 
# Number of connections to obtain on pool exhaustion condition.
# The maximum pool size is always respected when acquiring
# new connections.
database.pool.acquireIncrement=6
 
# == Connection testing settings ==
 
# Period in s at which a health query will be issued on idle
# connections to determine connection liveliness.
database.pool.idleConnectionTestPeriod=30
 
# Query executed periodically to test health
database.pool.connectionHealthQuery=select 1
 
# == Database recovery settings ==
 
# Number of times to retry acquiring a _new_ connection
# when an error is encountered during acquisition.
database.pool.acquireRetryAttempts=5
 
# Amount of time in ms to wait between successive aquire retry attempts.
database.pool.acquireRetryDelay=2000
{% endhighlight %}

The following maven dependency for the library must be included in your Maven overlay:
{% highlight xml %}
<dependency>
	<groupId>c3p0</groupId>
	<artifactId>c3p0</artifactId>
	<version>${c3p0.version}</version>
	<scope>runtime</scope>
</dependency>
{% endhighlight %}


# Platform Considerations

## MySQL

### Use InnoDB Tables
The use of InnoDB tables is strongly recommended for the MySQL platform for a couple reasons:

- InnoDB provides referential integrity that is helpful for preventing orphaned records in ticket tables.
- Provides better locking semantics (e.g. support for SELECT ... FOR UPDATE) than the default MyISAM table type.

InnoDB tables are easily specified via the use of the following Hibernate dialect:

{% highlight xml %}
<prop key="hibernate.dialect">org.hibernate.dialect.MySQLInnoDBDialect</prop>
 
<!-- OR for MySQL 5.x use the following instead -->
<prop key="hibernate.dialect">org.hibernate.dialect.MySQL5InnoDBDialect</prop>
{% endhighlight %}


###BLOB vs LONGBLOB
Hibernate on recent versions of MySQL (e.g. 5.1) properly maps the `@Lob` JPA annotation onto type `LONGBLOB`, which is very important since these fields commonly store serialized graphs of Java objects that grow proportionally with CAS SSO session lifetime. Under some circumstances, Hibernate may treat these columns as type `BLOB`, which have storage limits that are easily exceeded. It is recommended that the generated schema be reviewed and any BLOB type columns be converted to `LONGBLOB`.

The following MySQL statement would change this `SERVICES_GRANTED_ACCESS_TO` column's type to `LONGBLOB`:

{% highlight sql %}
ALTER TABLE TICKETGRANTINGTICKET MODIFY SERVICES_GRANTED_ACCESS_TO LONGBLOB;
{% endhighlight %}


###Case Sensitive Schema
It may necessary to force lowercase schema names in the MySQL configuration:

Adjust the `my.cnf` file to include the following:
{% highlight bash %}
lower-case-table-names = 1
{% endhighlight %}









