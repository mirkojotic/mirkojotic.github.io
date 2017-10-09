---
title: Java Spring - bidirectional relationship between models
layout: post
date: '2017-08-27 16:50:00'
---

This is a part of a practice project I'm working on to learn Spring Boot. The end goal is to make an REST API that would support an app similar to Trello. 

Goal of this post is to demonstrate how to build bidirectional relationship between entities. In this case that will be a relationship between `Project` and `Lane` entities.

`Project` can have multiple lanes while a `Lane` can only have one `Project`. Let us describe our end goal with a couple of integration tests. 

```java
package us.jotic.trello.lane;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.MvcResult;

import com.fasterxml.jackson.databind.ObjectMapper;

import us.jotic.trello.project.Project;
import us.jotic.trello.project.ProjectService;

@RunWith(SpringRunner.class)
@SpringBootTest
@AutoConfigureMockMvc
public class LaneControllerTest {

	@Autowired
	private MockMvc mockMvc;
	@Autowired
	private ProjectService projectService;
	
	@Test
	public void createLaneBelongingToOneProject() throws Exception {
		// create a project
		Project project = new Project("Super duper Project", "Some description");
		Project p = projectService.addProject(project);
	
		// create a lane
		MvcResult result = mockMvc.perform(post("/projects/" + p.getUuid() + "/lanes")
					.contentType("application/json")
					.content("{\"name\":\"Test\",\"description\":\"Testing\"}"))
				.andExpect(status().is2xxSuccessful())
				.andReturn();
		
		// unmarshal JSON response into a Lane class
		String jsonResponse = result.getResponse().getContentAsString();
		Lane laneFromJson = new ObjectMapper().readValue(jsonResponse, Lane.class);
	
		// assert returned lane object has everything it has to have
		assert(laneFromJson.getName().equals("Test"));
		assert(laneFromJson.getDescription().equals("Testing"));
		// assert relationship with project was established
		assert(laneFromJson.getProject().getUuid().equals(p.getUuid()));
	}

	@Test
	public void assertProjectHasLanesAssociatedWithIt() throws Exception {
		// create a project
		Project project = new Project("Super duper Project", "Some description");
		Project p = projectService.addProject(project);
	
		// create a lane
		MvcResult resultLane = mockMvc.perform(post("/projects/" + p.getUuid() + "/lanes")
					.contentType("application/json")
					.content("{\"name\":\"Test\",\"description\":\"Testing\"}"))
				.andExpect(status().is2xxSuccessful())
				.andReturn();
				
		String laneResponse = resultLane.getResponse().getContentAsString();
		Lane lane = new ObjectMapper().readValue(laneResponse, Lane.class);
		
		// get project that our new Lane belongs to
		MvcResult result = mockMvc.perform(get("/projects/" + p.getUuid()))
									.andExpect(status().is2xxSuccessful())
									.andReturn();
		
		// unmarshal JSON to Project class
		String jsonResponse = result.getResponse().getContentAsString();
		Project projectFromJson = new ObjectMapper().readValue(jsonResponse, Project.class);
	
		// Project lanes should be a list of lanes - right now only one Lane
		assert(projectFromJson.getLanes().size() == 1);
		assert(projectFromJson.getLanes().get(0).getUuid().equals(lane.getUuid()));
	}
	
}

```

How do we go about accomplishing this?

First we need to annotate our entities appropriatelly. Lane is associated with one Project while a Project can have multiple Lanes. Let's translate this into annotations.

```java
// Lane imports not included
@Entity
@Table(name = "lane")
public class Lane {

	@Id
	@GeneratedValue(generator="system-uuid")
	@GenericGenerator(name="system-uuid", strategy = "uuid")
	@Column(name = "uuid", unique = true)
	private String uuid;
	private String name;
	private String description;
	
	@ManyToOne
	private Project project;
	
	// constructors, getters and setters excluded for brevity
```

All we have to do is add a project property to our Lane entity and annotate it wih `@ManyToOne`. This is basically all we need to do to establish relationship between a Lane and a Project but what if we need to have access to Lanes from our Project model. In this case we would establish what is refered to as Bidirectional relationship where two entities reference each other.

This is what we need to add to our `Project` class.

```java
// Project imports not included
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

	@OneToMany(mappedBy = "project")
	private List<Lane> lanes;
```

So to associate a `Project` with a `Lane` we need to add `lanes` property and annotate it with `@OneToMany(mappedBy = "project")`. `mappedBy` argument defines a property in `Lane` entity that references a  `Project`.

This is going to work but we will run into an issue with our JSON serializer when we want to return `Project` with all associated lanes. The problem is that these two entities reference each other and while trying to serialize them into JSON, our serializer ( in my case Jackson ) will run into a circular dependency problem. It will try to serialize a `Project` and then it will try to serialize every `Lane` but since `Lane` references `Project` it will serialize the `Project` again and eventually following this circular reference it will throw StackOverflowException.  So we will need a way to tell our serializer to stop circulary serializing these entities.

Fortunatelly, kind people who are developing `jackson` have came up with a solution to this and it's by object reference. Their JSON object reference requires every object to have unique id. The only issue here is that object reference needs to unique for all entities involved as oposed to only being unique within entity. For me this was the easiest solution because I'm using `UUID` for PK of all entities which are unique accross the board.

**Sidenote.** My solution is only one of many available ones. Check out this [article](http://www.baeldung.com/jackson-bidirectional-relationships-and-infinite-recursion) for more.

The only thing we need to do is to add a line to both our entities.
```java
@Entity
@Table(name = "project")
@JsonIdentityInfo(generator = ObjectIdGenerators.PropertyGenerator.class, property = "uuid")
public class Project {
	// ...
}
```
and 
```java
// Lane imports not included
@Entity
@Table(name = "lane")
@JsonIdentityInfo(generator = ObjectIdGenerators.PropertyGenerator.class, property = "uuid")
public class Lane {
	// ...
}
```

With `@JsonIdentityInfo` we notify `jackson` what unique id for our objects will be. In my case it is "uuid".

Now we can add our `LaneRepository`, `LaneService` and `LaneController`
```java
// LaneRepository
package us.jotic.trello.lane;

import java.util.List;

import org.springframework.data.repository.CrudRepository;

public interface LaneRepository extends CrudRepository<Lane, String>{

	public List<Lane> findByProjectUuid(String projectId);
}
```

```java
// LaneService
package us.jotic.trello.lane;

import java.util.List;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class LaneService {

	@Autowired
	private LaneRepository laneRepository;
	
	public List<Lane> getAll(String projectUuid) {
		return (List<Lane>) laneRepository.findByProjectUuid(projectUuid);
	}
	
	public Lane get(String uuid) {
		return laneRepository.findOne(uuid);
	}
	
	public Lane create(Lane lane) {
		return laneRepository.save(lane);
	}
	
	public Lane update(Lane lane) {
		return laneRepository.save(lane);
	}
}
```

```java
// LaneController
package us.jotic.trello.lane;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

import us.jotic.trello.project.Project;
import us.jotic.trello.project.ProjectService;

import java.util.List;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@RestController
public class LaneController {

	public static final Logger logger = LoggerFactory.getLogger(LaneController.class);
    
	@Autowired
	private LaneService laneService;
	
	@Autowired
	private ProjectService projectService;
	
	@RequestMapping(method = RequestMethod.GET, value = "/projects/{projectUuid}/lanes")
	public List<Lane> getAllLanes(@PathVariable String projectUuid) {
		return laneService.getAll(projectUuid);
	}
	
	@RequestMapping(method = RequestMethod.POST, value = "/projects/{projectUuid}/lanes")
	public Lane saveLane(@PathVariable String projectUuid, @RequestBody Lane lane) {
		Project project = projectService.getProject(projectUuid);
		lane.setProject(project);
		return laneService.create(lane);
	}
}
```

Creating a couple of projects and attaching a lane to one of them will result in following output in JSON:
```json
[
    {
        "uuid": "ff8081815e32c850015e32c8905b0000",
        "name": "Learning Spring Boot",
        "description": "Fun, so much fun",
        "lanes": []
    },
    {
        "uuid": "ff8081815e32c850015e32c8e3b40001",
        "name": "Learning Javascript",
        "description": "Experimental project",
        "lanes": [
            {
                "uuid": "ff8081815e32c850015e32c934460002",
                "name": "TODO",
                "description": "List of tasks to do",
                "project": "ff8081815e32c850015e32c8e3b40001"
            }
        ]
    }
]
```

With these latest additions our tests should be passing. Some parts of code were omited for brevity. Here is a link to the github [repo](https://github.com/mirkojotic/trello-look-alike) if you need a reference.