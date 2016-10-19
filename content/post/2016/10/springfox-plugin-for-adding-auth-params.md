+++
categories = [
]
description = ""
draft = false
title = "Springfox plugin for adding auth params"
tags = ["Springfox", "Swagger", "Spring"]
banner = "/post/2016/10/springfox-plugin-for-adding-auth-params.png"
images = [
]
date = "2016-10-17T21:08:31+07:00"
menu = ""

+++

Generating REST API documentation using Springfox for one of the recent projects I found out that adding authentication header to private api methods is not an obvious task. In accordance with [Springfox reference](http://springfox.github.io/springfox/docs/current/) it could be done using *globalOperationParameters*, however in this case parameter will be added to every endpoint. Dilip Krishnan [suggests](http://stackoverflow.com/questions/36475452/reuse-complex-spring-fox-swagger-annotation) to create multiple dockets to separate public api from private one. But in my case the only method of public api is */login* and it is not reasonable to create docket for a single endpoint. Thus, I wrote a plugin to solve the issue. 
<!--more-->
All extensibility points are described in [Springfox reference](http://springfox.github.io/springfox/docs/current/#plugins). As the documentation states plugin should:

1. Implement one of the extention points.
2. Have apprpriate initialization order.
3. Be declared as a Spring bean for plugin registry to pick it up.
 
```java
import org.springframework.stereotype.Component;
import org.springframework.core.annotation.Order;
import springfox.documentation.spi.service.OperationBuilderPlugin;
import springfox.documentation.swagger.common.SwaggerPluginSupport;

@Component
@Order(SwaggerPluginSupport.SWAGGER_PLUGIN_ORDER)
public class AuthenticationTokenHeaderBuilder implements OperationBuilderPlugin {
    ...
}
```

In my case the most appropriate extension point is *OperationBuilderPlugin* as I need to process each endpoint and add authentication header to private api ones. 

Extension points require two methods to be implemented:

1. Spring plugin's method ```boolean supports()``` to understand if plugin should be applied. The implementation for Springfox is pretty simple

    ```java
    @Override
    public boolean supports(final DocumentationType documentationType) {
        return DocumentationType.SWAGGER_2.equals(documentationType);
    }
    ```
2. Method ```void apply(context)```, which is being invoked for each api endpoint and provides appropriate context (e.g. *OperationContext* for *OperationBuilderPlugin*) as an argument. 

    ```java
    @Override
    public void apply(final OperationContext context) {
        // Get endpoint request mapping
        final String mapping = context.requestMappingPattern();      
        // Check if private api endpoint    
        if (!PUBLIC_API_MAPPINGS.contains(mapping)) {                    
            final List<Parameter> parameters = new LinkedList<>();
            // Create auth header parameter
            parameters.add(parameterBuilder
                                   .parameterType(HEADER_PARAMETER_TYPE)
                                   .name(HttpTokenHelper.HEADER_TOKEN)
                                   .modelRef(new ModelRef(STRING_TYPE))
                                   .description(HEADER_DESCRIPTION)
                                   .allowMultiple(false)
                                   .required(true)
                                 .build());
            // Add parameter to endpoint documentation 
            context.operationBuilder().parameters(parameters);           
          }
    } 
    ```
    
As a result, every endpoint, except */login* has authentication token header in generated documentation.

![Image](/post/2016/10/springfox-plugin-for-adding-auth-params-1.png)
![Image](/post/2016/10/springfox-plugin-for-adding-auth-params-2.png)

The whole code of the plugin is the following.

```java
import com.invitro.registry.security.HttpTokenHelper;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import springfox.documentation.builders.ParameterBuilder;
import springfox.documentation.schema.ModelRef;
import springfox.documentation.service.Parameter;
import springfox.documentation.spi.DocumentationType;
import springfox.documentation.spi.service.OperationBuilderPlugin;
import springfox.documentation.spi.service.contexts.OperationContext;
import springfox.documentation.swagger.common.SwaggerPluginSupport;

import java.util.Collections;
import java.util.LinkedList;
import java.util.List;

@Component
@Order(SwaggerPluginSupport.SWAGGER_PLUGIN_ORDER)
public class AuthenticationTokenHeaderBuilder implements OperationBuilderPlugin {

    public static final String API_LOGIN_ENDPOINT = "/api/login";
    public static final String HEADER_PARAMETER_TYPE = "header";
    public static final String HEADER_DESCRIPTION = "Authentication token (see " + API_LOGIN_ENDPOINT + ")";
    public static final String STRING_TYPE = "string";

    private static final List<String> PUBLIC_API_MAPPINGS = Collections.singletonList(API_LOGIN_ENDPOINT);

    private ParameterBuilder parameterBuilder = new ParameterBuilder();

    @Override
    public boolean supports(final DocumentationType documentationType) {
        return DocumentationType.SWAGGER_2.equals(documentationType);
    }

    @Override
    public void apply(final OperationContext context) {
        final String mapping = context.requestMappingPattern();
        if (!PUBLIC_API_MAPPINGS.contains(mapping)) {
            final List<Parameter> parameters = new LinkedList<>();
            parameters.add(parameterBuilder
                                   .parameterType(HEADER_PARAMETER_TYPE)
                                   .name(HttpTokenHelper.HEADER_TOKEN)
                                   .modelRef(new ModelRef(STRING_TYPE))
                                   .description(HEADER_DESCRIPTION)
                                   .allowMultiple(false)
                                   .required(true)
                                   .build());
            context.operationBuilder().parameters(parameters);
        }
    }

}
```