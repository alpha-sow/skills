# Exemple : terminer une tâche

Une personne marque une tâche existante comme terminée. Règle métier : une tâche ne se termine qu’une fois. Une tâche inconnue est signalée sans créer de donnée.

Cet exemple illustre une séparation des responsabilités, pas une structure obligatoire. Java 17+, Spring Boot avec Spring MVC et Spring Data JPA ; seuls les fichiers utiles au parcours sont montrés (pas de `pom.xml`).

## 1. Arborescence

`task` est un module métier (un package), pas un module Maven.

```text
src/
├── main/java/com/example/tasks/
│   ├── TasksApplication.java
│   └── task/
│       ├── domain/
│       │   ├── model/Task.java
│       │   └── exception/TaskAlreadyCompletedException.java
│       ├── application/
│       │   ├── exception/TaskNotFoundException.java
│       │   ├── port/out/TaskRepository.java
│       │   └── usecase/CompleteTask.java
│       └── infrastructure/
│           ├── persistence/
│           │   ├── entity/TaskJpaEntity.java
│           │   ├── repository/SpringDataTaskRepository.java
│           │   ├── mapper/TaskMapper.java
│           │   └── adapter/JpaTaskRepositoryAdapter.java
│           ├── web/
│           │   ├── TaskController.java
│           │   └── TaskExceptionHandler.java
│           └── configuration/
│               ├── TaskConfiguration.java
│               └── TransactionalCompleteTask.java
└── test/java/com/example/tasks/task/
    ├── domain/model/TaskTest.java
    ├── application/usecase/CompleteTaskTest.java
    └── infrastructure/
        ├── web/TaskControllerTest.java
        └── persistence/adapter/JpaTaskRepositoryAdapterTest.java
```

`domain` et `application` n’importent ni Spring ni JPA. L’infrastructure fournit HTTP, persistance, assemblage et transaction.

## 2. Domaine

`domain/exception/TaskAlreadyCompletedException.java` :

```java
package com.example.tasks.task.domain.exception;

public class TaskAlreadyCompletedException extends RuntimeException {
    public TaskAlreadyCompletedException() {
        super("La tâche est déjà terminée");
    }
}
```

`domain/model/Task.java` — modèle immuable : terminer produit une nouvelle valeur.

```java
package com.example.tasks.task.domain.model;

import com.example.tasks.task.domain.exception.TaskAlreadyCompletedException;
import java.util.Objects;
import java.util.UUID;

public record Task(UUID id, boolean completed) {
    public Task {
        Objects.requireNonNull(id, "L’identifiant est obligatoire");
    }

    public Task complete() {
        if (completed) {
            throw new TaskAlreadyCompletedException();
        }
        return new Task(id, true);
    }
}
```

## 3. Application

`application/port/out/TaskRepository.java` — port sortant : `Optional.empty()` signifie tâche inconnue ; `save` met à jour la tâche de même identifiant. Il échange des `Task`, jamais des entités JPA.

```java
package com.example.tasks.task.application.port.out;

import com.example.tasks.task.domain.model.Task;
import java.util.Optional;
import java.util.UUID;

public interface TaskRepository {
    Optional<Task> findById(UUID id);

    void save(Task task);
}
```

`application/exception/TaskNotFoundException.java` :

```java
package com.example.tasks.task.application.exception;

public class TaskNotFoundException extends RuntimeException {
    public TaskNotFoundException() {
        super("Tâche inconnue");
    }
}
```

`application/usecase/CompleteTask.java` — charge, applique la règle, enregistre. Ni code HTTP, ni transaction, ni annotation Spring : la transaction est ajoutée par l’infrastructure (section 6).

```java
package com.example.tasks.task.application.usecase;

import com.example.tasks.task.application.exception.TaskNotFoundException;
import com.example.tasks.task.application.port.out.TaskRepository;
import com.example.tasks.task.domain.model.Task;
import java.util.Objects;
import java.util.UUID;

public class CompleteTask {
    private final TaskRepository repository;

    public CompleteTask(TaskRepository repository) {
        this.repository = Objects.requireNonNull(repository);
    }

    public void execute(UUID id) {
        Objects.requireNonNull(id, "L’identifiant est obligatoire");
        Task task = repository.findById(id)
                .orElseThrow(TaskNotFoundException::new);

        repository.save(task.complete());
    }
}
```

Si la tâche est absente ou déjà terminée, `save` n’est pas appelé.

## 4. Persistance

`infrastructure/persistence/entity/TaskJpaEntity.java` :

```java
package com.example.tasks.task.infrastructure.persistence.entity;

import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import java.util.UUID;

@Entity
@Table(name = "tasks")
public class TaskJpaEntity {
    @Id
    private UUID id;

    private boolean completed;

    protected TaskJpaEntity() {
    }

    public TaskJpaEntity(UUID id, boolean completed) {
        this.id = id;
        this.completed = completed;
    }

    public UUID getId() {
        return id;
    }

    public boolean isCompleted() {
        return completed;
    }
}
```

`infrastructure/persistence/repository/SpringDataTaskRepository.java` :

```java
package com.example.tasks.task.infrastructure.persistence.repository;

import com.example.tasks.task.infrastructure.persistence.entity.TaskJpaEntity;
import java.util.UUID;
import org.springframework.data.jpa.repository.JpaRepository;

public interface SpringDataTaskRepository extends JpaRepository<TaskJpaEntity, UUID> {
}
```

`infrastructure/persistence/mapper/TaskMapper.java` :

```java
package com.example.tasks.task.infrastructure.persistence.mapper;

import com.example.tasks.task.domain.model.Task;
import com.example.tasks.task.infrastructure.persistence.entity.TaskJpaEntity;
import org.springframework.stereotype.Component;

@Component
public class TaskMapper {
    public Task toDomain(TaskJpaEntity entity) {
        return new Task(entity.getId(), entity.isCompleted());
    }

    public TaskJpaEntity toEntity(Task task) {
        return new TaskJpaEntity(task.id(), task.completed());
    }
}
```

`infrastructure/persistence/adapter/JpaTaskRepositoryAdapter.java` — implémente le port :

```java
package com.example.tasks.task.infrastructure.persistence.adapter;

import com.example.tasks.task.application.port.out.TaskRepository;
import com.example.tasks.task.domain.model.Task;
import com.example.tasks.task.infrastructure.persistence.mapper.TaskMapper;
import com.example.tasks.task.infrastructure.persistence.repository.SpringDataTaskRepository;
import java.util.Optional;
import java.util.UUID;
import org.springframework.stereotype.Component;

@Component
public class JpaTaskRepositoryAdapter implements TaskRepository {
    private final SpringDataTaskRepository repository;
    private final TaskMapper mapper;

    public JpaTaskRepositoryAdapter(SpringDataTaskRepository repository, TaskMapper mapper) {
        this.repository = repository;
        this.mapper = mapper;
    }

    @Override
    public Optional<Task> findById(UUID id) {
        return repository.findById(id).map(mapper::toDomain);
    }

    @Override
    public void save(Task task) {
        repository.save(mapper.toEntity(task));
    }
}
```

Avec un UUID assigné et sans `@Version`, Spring Data traite l’entité comme existante : `save` fait un `merge`. Un futur cas de création devra définir sa détection des nouvelles entités, par exemple avec `Persistable.isNew()`.

Dépendances : `spring-boot-starter-data-jpa` et, à l’exécution, `com.h2database:h2`, versions gérées par Spring Boot.

## 5. Web

`infrastructure/web/TaskController.java` :

```java
package com.example.tasks.task.infrastructure.web;

import com.example.tasks.task.application.usecase.CompleteTask;
import java.util.UUID;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/tasks")
public class TaskController {
    private final CompleteTask completeTask;

    public TaskController(CompleteTask completeTask) {
        this.completeTask = completeTask;
    }

    @PostMapping("/{id}/complete")
    public ResponseEntity<Void> complete(@PathVariable("id") UUID id) {
        completeTask.execute(id);
        return ResponseEntity.noContent().build();
    }
}
```

`infrastructure/web/TaskExceptionHandler.java` — traduit uniquement les deux erreurs dédiées, sans exposer leur message :

```java
package com.example.tasks.task.infrastructure.web;

import com.example.tasks.task.application.exception.TaskNotFoundException;
import com.example.tasks.task.domain.exception.TaskAlreadyCompletedException;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@RestControllerAdvice(assignableTypes = TaskController.class)
public class TaskExceptionHandler {
    @ExceptionHandler(TaskNotFoundException.class)
    public ResponseEntity<Void> handleTaskNotFound() {
        return ResponseEntity.notFound().build();
    }

    @ExceptionHandler(TaskAlreadyCompletedException.class)
    public ResponseEntity<Void> handleTaskAlreadyCompleted() {
        return ResponseEntity.status(HttpStatus.CONFLICT).build();
    }
}
```

Réponses : 204 en cas de succès, 404 si la tâche est inconnue, 409 si elle est déjà terminée, 400 si l’identifiant n’est pas un UUID (conversion Spring MVC).

## 6. Assemblage et transaction

`infrastructure/configuration/TransactionalCompleteTask.java` — décorateur qui rend le chargement et l’enregistrement atomiques sans annotation dans `application` :

```java
package com.example.tasks.task.infrastructure.configuration;

import com.example.tasks.task.application.port.out.TaskRepository;
import com.example.tasks.task.application.usecase.CompleteTask;
import java.util.UUID;
import org.springframework.transaction.support.TransactionTemplate;

class TransactionalCompleteTask extends CompleteTask {
    private final TransactionTemplate transaction;

    TransactionalCompleteTask(TaskRepository repository, TransactionTemplate transaction) {
        super(repository);
        this.transaction = transaction;
    }

    @Override
    public void execute(UUID id) {
        // Une exception non vérifiée, dont les erreurs métier, annule la transaction.
        transaction.executeWithoutResult(status -> super.execute(id));
    }
}
```

`infrastructure/configuration/TaskConfiguration.java` — déclare le cas d’utilisation ; Spring Boot fournit `TransactionTemplate` avec JPA.

```java
package com.example.tasks.task.infrastructure.configuration;

import com.example.tasks.task.application.port.out.TaskRepository;
import com.example.tasks.task.application.usecase.CompleteTask;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.transaction.support.TransactionTemplate;

@Configuration
public class TaskConfiguration {
    @Bean
    public CompleteTask completeTask(TaskRepository repository, TransactionTemplate transaction) {
        return new TransactionalCompleteTask(repository, transaction);
    }
}
```

`TasksApplication.java`, au package racine, permet la détection de tous les sous-packages :

```java
package com.example.tasks;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class TasksApplication {
    public static void main(String[] args) {
        SpringApplication.run(TasksApplication.class, args);
    }
}
```

Flux d’appel :

```text
Requête HTTP → TaskController → TransactionalCompleteTask (transaction)
                              → CompleteTask → TaskRepository.findById
                                             → Task.complete
                                             → TaskRepository.save
Port TaskRepository → JpaTaskRepositoryAdapter → TaskMapper / SpringDataTaskRepository → H2
```

Les dépendances du code vont dans l’autre sens : web et persistance dépendent de l’application, l’application du domaine ; le domaine ne dépend de rien d’extérieur.

## 7. Tests

`TaskTest.java` — règle métier seule (JUnit Jupiter, AssertJ) :

```java
package com.example.tasks.task.domain.model;

import com.example.tasks.task.domain.exception.TaskAlreadyCompletedException;
import java.util.UUID;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class TaskTest {
    @Test
    void termineUneTacheUneSeuleFois() {
        Task initial = new Task(UUID.randomUUID(), false);

        Task completed = initial.complete();

        assertThat(completed).isEqualTo(new Task(initial.id(), true));
        assertThat(initial.completed()).isFalse();
        assertThatThrownBy(completed::complete)
                .isInstanceOf(TaskAlreadyCompletedException.class);
    }
}
```

`CompleteTaskTest.java` — orchestration sans Spring ni base, avec un faux repository :

```java
package com.example.tasks.task.application.usecase;

import com.example.tasks.task.application.exception.TaskNotFoundException;
import com.example.tasks.task.application.port.out.TaskRepository;
import com.example.tasks.task.domain.exception.TaskAlreadyCompletedException;
import com.example.tasks.task.domain.model.Task;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.UUID;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class CompleteTaskTest {
    private final UUID id = UUID.randomUUID();

    @Test
    void enregistreLaTacheTerminee() {
        FakeTaskRepository repository = new FakeTaskRepository(new Task(id, false));

        new CompleteTask(repository).execute(id);

        assertThat(repository.saved).containsExactly(new Task(id, true));
    }

    @Test
    void refuseUneTacheAbsenteSansEnregistrer() {
        FakeTaskRepository repository = new FakeTaskRepository(null);

        assertThatThrownBy(() -> new CompleteTask(repository).execute(id))
                .isInstanceOf(TaskNotFoundException.class);
        assertThat(repository.saved).isEmpty();
    }

    @Test
    void refuseUneTacheDejaTermineeSansEnregistrer() {
        FakeTaskRepository repository = new FakeTaskRepository(new Task(id, true));

        assertThatThrownBy(() -> new CompleteTask(repository).execute(id))
                .isInstanceOf(TaskAlreadyCompletedException.class);
        assertThat(repository.saved).isEmpty();
    }

    private static class FakeTaskRepository implements TaskRepository {
        private final Task stored;
        private final List<Task> saved = new ArrayList<>();

        private FakeTaskRepository(Task stored) {
            this.stored = stored;
        }

        @Override
        public Optional<Task> findById(UUID id) {
            return Optional.ofNullable(stored).filter(task -> task.id().equals(id));
        }

        @Override
        public void save(Task task) {
            saved.add(task);
        }
    }
}
```

`TaskControllerTest.java` — frontière web seule : `@WebMvcTest` charge le contrôleur et l’advice, `@MockitoBean` remplace le cas d’utilisation. Imports pour Spring Boot 4 ; sous Boot 3.4/3.5, `WebMvcTest` est dans `org.springframework.boot.test.autoconfigure.web.servlet`.

```java
package com.example.tasks.task.infrastructure.web;

import com.example.tasks.task.application.exception.TaskNotFoundException;
import com.example.tasks.task.application.usecase.CompleteTask;
import com.example.tasks.task.domain.exception.TaskAlreadyCompletedException;
import java.util.UUID;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.web.servlet.MockMvc;
import static org.mockito.Mockito.doThrow;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.verifyNoInteractions;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@WebMvcTest(TaskController.class)
class TaskControllerTest {
    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private CompleteTask completeTask;

    private final UUID id = UUID.randomUUID();

    @Test
    void renvoie204() throws Exception {
        mockMvc.perform(post("/tasks/{id}/complete", id))
                .andExpect(status().isNoContent());
        verify(completeTask).execute(id);
    }

    @Test
    void renvoie404SansCorpsQuandLaTacheEstInconnue() throws Exception {
        doThrow(new TaskNotFoundException()).when(completeTask).execute(id);

        mockMvc.perform(post("/tasks/{id}/complete", id))
                .andExpect(status().isNotFound())
                .andExpect(content().string(""));
    }

    @Test
    void renvoie409SansCorpsQuandLaTacheEstDejaTerminee() throws Exception {
        doThrow(new TaskAlreadyCompletedException()).when(completeTask).execute(id);

        mockMvc.perform(post("/tasks/{id}/complete", id))
                .andExpect(status().isConflict())
                .andExpect(content().string(""));
    }

    @Test
    void renvoie400QuandIdentifiantEstMalForme() throws Exception {
        mockMvc.perform(post("/tasks/{id}/complete", "pas-un-uuid"))
                .andExpect(status().isBadRequest());
        verifyNoInteractions(completeTask);
    }
}
```

`JpaTaskRepositoryAdapterTest.java` — adaptateur avec H2 embarquée ; `@DataJpaTest` n’analyse pas les `@Component`, d’où `@Import`. Sous Boot 3.4/3.5, `DataJpaTest` est dans `org.springframework.boot.test.autoconfigure.orm.jpa`.

```java
package com.example.tasks.task.infrastructure.persistence.adapter;

import com.example.tasks.task.domain.model.Task;
import com.example.tasks.task.infrastructure.persistence.mapper.TaskMapper;
import jakarta.persistence.EntityManager;
import java.util.UUID;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest;
import org.springframework.context.annotation.Import;
import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest
@Import({JpaTaskRepositoryAdapter.class, TaskMapper.class})
class JpaTaskRepositoryAdapterTest {
    @Autowired
    private JpaTaskRepositoryAdapter repository;

    @Autowired
    private EntityManager entityManager;

    @Test
    void enregistreRelitEtMetAJourUneTache() {
        Task initial = new Task(UUID.randomUUID(), false);

        repository.save(initial);
        // flush + clear : relecture depuis la base, pas depuis le cache.
        entityManager.flush();
        entityManager.clear();
        assertThat(repository.findById(initial.id())).contains(initial);

        repository.save(initial.complete());
        entityManager.flush();
        entityManager.clear();
        assertThat(repository.findById(initial.id())).contains(new Task(initial.id(), true));
        assertThat(repository.findById(UUID.randomUUID())).isEmpty();
    }
}
```

## Limites

L’exemple ne couvre ni la création de tâche, ni les mises à jour concurrentes (deux requêtes simultanées peuvent terminer la même tâche ; `@Version` ou un verrou l’empêcherait), ni les droits d’accès, ni le test de la transaction assemblée.
