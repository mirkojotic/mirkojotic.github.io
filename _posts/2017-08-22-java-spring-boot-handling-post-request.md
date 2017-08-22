---
title: Java Spring Boot - handling POST request
layout: post
date: '2017-08-22 20:30:00'
---

We want to be able to send a POST request to our Spring Boot REST API and create a resource. We also want to be able to test this functionality automatically without relying to manual testing ( Postman and such ).

This is a part of my learning experience and I'll be using an example from a simple practice project I'll be building. - a trello like application. Naming stuff is hard :)

Project structure:
```
src
├── main
│   ├── java
│   │   └── us
│   │       └── jotic
│   │           └── trello
│   │               ├── project
│   │               │   ├── ProjectController.java
│   │               │   ├── Project.java
│   │               │   ├── ProjectRepository.java
│   │               │   └── ProjectService.java
│   │               └── TrelloLookAlikeApplication.java
...
```

Let's start with `Project.java`

```
package us.jotic.trello.project;

import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.Id;
import javax.persistence.Table;

import org.hibernate.annotations.GenericGenerator;


@Entity
@Table(name = "project")
public class Project {

	@Id
	@GeneratedValue(generator="system-uuid")
	@GenericGenerator(name="system-uuid", strategy = "uuid")
	@Column(name = "uuid", unique = true)
	private String uuid;
	private String name;
	private String description;
	
	public Project(String name, String description) {
		this.setName(name);
		this.setDescription(description);
	}
	
	public Project() {
		
	}

	public String getUuid() {
		return uuid;
	}

	public void setUuid(String uuid) {
		this.uuid = uuid;
	}

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public String getDescription() {
		return description;
	}

	public void setDescription(String description) {
		this.description = description;
	}
	
	
}
```

This is our model and it will map to a row of data in our database. Properties mapped to columns. 

There is a couple of interesting details here. 

As a challenge more than anything else I've decided to use UUID's as my primary keys. It took some research but I'm satisfied how it turned out.
```
...
@GeneratedValue(generator="system-uuid") ; // name of the primary key generator to use
@GenericGenerator(name="system-uuid", strategy = "uuid"); // used by our ORM
...
```

Another thing that I had a problem with is a JSON serializer ( jackson ). It appears that when serializing/desirializing Java objects jackson needs to have either a dummy constructor ( that accepts no parameters ) or a @JsonCreator annotation. I've opted for a simpler although not necessarily better solution and  just added a dummy constructor.

Next we need to make a repository for our model (`ProjectRepository.java`)

```
package us.jotic.trello.project;

import org.springframework.data.repository.CrudRepository;

public interface ProjectRepository extends CrudRepository<Project, String> {

}
```

Very simple but gives us great power by exposing CRUD methods on our model.

Following an established convention we need to make a `ProjectService.java` to interact with our repository

```
package us.jotic.trello.project;

import java.util.List;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class ProjectService {

	@Autowired
	private ProjectRepository projectRepository;

	public List<Project> getAllProjects() {
		return (List<Project>) projectRepository.findAll();
	}

	public Project getProject(String uuid) {
		return projectRepository.findOne(uuid);
	}

	public Project addProject(Project project) {
		return projectRepository.save(project);
	}
}

```

We are using @Autowired to inject our repository class and the rest of the code is pretty self explanotory.

And now the actuall outward facing API. Let's make a `ProjectController.java`

```
package us.jotic.trello.project;

import java.util.List;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class ProjectController {
	
	@Autowired
	private ProjectService projectService;

	@RequestMapping(method = RequestMethod.GET, value = "/projects")
	public List<Project> getAllProjects() {
		return projectService.getAllProjects();
	}
	
	@RequestMapping(method = RequestMethod.GET, value = "/projects/{uuid}")
	public Project getProject(@PathVariable String uuid) {
		return projectService.getProject(uuid);
	}
	
	@RequestMapping(method = RequestMethod.POST, value = "/projects")
	public Project addProject(@RequestBody Project project) {
		return projectService.addProject(project);
	}
}

```

We are using @Autowired annotation to inject `ProjectService` in our controller. Also there are three endpoints. Two for getting Projects and a third that we'll actually use to create new projects. This should be all we need to create a project through our API but we need to check if this is actually working.

In order to avoid manual testing every time we make a change to any of the parts of our API we will want to set up some sort of testing. 

Let's set up integration testing with `MockMvc`.

This is the rest of our updated folder structure

```
...
└── test
    └── java
        └── us
            └── jotic
                └── trello
                    ├── project
                    │   └── ProjectControllerTest.java
                    └── TrelloLookAlikeApplicationTests.java

```

Let's add a test class for our controller `ProjectControllerTest`

```
package us.jotic.trello.project;

import static org.mockito.Matchers.contains;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

import java.util.List;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.MvcResult;

@RunWith(SpringRunner.class)
@SpringBootTest
@AutoConfigureMockMvc
public class ProjectControllerTest {

	@Autowired
	private MockMvc mockMvc;

	@Test
	public void testAddingProject_shouldSucced() throws Exception {
		MvcResult result = mockMvc.perform(post("/projects")
					.contentType("application/json")
					.content("{\"name\":\"Test\",\"description\":\"Testing\"}"))
				.andExpect(status().is2xxSuccessful())
				.andReturn();
		
		String jsonResponse = result.getResponse().getContentAsString();
		assert(jsonResponse.contains("\"name\":\"Test\",\"description\":\"Testing\""));
	}
}

```

This might not be the best way of approaching testing but is the way I've found easy to set up. I would rather have weird looking tests then no tests at all.

We are injecting `MockMvc` with `@AutoConfigureMockMvc` annotation.

`MockMvc` enables us to bootstrap our app and gives us application context without starting a server. Our code will be executed the same way it would be in production. 

In order for tests to be ran we need to add `@Test` annotation before public methods in our test classes. Otherwise our methods will be ignored.

What we are doing in `testAddingProject_shouldSucced` test case? We are  performing a `POST` request to `/projects`  endpoint.
We also  set  "`Content-Type`": "`application/json`" as a header on our Request. Lastly we need to send our payload ( data to be saved as a `Project` ).
We want to assert that HTTP status code returned was in 2xx and that saved object has same values as those passed in Request.