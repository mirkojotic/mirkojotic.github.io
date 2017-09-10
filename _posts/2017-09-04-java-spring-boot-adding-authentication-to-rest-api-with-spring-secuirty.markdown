---
title: Java Spring Boot - Adding Authentication to REST API with Spring Security
layout: post
date: '2017-09-10 16:17:26'
---

> Spring Security is a powerful and highly customizable authentication and access-control framework. It is the de-facto standard for securing Spring-based applications.

Goals of this post:
* protect some of our REST API endpoints with Spring Security
* Add a couple of in memory users for testing purposes
* Change our tests so they don't fail after protecting our endpoints

Even though there are quite a few very usefull articles on how to protect Spring Boot REST API with Spring Security the matter is still a bit confusing for someone just starting to work with Spring.

The first thing we need to do is to add a Spring Security as a dependency in our pom.xml ( I'm using Maven ).

```xml
...
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-security</artifactId>
</dependency>
...
```

Now that we have Spring Security as our dependency and included in the project we can start implementing it.

Before I start working on this my directory structure looks like this:
```
├── derby.log
├── mvnw
├── mvnw.cmd
├── pom.xml
├── README.md
├── src
│   ├── main
│   │   ├── java
│   │   │   └── us
│   │   │       └── jotic
│   │   │           └── trello
│   │   │               ├── lane
│   │   │               │   ├── LaneController.java
│   │   │               │   ├── Lane.java
│   │   │               │   ├── LaneRepository.java
│   │   │               │   └── LaneService.java
│   │   │               ├── project
│   │   │               │   ├── ProjectController.java
│   │   │               │   ├── Project.java
│   │   │               │   ├── ProjectRepository.java
│   │   │               │   └── ProjectService.java
│   │   │               └── TrelloLookAlikeApplication.java
│   │   └── resources
│   │       ├── application.properties
│   │       ├── static
│   │       └── templates
```

Spring Security can be configured through xml config file or through Java classes. I've chosen the latter.

What do we need to do?

Following this [excellent article](http://www.baeldung.com) I was able to understand what needs to be done. 

Default settings of Spring Security assume that we will be using it in a full stack app along with templates and not just for a REST API. 

Because of that a response to an Unauthenticated request will be redirected to a login page. Similar thing will happen after successful login - we will be redirected to the page we wanted to access in the first place. 

When building a REST API we really don't want any of that. We want a **401 UNAUTHORIZED** response if a user is not authenticated to access a protected resource or we enter bad credentials. If we succeed with login we want  **200 OK** status code.

In order to accomplish this we need to extend a couple of classes and override a couple of methods.

But first let's make a brand new package which I will call `security`. This will hold all of our security related configuration classes.

Let's first make a entry point class for our authentication. We need it to respond to unauthenticated request with`401` status code instead of `302`.

```java
package us.jotic.trello.security;

import java.io.IOException;

import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.springframework.security.web.AuthenticationEntryPoint;
import org.springframework.stereotype.Component;

@Component("restAuthenticationEntryPoint")
public class RestAuthenticationEntryPoint implements AuthenticationEntryPoint {

   @Override
   public void commence(
		   HttpServletRequest request,
		   HttpServletResponse response,
		   org.springframework.security.core.AuthenticationException authException
		   ) throws IOException, ServletException {
	   
      response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Unauthorized");
	
	}
}
```

Since our entry point class implements `AuthenticationEntryPoint` interface we need to implement `commence` method. This is where we basically set the status code of response for any request that isn't authenticated.

After this we need to create our own custom on-login-success-handler.

```java
package us.jotic.trello.security;

import java.io.IOException;

import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.springframework.security.core.Authentication;
import org.springframework.security.web.authentication.SimpleUrlAuthenticationSuccessHandler;
import org.springframework.security.web.savedrequest.HttpSessionRequestCache;
import org.springframework.security.web.savedrequest.RequestCache;
import org.springframework.security.web.savedrequest.SavedRequest;
import org.springframework.util.StringUtils;

public class CustomAuthenticationSuccessHandler extends SimpleUrlAuthenticationSuccessHandler {

	private RequestCache requestCache = new HttpSessionRequestCache();
	
	@Override
	public void onAuthenticationSuccess(
			HttpServletRequest request,
			HttpServletResponse response,
			Authentication authentication) throws ServletException, IOException {
		
		SavedRequest savedRequest = requestCache.getRequest(request, response);
		
		if (savedRequest == null) {
			clearAuthenticationAttributes(request);
			return;
		}
		
		String targetUrlParam = getTargetUrlParameter();
		if(isAlwaysUseDefaultTargetUrl() 
				|| (targetUrlParam != null && StringUtils.hasText(request.getParameter(targetUrlParam)))) {
			requestCache.removeRequest(request, response);
			clearAuthenticationAttributes(request);
			return;
		}
		
		clearAuthenticationAttributes(request);
	}
	
	public void setRequestCache(RequestCache requestCache) {
		this.requestCache = requestCache;
	}
	
}
```

Much of this code was shamelesly stolen from an article that I've linked at the beginning of this post. 

What's happening here is that we're extending SimpleUrlAuthenticationSuccessHandler which is the success handler Spring Security uses by default. What we want to do override `onAuthenticationSuccess` method to avoid redirect ( `302` ).

So we don't just blindly override stuff let's take a look at what this `onAuthenticationSuccess` method is actually doing.

```java
	/**
	 * Calls the parent class {@code handle()} method to forward or redirect to the target
	 * URL, and then calls {@code clearAuthenticationAttributes()} to remove any leftover
	 * session data.
	 */
	public void onAuthenticationSuccess(HttpServletRequest request,
			HttpServletResponse response, Authentication authentication)
			throws IOException, ServletException {

		handle(request, response, authentication);
		clearAuthenticationAttributes(request);
	//...
```

Now we can clearly see what we wanted to avoid. `handle(request, response, authentication)` will redirect us after a successful login. So we got rid of that by overriding it in `CustomAuthenticationSuccessHandler`.

Finally, this is how our main configuration class for Spring Security ( `CustomSecurityConfigurer.java`) will look like:

```java
package us.jotic.trello.security;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.annotation.Order;
import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.web.authentication.SimpleUrlAuthenticationFailureHandler;

@Configuration
@EnableWebSecurity
@Order(101)
public class CustomSecurityConfigurer extends WebSecurityConfigurerAdapter {
	
	@Autowired
	private RestAuthenticationEntryPoint restAuthenticationEntryPoint;

	@Autowired
	private CustomAuthenticationSuccessHandler authenticationSuccessHandler;

	@Override
	protected void configure(AuthenticationManagerBuilder builder) throws Exception {
		builder.inMemoryAuthentication()
			.withUser("user").password("password").roles("USER")
			.and().withUser("admin").password("admin").roles("ADMIN");
	}
	
	@Override
	protected void configure(HttpSecurity http) throws Exception {
		http
			.csrf().disable()
			.exceptionHandling()
			.authenticationEntryPoint(restAuthenticationEntryPoint)
			.and()
			.authorizeRequests()
			.antMatchers("/api/**").authenticated()
			.and()
			.formLogin()
			.successHandler(authenticationSuccessHandler)
			.failureHandler(new SimpleUrlAuthenticationFailureHandler())
			.and()
			.logout();
	}
	
	
	@Bean
	public CustomAuthenticationSuccessHandler mySuccessHandler() {
		return new CustomAuthenticationSuccessHandler();
	}
	
	@Bean
	public SimpleUrlAuthenticationFailureHandler myFailureHandler() {
		return new SimpleUrlAuthenticationFailureHandler();
	}
	
}
```

This is our main configuration class and it extends `WebSecurityConfigurerAdapter`.

We are autowiring our newly created classes. Then we procede with creating our in memory users and overriding `configure` method to let Spring Security know which routes to protect and what to respond with on success and failiure ( we are registering our own handlers ).

This is the folder structure we end up with:
```
src/
├── main
│   ├── java
│   │   └── us
│   │       └── jotic
│   │           └── trello
│   │               ├── lane
│   │               │   ├── LaneController.java
│   │               │   ├── Lane.java
│   │               │   ├── LaneRepository.java
│   │               │   └── LaneService.java
│   │               ├── project
│   │               │   ├── ProjectController.java
│   │               │   ├── Project.java
│   │               │   ├── ProjectRepository.java
│   │               │   └── ProjectService.java
│   │               ├── security
│   │               │   ├── CustomAuthenticationSuccessHandler.java
│   │               │   ├── MySecurityConfigurer.java
│   │               │   └── RestAuthenticationEntryPoint.java
│   │               └── TrelloLookAlikeApplication.java
```

That will be all we need to set up authentication but now our test will fail. Fortunately we don't need to do a lot to make them work again.

```java
package us.jotic.trello.project;

//...
import org.springframework.security.test.context.support.WithMockUser;
//...

//...
public class ProjectControllerTest {

	//...	
	@Test
	@WithMockUser
	public void testShouldGetProject() throws Exception {
		Project project = new Project("Super duper Project", "Some description");
		Project p = projectService.addProject(project);
		
		MvcResult result = mockMvc.perform(get("/api/projects/" + p.getUuid()))
				.andExpect(status().is2xxSuccessful())
				.andReturn();
		String jsonResponse = result.getResponse().getContentAsString();
		Project projectFromJson = new ObjectMapper().readValue(jsonResponse, Project.class);
	
		assert(projectFromJson.getUuid().equals(p.getUuid()));
		assert(projectFromJson.getName().equals("Super duper Project"));
		assert(projectFromJson.getDescription().equals("Some description"));
	}
	//...
```

All we really need to do is add `@WithMockUser` annotation to our test cases.

Last but not least let's add tests for our authentication routes.

```java
package us.jotic.trello.security;


import static org.hamcrest.CoreMatchers.containsString;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.test.web.servlet.MockMvc;

@RunWith(SpringRunner.class)
@SpringBootTest
@AutoConfigureMockMvc
public class AuthenticationTest {

	@Autowired
	private MockMvc mockMvc;
	
	@Test
	public void testShouldReturn401_IfCredentialsBad() throws Exception {
		mockMvc.perform(post("/login")
					.contentType("application/x-www-form-urlencoded")
					.accept(MediaType.APPLICATION_JSON)
					.param("username", "Mirko")
					.param("password", "Password"))
				.andExpect(status().is4xxClientError())
				.andExpect(status().reason(containsString("Bad credentials")));
	}
	
	@Test
	public void testShouldReturn200_IfCredentialsRight() throws Exception {
		mockMvc.perform(post("/login")
					.contentType("application/x-www-form-urlencoded")
					.param("username", "root")
					.param("password", "root"))
				.andExpect(status().is2xxSuccessful());
	}
}
```

Some parts of code were omited for brevity. Here is a link to the github [repo](https://github.com/mirkojotic/trello-look-alike) if you need a reference.