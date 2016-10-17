+++
categories = [
]
description = "bla bla"
draft = true
title = "Springfox plugin for adding auth params"
tags = ["Springfox, Swagger, Spring"]
banner = ""
images = [
]
date = "2016-10-17T21:08:31+07:00"
menu = ""

+++

Generating REST API documentation using Springfox for one of the recent projects I found out that adding authentication header to private api methods is not an obvious task. In accordance with [Springfox reference](http://springfox.github.io/springfox/docs/current/) it could be done using *globalOperationParameters*, however in this case parameter will be added to every endpoint. [Dilip Krishnan suggests](http://stackoverflow.com/questions/36475452/reuse-complex-spring-fox-swagger-annotation) to create multiple dockets to separate public api from private one. But in my case the only method of public api is */login* and it is not reasonable to create docket for a single endpoint. Thus, I wrote a plugin to solve the issue. 
<!--more-->

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

/**
 * Springfox plugin that adds authentication token header to private api methods in documentation.
 */
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