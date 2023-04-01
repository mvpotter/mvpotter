+++
tags = ["test"]
description = ""
menu = ""
banner = ""
title = "PODAM (POjo DAta Mocker)"
images = [
]
categories = [
]
date = "2017-01-01T19:30:40+07:00"
+++

Writing unit tests, especially for data converters, requires creating a lot of POJOs and populate them to see that fields' values are mapped correctly. In my previous experience, projects had simple POJOs and it was not a big deal to fill them with data manually, however, one of the last projects has complex model and creating test objects with all the relations became an issue. I started to finding for a solution that can create and populate objects for me and found a great tool called [PODAM](https://devopsfolks.github.io/podam/).

<!--more-->

PODAM is abbreviation for POjo DAta Mocker and its API for creating beans is pretty simple. All you need to do is create ```PodamFactory``` instance and use its ```manufacturePojo``` method that instantiates object and fills its fields with random values:

```java
PodamFactory factory = new PodamFactoryImpl();
Order order = factory.manufacturePojo(Order.class);
```

Sometimes you need to specify rules for field values. For example, ids of entities have ```String``` type and ```UUID```format. For generated entities to pass validation logic smoothly it is necessary to generate valid ids. [PODAM](https://devopsfolks.github.io/podam/) suggests two ways to deal with such issues:

The first one is to implement ```AttributeStrategy``` class and add ```@PodamStrategyValue``` annotation on required field. However, in my case this approach was undesirable, because I needed to test business entities and it is not a good idea to clutter them with annotation from test dependencies.

The second approach is to implement custom ```TypeManufacturer```

```java
public class CustomStringManufacturer extends StringTypeManufacturerImpl {

    private static final String FIELD_ID = "id";

    @Override
    public String getType(final DataProviderStrategy strategy, final AttributeMetadata attributeMetadata,
                    	  final Map<String, Type> genericTypesArgumentsMap) {
        if (FIELD_ID.equals(attributeMetadata.getAttributeName())) {
            return UUID.randomUUID().toString();
        }
        return super.getType(strategy, attributeMetadata, genericTypesArgumentsMap);
    };

}
```

and register it for using by ```PodamFactory```

```java
podamFactory.getStrategy().addOrReplaceTypeManufacturer(String.class, new CustomStringManufacturer());
```

I described only a subset of [PODAM](https://devopsfolks.github.io/podam/) features. You can always visit their [documentation](https://devopsfolks.github.io/podam/) for further info.

Another alternative that I found later is [random-beans](https://github.com/benas/random-beans/wiki) project. However, I did not have time to test it thoroughly. If you know other tools for generating and populating beans feel free to share your experience in comments.
