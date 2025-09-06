# odoo-nmit-hackathon-2025
#pom.xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.synergy</groupId>
    <artifactId>synergy-sphere-java</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.5</version>
        <relativePath/>
    </parent>

    <properties>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <!-- Spring Boot Web + JPA -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>

        <!-- Database -->
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>runtime</scope>
        </dependency>

        <!-- Vaadin -->
        <dependency>
            <groupId>com.vaadin</groupId>
            <artifactId>vaadin-spring-boot-starter</artifactId>
            <version>24.3.6</version>
        </dependency>

        <!-- Lombok (optional for getters/setters) -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
        
#src/main/java/com/synergy/SynergySphereApplication.java
package com.synergy;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SynergySphereApplication {
    public static void main(String[] args) {
        SpringApplication.run(SynergySphereApplication.class, args);
    }
}
package com.synergy.model;

import jakarta.persistence.*;
import java.util.List;

@Entity
public class Project {
    @Id
    @GeneratedValue
    private Long id;

    private String name;
    private String description;

    @OneToMany(mappedBy = "project", cascade = CascadeType.ALL)
    private List<Task> tasks;

    // getters and setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }
}
package com.synergy.model;

import jakarta.persistence.*;

@Entity
public class Task {
    @Id
    @GeneratedValue
    private Long id;

    private String title;
    private String status; // todo, inprogress, done

    @ManyToOne
    @JoinColumn(name = "project_id")
    private Project project;

    // getters/setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String getTitle() { return title; }
    public void setTitle(String title) { this.title = title; }

    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }

    public Project getProject() { return project; }
    public void setProject(Project project) { this.project = project; }
}
package com.synergy.repo;

import com.synergy.model.Project;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ProjectRepository extends JpaRepository<Project, Long> {}
package com.synergy.repo;

import com.synergy.model.Task;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;

public interface TaskRepository extends JpaRepository<Task, Long> {
    List<Task> findByProjectId(Long projectId);
}
package com.synergy.controller;

import com.synergy.model.Project;
import com.synergy.model.Task;
import com.synergy.repo.ProjectRepository;
import com.synergy.repo.TaskRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/projects")
@CrossOrigin
public class ProjectController {

    @Autowired
    private ProjectRepository projectRepo;

    @Autowired
    private TaskRepository taskRepo;

    @GetMapping
    public List<Project> allProjects() {
        return projectRepo.findAll();
    }

    @GetMapping("/{id}/tasks")
    public List<Task> getTasks(@PathVariable Long id) {
        return taskRepo.findByProjectId(id);
    }

    @PostMapping
    public Project createProject(@RequestBody Project p) {
        return projectRepo.save(p);
    }

    @PostMapping("/{id}/tasks")
    public Task addTask(@PathVariable Long id, @RequestBody Task task) {
        Project p = projectRepo.findById(id).orElseThrow();
        task.setProject(p);
        return taskRepo.save(task);
    }

    @PutMapping("/tasks/{taskId}")
    public Task updateTask(@PathVariable Long taskId, @RequestBody Task updates) {
        Task t = taskRepo.findById(taskId).orElseThrow();
        t.setStatus(updates.getStatus());
        return taskRepo.save(t);
    }
}
package com.synergy.ui;

import com.synergy.model.Project;
import com.synergy.model.Task;
import com.synergy.repo.ProjectRepository;
import com.synergy.repo.TaskRepository;
import com.vaadin.flow.component.button.Button;
import com.vaadin.flow.component.grid.Grid;
import com.vaadin.flow.component.html.H1;
import com.vaadin.flow.component.html.H2;
import com.vaadin.flow.component.orderedlayout.VerticalLayout;
import com.vaadin.flow.router.Route;
import org.springframework.beans.factory.annotation.Autowired;

@Route("")
public class MainView extends VerticalLayout {

    @Autowired
    private ProjectRepository projectRepo;
    @Autowired
    private TaskRepository taskRepo;

    private Grid<Project> projectGrid = new Grid<>(Project.class);
    private Grid<Task> taskGrid = new Grid<>(Task.class);

    public MainView(ProjectRepository projectRepo, TaskRepository taskRepo) {
        this.projectRepo = projectRepo;
        this.taskRepo = taskRepo;

        H1 header = new H1("Synergy Sphere");

        projectGrid.setItems(projectRepo.findAll());
        projectGrid.setColumns("id", "name", "description");
        projectGrid.asSingleSelect().addValueChangeListener(e -> {
            Project selected = e.getValue();
            if (selected != null) {
                showTasks(selected);
            }
        });

        add(header, projectGrid, taskGrid);
    }

    private void showTasks(Project project) {
        taskGrid.setItems(taskRepo.findByProjectId(project.getId()));
        taskGrid.setColumns("id", "title", "status");

        Button addTask = new Button("Add Task", event -> {
            Task t = new Task();
            t.setTitle("New Task");
            t.setStatus("todo");
            t.setProject(project);
            taskRepo.save(t);
            taskGrid.setItems(taskRepo.findByProjectId(project.getId()));
        });

        add(new H2("Tasks for " + project.getName()), taskGrid, addTask);
    }
}
spring.datasource.url=jdbc:h2:mem:synergydb
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true

spring.h2.console.enabled=true

    </build>
</project>
