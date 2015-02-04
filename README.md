Setting up Spring Security in the HST
===============================
##Introduction
One of the challenges in implementing HippoCMS usually comes down to authenticating user on the site (HST) with external systems. These systems are either integrated with by usage of a standardized protocol or by implementing custom authentication logic to authenticate with e.g. legacy systems. In this lab I will elaborate on using a common security integration framework, Spring Security, in HippoCMS.
 
Spring Security is best described by its definition, as seen on its webpage: "Spring Security is a powerful and highly customizable authentication and access-control framework. It is the de-facto standard for securing Spring-based applications.". HippoCMS is a Spring-based application, so it seems trivial to use Spring Security in a HippoCMS-based application. Spring Security on its own is a framework, thus allowing us to bridge our own security logic into HippoCMS by means of Spring Security.

As one of the most mature security frameworks for Spring-based applications, Spring Security provides many out-of-the-box components for authentication and access control. Normally, integrating with external identity providers potentially is really difficult, even when using  standardized security protocols like SAML2, OAuth2 and LDAP are used. Using a best-of-breed security framework, providing out-of-the-box components, HippoCMS will be empowered with all the security integration capabilities offered by Spring Security. 

So, let’s get started on enabling the endless possibilities of Spring Security on a vanilla HippoCMS. In this lab we’ll dive into setting up Spring Security in a HippoCMS project. If you're already familiar with setting up Spring Security and/or you're more interesting in implementing Spring Security in your HST managed website, proceed to the "Implementing Spring Security in HippoCMS" lab.

####Prerequisites
 - Completion of the "Hello World with Hippo" labs 
 - A vanilla HippoCMS (built with the OneHippo Archetype),  version 7.9.x+

####Step 1 - Declare versions to use as Maven properties
It’s good practise in Maven to manage dependency versions in a central place. Add the versions to use in this walkthrough to your properties section in the root POM of your project (./pom.xml):
```xml
<properties>
  ...
  <spring.security.version>3.2.3.RELEASE</spring.security.version>
  <hst-springsec.version>0.03.01</hst-springsec.version>
  ...
</properties>
```
####Step 2 - Install Spring Security
First, you need Spring Security to be actually included in your HST application, to leverage the framework its possibilities in your HST application. Adding the Spring Security dependencies, allows you to configure your filter chain to intercept inbound requests and process them using security rules. 

In Maven, dependency management is best done in a central place. In a HippoCMS project, this would be the root POM of your project ('/pom.xml'). Include the required Spring Security dependencies in your root POM like this (note that we use the version properties declared in Step 1):
```xml
<dependencyManagement>
  ...
  <dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-core</artifactId>
    <version>${spring.security.version}</version>
  </dependency>
  <dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-web</artifactId>
    <version>${spring.security.version}</version>
  </dependency>
  <dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-config</artifactId>
    <version>${spring.security.version}</version>
  </dependency>
  <dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-taglibs</artifactId>
    <version>${spring.security.version}</version>
  </dependency>
  ...
</dependencyManangement>
```
Next, reference the managed dependencies in the POM of your site module (./site/pom.xml) to actually have Spring Security packaged in the site.war:
```xml
<dependencies>
  ...
  <dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-core</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-config</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-taglibs</artifactId>
  </dependency>
  ...
</dependencies>
```
####Step 3 - Install the HST Spring Security Plugin
Having Spring Security on board, we repeat the same for the HST Spring Security Plugin. This plugin is required mainly for two reasons:

 - It provides a Spring Security Authentication Provider on top of the Hippo Repository. This allows you to authenticate against your users/groups stored in Hippo, using Spring Security.
 - It provides a SpringSecurity valve, which is to be plugged in the HST request processing valves. This valve is required to convert Spring Security security principal to an HST-compatible security principal. You’ll need this to actually have the remaining HST valves reuse the security principal set by Spring Security.

So, again we first add the HST Spring Security Plugin to the root POM of your project (./pom.xml) as a managed dependency, alongside the Spring Security dependencies from Step 2:
```xml
<dependencyManagement>
  ...
  <dependency>
    <groupId>org.onehippo.forge.hst-springsec</groupId>
    <artifactId>hst-springsec</artifactId>
    <version>${hst-springsec.version}</version>
  </dependency>
  ...
</dependencyManagement>
```
And again, reference the managed dependency in the POM of your site module (./site/pom.xml) to actually have HST Spring Security Plugin packaged in the site.war:
```xml
<dependencies>
  ...
  <dependency>
    <groupId>org.onehippo.forge.hst-springsec</groupId>
    <artifactId>hst-springsec</artifactId>
  </dependency>
  ...
</dependencies>
```
Great, by now all your dependencies are in place, and your HippoCMS project should start without errors when booting it with Cargo:

`mvn clean install && mvn -Pcargo.run`

####Step 4 - Set up the Spring context for Spring Security
To start configuring a security context, create a general application context first to store common Spring configuration used throughout the entire application. Create a new context under ‘./site/src/main/webapp/WEB-INF/applicationContext.xml’, containing an empty configuration:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">
</beans>
```

Next, we'll also add a security context configuration, where we'll configure Spring Security authentication providers and security rules, which are matched against intercepted requests. Once an intercepted request can be matched to one of your declared rules, Spring Security will act upon your declaration, e.g. authenticating with the authentication provider, checking the authentication principal for required roles, finally forwarding to the requested page. Create a new security context under './site/src/main/webapp/WEB-INF/applicationContext.xml’, containing a minimum viable configuration, authenticating all inbound requests against the Hippo Repository:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:security="http://www.springframework.org/schema/security"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd                            
         http://www.springframework.org/schema/security http://www.springframework.org/schema/security/spring-security.xsd">

  <!-- Bypass SITE requests, originating from the CMS, for correct Channel Manager  operation -->
  <security:http pattern="/_rp/**" security="none"/>
  <security:http pattern="/_cmsrest/**" security="none"/>
  <security:http pattern="/_cmsinternal/**" security="none"/>

  <!-- Bypass SITE requests targeting static resources  -->
  <security:http pattern="/static/**" security="none"/>

  <!-- Bypass SITE requests targeting the ping servlet to allow health checks -->
  <security:http pattern="/ping/**" security="none"/>

  <!-- Secure SITE other requests -->
  <security:http auto-config="true" use-expressions="true">
    <!-- Interception constraints -->
    <security:intercept-url pattern="/**" access="isFullyAuthenticated()"/>
         
    <!-- Login/Logout -->
    <security:form-login default-target-url="/"/>
    <security:logout logout-url="/logout" 
      invalidate-session="true"
      delete-cookies="JSESSIONID"/>
  </security:http>

  <!-- Spring Security Authentication Manager -->
  <security:authentication-manager>
    <security:authentication-provider ref="hippoAuthenticationProvider"/>
  </security:authentication-manager>

  <!-- Hippo Repository Authentication Provider, as provided by the HST Spring Security Plugin -->
  <bean id="hippoAuthenticationProvider"   
class="org.onehippo.forge.security.support.springsecurity.authentication.HippoAuthenticationProvider">
  </bean>
</beans>
```
####Step 5 - Add the Spring Security Processing valve to the HST
All the (remaining) valves in the HST request processing pipeline, need to become aware of Spring Security authenticated principals. To do so, we need to inject a Spring Security Valve in the HST request processing pipeline. This additional processing valve will pick up a Spring Security authenticated principal and convert it to an HST-compatible principal. Doing so, allows all the remaining valves in the request processing pipeline to reuse your Spring Security authenticated principal, without the need to modify the HST.

To inject the Spring Security Valve into the appropriate pipeline (default, rest/content, rest/plain), 
you need to create a new Spring configuration file in the HST overrides. Do so, by adding this configuration to ‘site/src/main/resources/META-INF/hst-assembly/overrides/spring-security.xml’:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:lang="http://www.springframework.org/schema/lang"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:util="http://www.springframework.org/schema/util"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
                           http://www.springframework.org/schema/lang http://www.springframework.org/schema/beans/spring-lang-3.0.xsd
                           http://www.springframework.org/schema/util http://www.springframework.org/schema/util/spring-util-3.0.xsd
                           http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-3.0.xsd">

  <!-- Defininig SpringSecurityValve -->
  <bean id="springSecurityValve" class="org.onehippo.forge.security.support.springsecurity.container.SpringSecurityValve">
    <property name="valveName" value="springSecurityValve" />
  </bean>

  <!--
    Inserting SpringSecurityValve into the default pipeline.
    You may copy and paste the following block to insert the SpringSecurityValve for more pipelines.
    'DefaultSitePipeline' is for the default website pipeline.
    'JaxrsRestContentPipeline' is for the Content/Context Aware JAX-RS Service pipeline.
    'JaxrsRestPlainPipeline' is for the Plain JAX-RS Service pipeline.
  -->
  <bean class="org.springframework.beans.factory.config.MethodInvokingFactoryBean">
    <property name="targetObject">
      <bean class="org.springframework.beans.factory.config.MethodInvokingFactoryBean">
        <property name="targetObject" ref="org.hippoecm.hst.core.container.Pipelines" />
        <property name="targetMethod" value="getPipeline" />
        <property name="arguments">
          <list>
            <!--
              You may use one of the following: 'DefaultSitePipeline', 'JaxrsRestContentPipeline' and 'JaxrsRestPlainPipeline'.
            -->
            <value>DefaultSitePipeline</value>
          </list>
        </property>
      </bean>
    </property>
    <property name="targetMethod" value="addInitializationValve" />
    <property name="arguments">
      <ref bean="springSecurityValve" />
    </property>
  </bean>
</beans>
```
####Step 6 - Wire everything together
So, to wrap up, let's wire all configurations together in your site its web.xml. Here you’ll tie up your HST application, Spring Security framework, your Spring contexts and the request filters required to actually have all configurations work together to secure your application! 

**Configure context loading in the site web.xml**
Add the following listener, enabling a listener to scan for Spring contexts and load them:
```
  <listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  </listener>
```

**Configure your Spring Contexts in the site web.xml**
Add the following context parameters, providing the path to your Spring (Security) contexts, picked up by the context loader configured before:
```
  <context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>
      /WEB-INF/applicationContext.xml
      /WEB-INF/applicationContext-security.xml
    </param-value>
  </context-param>
```

**Configure your request filters in the site web.xml**
Add a Spring request filter, to handle inbound requests by processing them in your Spring Security context. Make sure to add the filter **before** the HST Request Filter, since the HST Spring Security Valve depends on a Spring Security managed principal!
```
  <filter>
    <filter-name>springSecurityFilterChain</filter-name>
    <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
  </filter>
 
  <filter-mapping>
    <filter-name>springSecurityFilterChain</filter-name>
    <url-pattern>/*</url-pattern>
    <dispatcher>REQUEST</dispatcher>
  </filter-mapping>
```

Congratulations! Your HippoCMS project now has been fully configured with a basic Spring Security stack, authenticating against the Hippo Repository. Launch HippoCMS again using Cargo:
`mvn clean install && mvn -Pcargo.run`

Navigating to [http://localhost:8080/site](http://localhost:8080/site/) normally shows you the homepage (in this case a HST-managed 404 page). Now, since Spring Security intercepts your request for a secured page (remember the /** in your interception rules) and there is no authenticated principal, Spring Security redirects you to an internal login page. Login with your HippoCMS credentials (e.g. admin/admin) and you will be redirected back to the homepage, now correctly showing the HST-managed 404 page. 
To logout, visit  [http://localhost:8080/site/logout](http://localhost:8080/site/logout). You'll that your session gets invalidated (inspect JSESSIONID in a browser debugger) and that you're redirected to the default logout page ("/"), for which authentication is required, resulting in a second redirect back to the login page.

####Conclusion
In this lab, we've discovered how to construct a simple security stack in Hippo CMS in an easy way, using Spring Security. This allows us to take full control of our HST site users authenticating against some authentication backend. Hopefully, you’ve learned where and why you actually need to configure certain elements, what this means for the inner workings of HippoCMS, and how you can leverage a powerful security framework to easily extend HippoCMS.

From this point on, we can start to explore the seamless integration of Spring Security features with the HST, allowing you to interact with Spring Security from within your own HST components, HST templates and integrating your login/logout scenarios with Spring Security.
