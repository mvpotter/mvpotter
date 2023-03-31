+++
title = "Route Spring transactions to master and slave"
date = "2023-03-31T11:13:18+04:00"
tags = ["Spring", "DataSource"]
categories = []
description = ""
menu = ""
banner = ""
images = []
draft = false
+++

With the grow of a project there might be a need to forward read requests to slave node. Fortunately, it could be easily achieved in Spring with `@Transactional` annotation a bit of configuration.

Firstly, let's define a node type. Here, we have values for master and slave only. However, the set might be extended, for example if you need separate connection pools for fast and long queries.

```java
public enum DatabaseNodeType {  
    MASTER,  
    SLAVE
}
```

Then it is necessary to define a context holder to be aware which transaction is used in the specified thread. A bit later we would configure interceptor that checks arguments of `Transactional` annotation and sets appropriate node type.

```java
public final class DatabaseContextHolder {  
  
    private static final ThreadLocal<DatabaseNodeType> CONTEXT = new ThreadLocal<>();  
  
    private DatabaseContextHolder() {  
        throw new UnsupportedOperationException(DatabaseContextHolder.class.getSimpleName()  
                                                + " cannot be instantiated");  
    }  
  
    public static void set(DatabaseNodeType DatabaseNodeType) {  
        CONTEXT.set(DatabaseNodeType);  
    }  
  
    public static DatabaseNodeType getEnvironment() {  
        return CONTEXT.get();  
    }  
  
    public static void reset() {  
        CONTEXT.set(DatabaseNodeType.MASTER_FAST);  
    }  
  
}
```

`AbstractRoutingDataSource` makes almost all the magic in the current approach. We need to override `determineCurrentLookupKey` method and return a value from context holder. It would help to decide which datasource should be used when transaction is being created.

```java
public class MasterSlaveRoutingDataSource extends AbstractRoutingDataSource {  

    @Override  
    protected Object determineCurrentLookupKey() {  
        return DatabaseContextHolder.getEnvironment();  
    }  
 
}
```

Then it is necessary to define datasource beans and define `MasterSlaveRoutingDataSource` as primary datasource that would use appropriate datasource node type.

```java
@Configuration  
public class JpaRepositoryConfiguration {  
  
    public static final String MASTER_DATA_SOURCE = "MASTER_DATA_SOURCE";  
  
    public static final String SLAVE_DATA_SOURCE = "SLAVE_DATA_SOURCE";  
    
    @Bean  
    @Primary    
    public DataSource dataSource() {  
        MasterSlaveRoutingDataSource masterSlaveRoutingDataSource = new MasterSlaveRoutingDataSource();  
        Map<Object, Object> targetDataSources = new HashMap<>();  
        targetDataSources.put(DatabaseNodeType.MASTER, masterFastDataSource());  
        targetDataSources.put(DatabaseNodeType.SLAVE, slaveSlowDataSource());  
        masterSlaveRoutingDataSource.setTargetDataSources(targetDataSources);  
        masterSlaveRoutingDataSource.setDefaultTargetDataSource(masterFastDataSource());  
        return masterSlaveRoutingDataSource;  
    }  
  
    @Bean(MASTER_DATA_SOURCE)  
    @ConfigurationProperties(prefix = "datasource.master")  
    public DataSource masterDataSource() {  
        return DataSourceBuilder.create().build();  
    }  
  
    @Bean(SLAVE_DATA_SOURCE)  
    @ConfigurationProperties(prefix = "datasource.slave")  
    public DataSource slaveDataSource() {  
        return DataSourceBuilder.create().build();  
    }  
  
}
```

And all that is left is putting appropriate key to context holder when transaction starts. There are at least 2 approaches how to do it.

### Using AOP

The following aspect intersects invocations that marked as `@Transactional` and either located in **com.mvpotter** package or sub packages or annotated with `@Repository`. It is necessary to reset the context after method invocation. Otherwise, the thread might be reused later and it might try to write using read only transaction or do other unexpected things.

```java
@Aspect  
@Order(0)  
@Component  
public class TransactionAspect {  
  
    @Around("(within(com.mvpotter.*) || @annotation(org.springframework.stereotype.Repository)) && @annotation(transactional)")  
    public Object proceed(ProceedingJoinPoint proceedingJoinPoint, Transactional transactional) throws Throwable {  
        try {  
            if (transactional.readOnly()) {  
                DatabaseContextHolder.set(DatabaseNodeType.SLAVE);  
            } else { 
                DatabaseContextHolder.set(DatabaseNodeType.MASTER);  
            }  
            
            return proceedingJoinPoint.proceed();  
        } finally {  
            DatabaseContextHolder.reset();  
        }  
    }  
  
}
```

### Creating a wrapper for `PlatformTransactionManager`

We need to wrap `PlatformTransactionManager` and update context holder value when transaction is being fethed.

```java
public class ReplicaAwareTransactionManager implements PlatformTransactionManager {  
  
    private final PlatformTransactionManager wrapped;  
      
    public ReplicaAwareTransactionManager(final PlatformTransactionManager wrapped) {  
    this.wrapped = wrapped;  
    }  
      
    @Override  
    public TransactionStatus getTransaction(TransactionDefinition definition) {  
        if (definition != null && definition.isReadOnly() == true) {  
            DatabaseContextHolder.set(DatabaseNodeType.SLAVE);  
        } else {  
            DatabaseContextHolder.set(DatabaseNodeType.MASTER);  
        }  
      
        try {  
            return wrapped.getTransaction(definition);  
        } finally {  
            DatabaseContextHolder.reset();  
        }  
    }  
  
    @Override  
    public void commit(TransactionStatus status) throws TransactionException { 
        wrapped.commit(status);  
    }  
      
    @Override  
    public void rollback(TransactionStatus status) throws TransactionException {  
        wrapped.rollback(status);  
    }  
  
}
```

And then define beans to wrap transaction manager and make the wrapper primary manager.

```java
@Bean  
@Primary  
public PlatformTransactionManager transactionManager(@Qualifier("dataSourceTransactionManager") PlatformTransactionManager transactionManager) {  
    return ReplicaAwareTransactionManager(transactionManager);  
}  
  
@Bean("dataSourceTransactionManager")  
public PlatformTransactionManager platformTransactionManager() {  
    return DataSourceTransactionManager(dataSource());  
}
```