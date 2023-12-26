---
layout: post
title:  "Working with multiple authentication providers and managers in Spring Boot"
date:   2023-12-26 11:15:01 +0100
categories: jekyll update
---

In modern web applications, particularly within microservice architectures, it's common to have external services and identity providers interacting with your application or service. Did you know that you can adjust your Spring Boot application so that you can add or remove support for those parties without much hassle? In this article I want to explore the idea and see what can we do to build around it.

Why multiple Authentication Providers? If your application needs to interact with different identity providers or use different user pools, each having its own set of credentials and security configurations, handling this with single authentication provider can become cumbersome and less maintenable. Application with multiple authentication providers can authenticate tokens issued by different issuers seamlessly, making the system more modular and scalable.

Let's say you have a service in your landscape, and the service can expose secure authentication endpoints via identity providers such as AWS Cognito, Azure AD, and more. However, times change, your business grows and you also want to expose your service to external parties, where you want to enable exposing endpoints to your partners and let them use your service.

For the first part of the equation, process is rather simple and well known, you can keep your configuration in the `application.yml` file (or files if you have multiple environments). However for the second part some manual intervention is required.

Let's see, how we can set up the first part, where your service is used internally and then we will build later on from that.


```java
@Configuration
public class InternalWebSecurityConfig {

    @Value("${spring.security.oauth2.resourceserver.jwt.jwk-set-uri}")
    private String jwkSetUri;

    @Bean
    public JwtDecoder jwtDecoder() {
        return NimbusJwtDecoder.withJwkSetUri(jwkSetUri).build();
    }

    @Bean
    public SecurityFilterChain internalFilterChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/api/internal/**")
            .authorizeHttpRequests((auth) -> auth
                .requestMatchers("/api/internal/**")
                .authenticated()
                .anyRequest().permitAll())
            .csrf(CsrfConfigurer::disable)
            .sessionManagement(sessionManagement -> sessionManagement
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .oauth2ResourceServer(oauth2ResourceServer ->
                                        oauth2ResourceServer
                                                .jwt(jwt -> jwt.decoder(jwtDecoder())));

        return http.build();
    }
}

```

`InternalWebSecurityConfig` is dedicated for protecting internal endpoints, and the setup is quite simple, any request going to the `/api/internal/**` needs to be authenticated, everything else is permitted. Looking at the `InternalWebSecurityConfig` class it indicates that it should have a yml with cognito user pool set up, something like:

```yml
spring:
    security:
        oauth2:
            resourceserver:
                jwt:
                    jwk-set-uri: https://cognito-idp.eu-west-1.amazonaws.com/eu-west-id/.well-known/jwks.json
```

<b>NOTE</b>: This can be shortened more if you are using one token issuer and spring boot auto configuration, the entire class can be reduced to exposing one bean:

```java
    @Bean
    public SecurityFilterChain internalFilterChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/api/internal/**")
            .authorizeHttpRequests(authorize -> authorize
                .requestMatchers("/api/internal/**")
                .authenticated()
                .anyRequest().permitAll())
            .csrf(CsrfConfigurer::disable)
            .sessionManagement(sessionManagement -> sessionManagement
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .oauth2ResourceServer((oauth2) -> oauth2.jwt(withDefaults()));
        return http.build();
    }

```

And this should work, anything coming to the `/api/internal/**` endpoints needs to have a token issued by our cognito defined in the configuration.

Now where things start to get interesting is when you want to expose your service to the external parties, for this usage I would suggest creating another `WebSecurityConfiguration`, to have the clear intention to separate the endpoints for internal and external purposes, it can be named something like `ExternalWebSecurityConfig`, and it's going to be a bit different the the first one. 

`ExternalWebSecurityConfig` relies on `UserPoolConfiguration` entities, to get the information in order to be able to authenticate the requests coming in with the authentication tokens corresponding to the given user pool. `UserPoolConfiguration` class should at least have `region` and `userPoolId` attributes in order for this configuration to work. In the shown example Amazon Cognito is used, that's the reason for the base url string `COGNITO_BASE_URL`, and the information about the user pool is fetched from the `UserPoolConfiguration` repository. For simplicity sake, I've added everything in this example class, in real world scenario you would split it up in appropriate classes, use database, etc.

```java
@Configuration
public class ExternalWebSecurityConfig {

    private final String COGNITO_BASE_URL = "https://cognito-idp.%s.amazonaws.com/%s";

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
                oauth2ResourceServer.authenticationManagerResolver(getAuthenticationManagerResolver());
            });
        return http.build();
    }

    public JwtIssuerAuthenticationManagerResolver getAuthenticationManagerResolver() {
        Map<String, AuthenticationManager> authenticationManagers = new HashMap<>();

        List<UserPoolConfiguration> userPoolConfigurationList = new UserPoolConfigurationRepository().findAll();

        userPoolConfigurationList.forEach(userPoolConfiguration -> {
            String poolConfig = String.format(COGNITO_BASE_URL, 
                                              userPoolConfiguration.region(), 
                                              userPoolConfiguration.userPoolId());

            addManager(authenticationManagers, poolConfig);
        });

        return new JwtIssuerAuthenticationManagerResolver(authenticationManagers::get);
    }

    public void addManager(Map<String, AuthenticationManager> authenticationManagers, String issuer) {
        JwtAuthenticationProvider authenticationProvider = new JwtAuthenticationProvider(JwtDecoders.fromOidcIssuerLocation(issuer));
        authenticationManagers.put(issuer, authenticationProvider::authenticate);
    }

    public record UserPoolConfiguration(String region, String userPoolId) {

    }

    public record UserPoolConfigurationRepository() {

        List<UserPoolConfiguration> findAll() {
            return List.of(new UserPoolConfiguration("eu-west-1", "eu-west-1_id"), 
                           new UserPoolConfiguration("eu-central-1", "eu-central-1_id"));
        }
    }

}

```

This config will make sure that during the application startup, information about the user pools is picked up from the database and loaded into the application. `ExternalWebSecurityConfig.getAuthenticationManagerResolver()` method will fetch the information about the user pools, and will create the appropriate authentication manager.

Now we have a configuration that will serve the described purpose. Your application will have `internal` and `external` endpoints, where configuration for internal endpoints is relying on application configuration, which will not be subjected to frequent changes, and the configuration for external endpoints will rely on database, where more frequent changes are possible.

However, this configuration will only load what's in the database during application startup. What if we want to crank this up a notch? Can we load or remove the information for the `external` configuration during application runtime? We will explore that in the next part of the series.

#### Full example described in this article is available on my [GitHub profile](https://github.com/skaplar/SpringSecurityPart1).


## Appendix

If you are still using Spring Security that predates 5.4, configuring filter chain is bit different. You'll need to extend `WebSecurityConfigurerAdapter` and override `configure` method. 


```java
@Configuration
@Order(4)
public class InternalWebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .requestMatchers()
            .antMatchers("/api/internal/**")
            .and()
            .authorizeRequests()
            .anyRequest().authenticated()
            .and()
            // ... whatever else you need to set
            .oauth2ResourceServer()
            .jwt();
    }
}

```

This configuration implies that yml file contains the value for `jwk-set-uri`, which was mentioned earlier. Similarly `ExternalWebSecurityConfig` needs to be adjusted as well.

```java
@Configuration
@Order(5)
public class ExternalWebSecurityConfig extends WebSecurityConfigurerAdapter {

    private final String COGNITO_BASE_URL = "https://cognito-idp.%s.amazonaws.com/%s";

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .requestMatchers()
            .antMatchers("/api/external/**")
            .and()
            .authorizeRequests()
            .anyRequest().authenticated()
            .and()
            // ... whatever else you need to set
            .oauth2ResourceServer(oauth2ResourceServer -> {
                oauth2ResourceServer.authenticationManagerResolver(getAuthenticationManagerResolver());
            });
    }

    // Rest of the code is the same as in previous example for `ExternalWebSecurityConfig`
}
```