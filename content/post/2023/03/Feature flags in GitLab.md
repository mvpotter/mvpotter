+++
title = "Feature flags in GitLab"
date = "2023-03-24T18:03:14+07:00"
tags = ["GitLab", "CI", "Feature Flags"]
categories = []
description = ""
menu = ""
banner = ""
images = []
draft = false
+++

Feature flag is a concept that allows to enable and disable features without the need to redeploy application. They provide much flexible way to manage features’ lifecycle. With feature flags you would have the following advantages:

- Features can be released for a subset of users to minimise risk of bugs and errors that affect a lot of users or apply A/B testing
- Allow to continuously integrate and deploy features without affecting user experience (you can get rid of long lived feature branches)
- Changes can be rolled back easily without application redeployment

However, along with advantages you usually get a set of trade offs:

- Feature flags add complexity to a code base
- They create technical debt as flags and old logic should be removed after a while
- Require much discipline and effort from developers and QAs to maintain related logic and ensuring that necessary flags are enabled on required platforms

There are a number of solutions for feature flags management like [FlagSmith](https://flagsmith.com/), [GrowthBook](https://www.growthbook.io/), [Flipt](https://www.flipt.io/), [Unleash](https://www.getunleash.io/) and others. However, if you already use GitLab, you can use its [feature flags](https://docs.gitlab.com/ee/operations/feature_flags.html) feature. Since version 13.5 GitLab is integrated with [Unleash](https://www.getunleash.io/) and provides both UI and API for feature flags management.

![](https://miro.medium.com/v2/resize:fit:1400/1*q-9flWSEmJ895KjuHJvGvQ.png)

In the **Feature Flags** section of GitLab UI you can create feature flags and enable them either for all or specific environments and users.

To integrate an application with the API it is necessary to click **Configure** and copy _API URL_ and _Instance ID_. The following example code would be in Java, but any backend can be integrated with it. [Unleash](https://www.getunleash.io/) has SDKs for all popular stacks.

The first step is to add SDK dependency to the project:

```xml
<dependency>  
  <groupId>io.getunleash</groupId>  
  <artifactId>unleash-client-java</artifactId>  
  <version>Latest version here</version>  
</dependency>
```

Then it is necessary to put _API URL_ and _Instance ID_ to properties. Here _unleash.env_ set additionally. It allows application to be aware on which environment it was launched. So we can turn a feature on or off for it.

```
unleash:   
  url: https://gitlab.noveogroup.com/api/v4/feature_flags/unleash/4321   
  key: SDFasfsfasfdFDsafD   
  env: localya
```

Then we define Unleash bean with the parameters above. Using _UnleashConfig_ it is possible to configure other useful parameters, e.g. default value in case feature flag does not exist, or delay for a client to fetch actual flags’ state.

```java
@Configuration   
public class UnleashConfiguration {   
   
 @Bean   
 public Unleash unleash(   
   @Value("${unleash.url}") final String url,   
   @Value("${unleash.key}") final String instanceId,   
   @Value("${unleash.env}") final String env   
 ) {   
   return new DefaultUnleash(   
     UnleashConfig.builder()   
                  .unleashAPI(url)   
                  .instanceId(instanceId)   
                  .appName(env)   
                  .build()   
   );   
 }   
   
}
```

Now lets define some feature flag. Actually, feature flag is defined by its name. However, for the sake of type safety let store it as enum.

```java
@Getter   
public enum FeatureToggleType {   
   
  AWESOME_FEATURE("awesome-feature");   
   
  private final String code;   
   
  FeatureToggleType(final String code) {   
    this.code = code;   
  }   
   
}
```

To fetch feature flag info from API we need appropriate class to put values into:

```java
@Getter   
@Setter   
@NoArgsConstructor   
@AllArgsConstructor   
public class FeatureToggle {   
  private String name;  
  private boolean enabled;  
}
```

And define a service that can check feature flag state or fetch a list of flags available for the current environment. Our application proxies requests to Unleash for frontend not to have additional dependencies. So, the latter method helps to provide necessary data.

```java
@Service   
public class FeatureToggleService {   
   
  private final Unleash unleash;   
   
  public FeatureToggleServiceImpl(final Unleash unleash) {   
    this.unleash = unleash;   
  }   
   
  public boolean isEnabled(final FeatureToggleType featureToggle) {   
    return unleash.isEnabled(featureToggle.getCode());   
  }   
   
  public List<FeatureToggle> list() {   
    return unleash.more().evaluateAllToggles().stream()   
                  .map(toggle -> new FeatureToggle(toggle.getName(), toggle.isEnabled()))   
                  .collect(Collectors.toList());   
  }   
   
}
```

And the last step is to add condition to check feature flag and launch necessary logic:

```java
if (featureToggleServive.isEnabled(FeatureToggleType.AWESOME_FEATURE)) {  
  awesomeStrategy.execute();  
} else {  
  legacyStrategy.execute();  
}
```

GitLab feature flags work great for relatively small projects as it has its limitations. For bigger project you can use [Unleash](https://www.getunleash.io/) directly. Fortunately, migration shouldn’t be that hard.