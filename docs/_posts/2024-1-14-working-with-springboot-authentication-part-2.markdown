---
layout: post
title:  "Dynamically adding support for identity providers in Spring Boot applications"
date:   2024-1-14 10:15:01 +0100
categories: java spring
---

In my previous article I've explained how it is possible to [work with multiple authentication providers and managers in Spring Boot]({% post_url 2023-12-26-working-with-springboot-authentication %}) in order to resolve the information and interact with different identity providers. While, that is super convenient and useful, with a bit of work it can be improved in order to support dynamic adding or removing identity providers during application runtime.

I will add the new functionality on top of the examples from the previous article, so if you're struggling to understand some of the concepts please do read that article first. 

Let's start by modifying the `ExternalWebSecurityConfig` class, which was introduced to allow external parties to connect and use some of the provided services via exposed endpoints. 

```java
@Configuration
public class ExternalWebSecurityConfig {

    private final String COGNITO_BASE_URL = "https://cognito-idp.%s.amazonaws.com/%s";

    private final CustomAuthManagerResolver customAuthManagerResolver;
    private final UserPoolConfigurationRepository userPoolConfigurationRepository;

    public ExternalWebSecurityConfig(final CustomAuthManagerResolver customAuthManagerResolver,
                                     final UserPoolConfigurationRepository userPoolConfigurationRepository) {
        this.customAuthManagerResolver = customAuthManagerResolver;
        this.userPoolConfigurationRepository = userPoolConfigurationRepository;
    }

    @Bean
    public SecurityFilterChain externalFilterChain(HttpSecurity http) throws Exception {
        http
                .securityMatcher("/api/external/**")
                .authorizeHttpRequests(authorize -> authorize
                        .requestMatchers("/api/external/**")
                        .authenticated()
                        .anyRequest().permitAll()
                ).csrf(CsrfConfigurer::disable)
                .sessionManagement(sessionManagement -> sessionManagement
                        .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .oauth2ResourceServer(oauth2ResourceServer -> {
                    customAuthManagerResolver.setIssuer(getAuthenticationManagerResolver());
                    oauth2ResourceServer.authenticationManagerResolver(customAuthManagerResolver);
                });
        return http.build();
    }

    public JwtIssuerAuthenticationManagerResolver getAuthenticationManagerResolver() {
        Map<String, AuthenticationManager> authenticationManagers = new HashMap<>();

        List<UserPoolConfiguration> userPoolConfigurationList = userPoolConfigurationRepository.findAll();

        userPoolConfigurationList.forEach(userPoolConfiguration -> {
            String poolConfig = String.format(COGNITO_BASE_URL, userPoolConfiguration.region(), userPoolConfiguration.userPoolId());
            addManager(authenticationManagers, poolConfig);
        });

        return new JwtIssuerAuthenticationManagerResolver(authenticationManagers::get);
    }

    public void addManager(Map<String, AuthenticationManager> authenticationManagers, String issuer) {
        JwtAuthenticationProvider authenticationProvider = new JwtAuthenticationProvider(JwtDecoders.fromOidcIssuerLocation(issuer));
        authenticationManagers.put(issuer, authenticationProvider::authenticate);
    }

}


```


Introduced change was injecting and leveraging `CustomAuthManagerResolver`, which is actually a class that implements `AuthenticationManagerResolver`. New snippet is part of the method `externalFilterChain` which is used for configuring security filter chain. Inside `oauth2ResourceServer` method is a lambda for configuring oauth2ResourceServer, first we setup custom auth manager resolver by calling `getAuthenticationManagerResolver`, then we set customAuthManagerResolver as the authentication manager resolver, like this: `oauth2ResourceServer.authenticationManagerResolver(customAuthManagerResolver)`.

As previously mentioned `CustomAuthManagerResolver` implements `AuthenticationManagerResolver`, and also declares `JwtIssuerAuthenticationManagerResolver` which is used for resolving http requests. Constructor can be implemented differently/shortened, however left like this for clarity.

```java
@Configuration
public class CustomAuthManagerResolver implements AuthenticationManagerResolver<HttpServletRequest> {

    private JwtIssuerAuthenticationManagerResolver customIssuerAuthenticationManagerResolver;

    public CustomAuthManagerResolver() {
        Map<String, AuthenticationManager> authenticationManagers = new HashMap<>();
        this.customIssuerAuthenticationManagerResolver = new JwtIssuerAuthenticationManagerResolver(authenticationManagers::get);
    }

    @Override
    public AuthenticationManager resolve(final HttpServletRequest request) {
        return customIssuerAuthenticationManagerResolver.resolve(request);
    }

    public void setIssuer(JwtIssuerAuthenticationManagerResolver jwtIssuerAuthenticationManagerResolver) {
        this.customIssuerAuthenticationManagerResolver = jwtIssuerAuthenticationManagerResolver;
    }

}
```

Now, once everything is ready let's see how the `UserPool` information can be added during runtime. This approach is customizable and can be implemented in various ways, depending on the application requirements. Inputing the needed data can be handled by an administrator, or by a backoffice employee, it can first get validated on the service which handles user pool configurations and communicated via event to this service, etc. But for the sake of brewity, we will do it straightforward and will allow `internal` endpoints on this service to receive and create different user pool configurations in order to support storing configurations for `UserPools` and expose `external` endpoints to the users whose tokens are coming from those user pools, during runtime. 

```java
@RestController
@RequestMapping("/api/internal")
public class InternalController {

    private final UserPoolConfigurationService userPoolConfigurationService;

    public InternalController(UserPoolConfigurationService userPoolConfigurationService) {
        this.userPoolConfigurationService = userPoolConfigurationService;
    }

    @PostMapping
    public void addManager() {
        userPoolConfigurationService.save(new UserPoolConfiguration("eu-west-1", "eu-west-1_identifier_no_2"));
    }
}

```

This is the internal controller, which can receive `POST`s and create new `UserPoolConfiguration`. `addManager` method should be customized with parameters corresponding to the appropriate `UserPool`. First parameter is the `region` while the second parameter is the `userPoolId`.`UserPoolConfigurationService` does rest of the work, which storing the configuration to the database and notifying the `CustomAuthManagerResolver` about the newly introduced changes.

```java
@Service
public class UserPoolConfigurationService {

    private final CustomAuthManagerResolver customAuthManagerResolver;
    private final UserPoolConfigurationRepository userPoolConfigurationRepository;
    private final ExternalWebSecurityConfig externalWebSecurityConfig;

    public UserPoolConfigurationService(CustomAuthManagerResolver customAuthManagerResolver,
                                        UserPoolConfigurationRepository userPoolConfigurationRepository,
                                        ExternalWebSecurityConfig externalWebSecurityConfig) {
        this.customAuthManagerResolver = customAuthManagerResolver;
        this.userPoolConfigurationRepository = userPoolConfigurationRepository;
        this.externalWebSecurityConfig = externalWebSecurityConfig;
    }

    public void save(UserPoolConfiguration userPoolConfiguration) {
        userPoolConfigurationRepository.save(userPoolConfiguration);
        customAuthManagerResolver.setIssuer(externalWebSecurityConfig.getAuthenticationManagerResolver());
    }
}
```

As we can see from the example, `save` method saves the user configuration, and also uses `CustomAuthManagerResolver` to set the issuer by passing the value obtained from call to `getAuthenticationManagerResolver` which will load the newly created `UserPoolConfiguration`.

Voilà, this is the minimal but complete example to make the mentioned things work. Now you can add the information about the user pools via `internal` endpoints, and have them immediately available on the `external` endpoints, without the need for the new deplyoments, scaling or changes to application configurations.

#### I've provided full example described in this article on my [GitHub profile](https://github.com/skaplar/SpringSecurityPart2).