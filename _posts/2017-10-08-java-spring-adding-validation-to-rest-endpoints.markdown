---
title: Java Spring - Adding validation to REST endpoints
layout: post
date: '2017-10-08 21:04:24'
---

In this post I will be demonstrating one of the ways you can set up validation for REST API endpoints that create entities. This specific problem can be solved in a number of ways, so I don't claim this is the way to do it but I did found it quite simple and easy to reason about.

In order to achieve validation or rather impose constraints on properties of the entity we need to add annotations to our entity class properties and add an annotation to respective methods in a controller that handle POST or PUT. That will prevent saving non-complete or non-sanatized objects to the database. The way it works is that when you try to save such a model to  a database an Exception will be thrown, a `MethodArgumentNotValidException` exception to be precise.

So that solves one problem, but introduces quite a bit of confusion for your front end developers because the response they will get while trying to save such a model is just a good ol' `500 Internal Server Error`. That is definitely not what we want.

So our goal here is to return all relevant validation error messages to the caller.

Let's start by annotating our Entities properly and configuring methods of our controller.

```java
package us.jotic.trello.project;

import java.util.List;
import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.Id;
import javax.persistence.OneToMany;
import javax.persistence.Table;
import javax.validation.constraints.NotNull;
import javax.validation.constraints.Size;
import org.hibernate.annotations.GenericGenerator;
import com.fasterxml.jackson.annotation.JsonIdentityInfo;
import com.fasterxml.jackson.annotation.ObjectIdGenerators;

import us.jotic.trello.lane.Lane;

@Entity
@Table(name = "projects")
@JsonIdentityInfo(generator = ObjectIdGenerators.PropertyGenerator.class, property = "uuid")
public class Project {

	@Id
	@GeneratedValue(generator="system-uuid")
	@GenericGenerator(name="system-uuid", strategy = "uuid")
	@Column(name = "uuid", unique = true)
	private String uuid;
	
	@NotNull
	@Size(min=5, max=20)
	private String name;
	
	@NotNull
	@Size(min=10, max=500)
	private String description;

	@OneToMany(mappedBy = "project")
	private List<Lane> lanes;
	
	public Project(String name, String description) {
		super();
		this.setName(name);
		this.setDescription(description);
	}
	
	public Project() {
		
	}
	// getters and setters omitted for brevity
}
```

What's new here is the `@NotNull` and `@Size(min=5, max=20)` annotations. `@NotNull` will ensure that the field is present when trying to save the entity and `@Size(min=5, max=20)` will make sure that the length of those properties are what we want them to be. 

Let's configure our controller to take this new configuration into account.

```java
	@RequestMapping(method = RequestMethod.POST)
	public Project addProject(@Valid @RequestBody Project project) {
		return projectService.addProject(project);
	}
```

Thing to look for here is `@Valid` annotation.

So if we try to save a an instance of `Project` that has properties that do not comply to our constraints a `MethodArgumentNotValidException` will be thrown.

We could handle these exceptions in methods of our controller but there is a better or at least more DRY way of accomplishing this.

**Spring enables us to create an Exception handler that will be responsible for handling exception of certain type in all of our controllers.**

But before we get to that we will want to make a DTO class that will hold all of our validation error messages. I've actually made all related classes into `validation` package. I will be showing:
- ValidationError ( container for all fields validation error messages )
- FieldError ( one field validation error message )
- ValidationRestErrorHandler ( exception handler for all `MethodArgumentNotValidException` that happen in our controllers )

```java
// ValidationError.java
package us.jotic.trello.validation;

import java.util.List;
import java.util.ArrayList;

public class ValidationError {
	
	private List<FieldError> fieldErrors = new ArrayList<>();

	public ValidationError() {
		
	}
	
	public void addFieldError(String path, String message) {
		FieldError error = new FieldError(path, message);
		fieldErrors.add(error);
	}
	
	public List<FieldError> getFieldErrors() {
		return fieldErrors;
	}
}
```
This is a very simple class that holds all error messages and that will be serialized and sent to the client.

```java
// FieldError.java
package us.jotic.trello.validation;

public class FieldError {
	
	private String field;
	private String message;
	
	public FieldError(String field, String message) {
		this.field = field;
		this.message = message;
	}

	public String getField() {
		return field;
	}

	public String getMessage() {
		return message;
	}
}
```
This is an even simpler class that will be basically be one instance of a validation error message and be apart of the `fieldErrors` ArrayList of the previous class.

And finally a class where all the magic happens `ValidationRestErrorHandler`.
```java
package us.jotic.trello.validation;

import java.util.List;
import java.util.Locale;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.MessageSource;
import org.springframework.context.i18n.LocaleContextHolder;
import org.springframework.validation.BindingResult;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.validation.FieldError;

// By default @ControllerAdvice apply globally to all @Controller annotated classes
// needless to say that this includes @RestController as it is a combination of 
// @Controller and @ResponseBody
// this annotation is how we let Spring know we want to handle all 
// MethodArgumentNotValidException here
@ControllerAdvice
public class ValidationRestErrorHandler {

	// Strategy interface for resolving messages,
	// with support for the parameterization and internationalization of such messages.
	private MessageSource messageSource;
    
	// Autowiring MessageSource interface
	// Implementation class => DelegatingMessageSource
	@Autowired
	public ValidationRestErrorHandler(MessageSource messageSource) {
		this.messageSource = messageSource;
	}
   
	 // defining exception handler for all MethodArgumentNotValidException
	 // this exception occurs when validation constraints are not satisfied
	 // we're building validation errors when this happens and returning them
	@ExceptionHandler(MethodArgumentNotValidException.class)
	@ResponseStatus(HttpStatus.BAD_REQUEST)
	@ResponseBody
	public ValidationError processValidationError(MethodArgumentNotValidException ex) {
		// MethodArgumentNotValidException.getBindingResult returns results of failed validation
		BindingResult result = ex.getBindingResult();
		List<FieldError> fieldErrors = result.getFieldErrors();
		return processFieldErrors(fieldErrors);
	}
   
	// processing validation error messages
	public ValidationError processFieldErrors(List<FieldError> fieldErrors) {
		// our data transfer object will hold all validation error messages
		ValidationError dto = new ValidationError(); 
		for(FieldError fieldError : fieldErrors) {
			String localizedErrorMessage = resolveLocalizedErrorMessage(fieldError);
			// adding a field error to our data transfer object
			dto.addFieldError(fieldError.getField(), localizedErrorMessage);
		}
		return dto;
	}
   
	public String resolveLocalizedErrorMessage(FieldError fieldError) {
		// this is where we would return a i18n translation of validation messages
		// it's out of scope but you can check out how here:
		// https://www.petrikainulainen.net/programming/spring-framework/spring-from-the-trenches-adding-validation-to-a-rest-api/
		// coincidentally ( :D ) this is the main resource for this post
		Locale currentLocale = LocaleContextHolder.getLocale();
		// messageSource is implementation of messageSource interface
		// class in question is DelegatingMessageSource
		return messageSource.getMessage(fieldError, currentLocale);
	}
}
```

I wont go into depth explaining what's happening here as it is overcommented, but in a nutshell we are handling an exeption that occurs when trying to save an entity that's not valid. We get errors from that exception and then format them and create  a ValidationError dto to be serialized and sent back to the client.

**Important** detail to note here is `@ControllerAdvice` annotation which Spring picks up and uses to let our controller know that this class is handling these type of exceptions.

With all that in place, we'll write a couple of tests to verify this functionality actually works.

Let's add a test case to our `ProjectControllerTest.java`

```java
	@Test
	@WithMockUser
	public void testFailWithValidationMessages() throws Exception {
		
		MvcResult result = mockMvc.perform(post("/api/projects")
					.contentType("application/json")
					.content("{\"name\":\"Test\",\"description\":\"Testing\"}"))
				.andExpect(status().is4xxClientError())
				.andReturn();
		
		String jsonResponse = result.getResponse().getContentAsString();
		
		assert(jsonResponse.contains("{\"field\":\"name\",\"message\":\"size must be between 5 and 20\"}"));
		assert(jsonResponse.contains("{\"field\":\"description\",\"message\":\"size must be between 10 and 500\"}"));
	}
	
	@Test
	@WithMockUser
	public void testFailWithValidationMessagesIfRequiredFieldsAreNotPresent() throws Exception {
		
		MvcResult result = mockMvc.perform(post("/api/projects")
					.contentType("application/json")
					.content("{}"))
				.andExpect(status().is4xxClientError())
				.andReturn();
		
		String jsonResponse = result.getResponse().getContentAsString();
		
		assert(jsonResponse.contains("{\"field\":\"name\",\"message\":\"may not be null\"}"));
		assert(jsonResponse.contains("{\"field\":\"description\",\"message\":\"may not be null\"}"));
	}
```

Both of these tests should pass and as we can see here both of our validation constraints are in action.

Some parts of code were omited for brevity. Here is a link to the github [repo](https://github.com/mirkojotic/trello-look-alike) if you need a reference.