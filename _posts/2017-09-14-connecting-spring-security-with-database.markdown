---
title: Java Spring - Connecting Spring Security To  A Database
layout: post
date: '2017-09-14 21:04:24'
---

In my previous [post](http://jotic.us/2017/09/10/java-spring-boot-adding-authentication-to-rest-api-with-spring-secuirty.html) I've successfully set up Spring Security to protect my REST API. But I did so with in memory users and where is the fun in that.  So "mad with power" I decided to add to current security setup and add a `/register` route for adding new users and also allow those users to log in and do stuff.

So by the end of this post I'll demonstrate:
* how to connect Spring Security to a database by using DAO authentication provider
* how to add users with UserDetailsService
* how to test all this with integration tests

### Connecting Spring Security to a database by using DAO authentication provider

Firsts thing first. In order to allow user registration and subsequent authentication we need to have them stored somewhere. So lets proceed and create a `User` entity. But before that it would I will be making a package where I'll store all user related classes.

```java
package us.jotic.trello.user;

import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.Id;
import javax.persistence.Table;

import org.hibernate.annotations.GenericGenerator;

@Entity
@Table(name = "users")
public class User {

	@Id
	@GeneratedValue(generator="system-uuid")
	@GenericGenerator(name="system-uuid", strategy = "uuid")
	@Column(name = "uuid", unique = true)
	private String id;
	
    @Column(nullable = false, unique = true)
    private String username;
 
    private String password;
    
    private boolean isAccountNonExpired;
	private boolean isAccountNonLocked;
    private boolean isCredentialsNonExpired;
    private boolean isEnabled;
		
		// getters and setters 
}
```

Next we will be needing a user repository, which our UserDetailsService will be using.

```java
package us.jotic.trello.user;

import org.springframework.data.repository.CrudRepository;

public interface UserRepository extends CrudRepository<User, String> {

	public User findByUsername(String username);
}
```

Now, lets create our CustomUserDetailsService class which will implement UserDetailsService interface. For our authentication we care about one specific method `loadUserByUsername`.

```java
package us.jotic.trello.user;

import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;

@Service
public class CustomUserDetailsService implements UserDetailsService {
	
	@Autowired
	private UserRepository userRepository;

	@Override
	public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
		User user = userRepository.findByUsername(username);
		if(user == null) {
			throw new UsernameNotFoundException(username);
		}
		return new CustomUserPrincipal(user);
	}
}
```

Last line of `loadUserByUsername` is wrapping our `User` object in a `CustomUserPrincipal`. Principal is our currently logged in user. In order for this to work our custom principal class has to implement `UserDetails` interface.

```java
package us.jotic.trello.user;

import java.util.Collection;

import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

public class CustomUserPrincipal implements UserDetails {

	private static final long serialVersionUID = 1L;
	private User user;
	
	public CustomUserPrincipal(User user) {
		this.setUser(user);
	}

	@Override
	public Collection<? extends GrantedAuthority> getAuthorities() {
		return null;
	}

	@Override
	public String getPassword() {
		return user.getPassword();
	}

	@Override
	public String getUsername() {
		return user.getUsername();
	}

	@Override
	public boolean isAccountNonExpired() {
		return true;
	}

	@Override
	public boolean isAccountNonLocked() {
		return true;
	}

	@Override
	public boolean isCredentialsNonExpired() {
		return true;
	}

	@Override
	public boolean isEnabled() {
		return true;
	}

	public User getUser() {
		return user;
	}

	public void setUser(User user) {
		this.user = user;
	}

}
```

The contract that is `UserDetails` interface requires us to define a couple of methods whether we chose to use it or not and they are pertaining to validity of users account. We'll return true from all those methods because authentication will not work otherwise but we might want to implement them for real later.

#### On to Spring Security Configuration

In the previous post I've linked above you can see how our custom Custom Security Configurer class looks like but after configuring our authentication provider and our password encoder it will look something like this

```java
package us.jotic.trello.security;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.annotation.Order;
import org.springframework.security.authentication.dao.DaoAuthenticationProvider;
import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.authentication.SimpleUrlAuthenticationFailureHandler;

import us.jotic.trello.user.CustomUserDetailsService;

@Configuration
@EnableWebSecurity
@Order(101)
public class MySecurityConfigurer extends WebSecurityConfigurerAdapter {
	
	@Autowired
	private RestAuthenticationEntryPoint restAuthenticationEntryPoint;
	@Autowired
	private CustomAuthenticationSuccessHandler authenticationSuccessHandler;
	@Autowired
	private CustomUserDetailsService userDetailsService;

	@Override
	protected void configure(AuthenticationManagerBuilder auth) throws Exception {
		auth.authenticationProvider(authenticationProvider());
	}
	
	@Bean
	public DaoAuthenticationProvider authenticationProvider() {
		DaoAuthenticationProvider authProvider = new DaoAuthenticationProvider();
		authProvider.setUserDetailsService(userDetailsService);
		authProvider.setPasswordEncoder(encoder());
		return authProvider;
	}

	@Bean
	public PasswordEncoder encoder() {
		return new BCryptPasswordEncoder();
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
}
```

And that's really all we need to connect Spring Security to a database and allow users stored in a database to authenticate. But lets go a step further and make a `/register` route so we can test this out. I mean automated testing and unit testing are amazing and irreplacable tools in a programmers toolbox but sometimes its just satisfying to see things working.


### Adding users

Let's make a `UserController` with a `/register` route.

```java
package us.jotic.trello.user;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class UserController {
	
	@Autowired
	private CustomUserDetailsService userService;

	@RequestMapping(method = RequestMethod.POST, value = "/register")
	public User register(@RequestBody User user) {
		return userService.saveNewUser(user);
	}
}
```

As you can see we are using `userService.saveNewUser` method. Well we have to add that method to our `userService` first.

```java
	//...
	@Autowired
	private PasswordEncoder passwordEncoder;
	//...
	public User saveNewUser(User userData) {
		User user = new User();
		user.setUsername(userData.getUsername());
		user.setPassword(passwordEncoder.encode(userData.getPassword()));
		return userRepository.save(user);
	}
	//...
```

And that should be it, not production ready but a good start. We're good to go... except we don't have any tests. We don't want that now do we.

Here is the content of our `UserControllerTest` class

```java
package us.jotic.trello.user;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.MvcResult;

import com.fasterxml.jackson.databind.ObjectMapper;

@RunWith(SpringRunner.class)
@SpringBootTest
@AutoConfigureMockMvc
public class UserControllerTest {

	@Autowired
	private MockMvc mockMvc;
	@Autowired
	private UserRepository userRepository;
	@Autowired
	private PasswordEncoder passwordEncoder;
	
	@Test
	public void shouldDemonstrateHowToCreateANewUser() throws Exception {
		// create a project
		String pwd = "secret";
		User user = new User();
		user.setUsername("mirko_jotic");
		user.setPassword(passwordEncoder.encode(pwd));
		User savedUser = userRepository.save(user);
		assert(passwordEncoder.matches(pwd, savedUser.getPassword()));
	}
	
	@Test
	public void shouldCreateANewUserTroughAPI() throws Exception {
		MvcResult result = mockMvc.perform(post("/register")
					.contentType("application/json")
					.content("{\"username\":\"mirko\",\"password\":\"secret\"}"))
				.andExpect(status().is2xxSuccessful())
				.andReturn();
		
		String jsonResponse = result.getResponse().getContentAsString();
		User user = new ObjectMapper().readValue(jsonResponse, User.class);
		assert(user.getUsername().equals("mirko"));
	}
	
	@Test
	public void createNewUserAndTryToLogIn() throws Exception {
		String un = "somePerson";
		String pw = "somePersonsPwd";
		MvcResult result = mockMvc.perform(post("/register")
					.contentType("application/json")
					.content("{\"username\":\""+ un +"\",\"password\":\""+ pw +"\"}"))
				.andExpect(status().is2xxSuccessful())
				.andReturn();
		
		String jsonResponse = result.getResponse().getContentAsString();
		User user = new ObjectMapper().readValue(jsonResponse, User.class);
		
		// perform actual login
		mockMvc.perform(post("/login")
					.contentType("application/x-www-form-urlencoded")
					.accept(MediaType.APPLICATION_JSON)
					.param("username", un)
					.param("password", pw))
					.andExpect(status().is2xxSuccessful());
		
	}
}
```

Some parts of code were omited for brevity. Here is a link to the github [repo](https://github.com/mirkojotic/trello-look-alike) if you need a reference.