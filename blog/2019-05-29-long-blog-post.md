---
slug: ddd-blog
title: DDD blog
tags: [hello, docusaurus]
---

Dans un premier temps nous allons créer un nouveau projet, pour ce faire vous connaissez la chanson : https://start.spring.io/

On choisit spring web JPA & mysql comme dépendences.

On ouvre le projet dans intellij IDE, on choisit le SKD 21, puis on build

lance la commande mvn spring-boot:run

Créons l’architecture de notre APP !

Comme vous le savez, nous allons créer un back pour un blog, notre architecture à sa plus haute contiendra le nom des grandes features. Commencer par créer un package Article, qui contiendra lui ainsi que tout les futurs packages mères 3 dossiers.

Nous allonrs créer 3 dossiers, qui correspondent dans le jargon à des Layers (couches) :

- application (partie qui va nous servir à communiquer avec notre app : Request & Response)
- domain (le core de notre application qui contient notre logique métier)
- infrastructure (la logique dont nous avons besoin pour run notre application)

A l’intérieur de notre package application, nous allons y créer notre controller qui contiendra les methodes de notre CRUD

```java
package com.crud.demo.article.application;

import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
public class ArticleController {

    @GetMapping('/articles')
    List<Article> readAll () {}

    @GetMapping('/articles/{id}')
    Article readOne (@PathVariable Long id) {}

    @PostMapping('/articles')
    Article create (@RequestBody Article newArticle) {}

    @PutMapping('/articles/{id}')
    Article edit (@RequestBody Article newArticle, @PathVariable Long id) {}

    @DeleteMapping('/articles/{id}')
    Article delete (@PathVariable Long id) {}
    
}
```

- `@RestController` indicates that the data returned by each method will be written straight into the response body instead of rendering a template.

Maintenant, créeons dans le dossier domain un package Entity et ajoutons-lui cette Entity Article qui va représenter notre objet article

```java
package com.crud.demo.article.domain.Entity;

public class Article {
    private Long id;
    private string title;
    private string description;
    private LocalDateTime createdAt;
}
```

Nous avons besoin d’un repository pour gérer la communication avec la base de donnée.
À ton avis, où est-ce que devrait figurer notre ArticleRepository ?

Dans le dossier infrastructure pardi !

```java
package com.crud.demo.article.infrastructure.repository;

import com.crud.demo.article.domain.Article;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ArticleRepository extends JpaRepository<Article, Long> {
    
}
```

Nous allons nous servir de notre Repository dans notre controller, pour ce faire ⇒ une petite injection s’impose 💉 :

```java
import com.crud.demo.article.infrastructure.repository.ArticleRepository;

public class ArticleController {

    private final ArticleRepository repository;

    ArticleController(ArticleRepository repository) {
        this.repository = repository;
    }
//
```

Mettons à jour le reste de notre controller

```java

//
		@GetMapping("/articles")
    List<Article> readAll () {
        return repository.findAll();
    }

    @GetMapping("/articles/{id}")
    Article readOne (@PathVariable Long id) {
        return repository.findById(id)
                .orElseThrow(() -> new ArticleNotFoundException(id));;
    }

    @PostMapping("/articles")
    Article create (@RequestBody Article newArticle) {
        return repository.save(newArticle);
    }

    @PutMapping("/articles/{id}")
    Article edit (@RequestBody Article newArticle, @PathVariable Long id) {
        return repository.findById(id)
                .map(article -> {
                    article.setTitle(newArticle.getTitle());
                    article.setDescription(newArticle.getDescription());
                    return repository.save(article);
                })
                .orElseGet(() -> {
                    newArticle.setId(id);
                    return repository.save(newArticle);
                });
    }

    @DeleteMapping("/articles/{id}")
    void delete (@PathVariable Long id) {
        repository.deleteById(id);
    }
```

- An `ArticleRepository` is injected by constructor into the controller.
- We have routes for each operation (`@GetMapping`, `@PostMapping`, `@PutMapping` and `@DeleteMapping`, corresponding to HTTP `GET`, `POST`, `PUT`, and `DELETE` calls). (NOTE: It’s useful to read each method and understand what they do.)
- `ArticleNotFoundException` is an exception used to indicate when an article is looked up but not found.

`ArticleNotFoundException` mais d’où vient ce truc ?!
Nous souhaitons pouvoir gérer la manière dont les erreurs sont envoyées, pour ce faire nous allons créer deux classes : `ArticleNotFoundException` ainsi que `ArticleNotFoundAdvice`

La package infrastructure contiendra un nouveau package nommé exception qui aura le rôle d’accueillir toutes les classes en lien avec les erreurs.

```java
package com.crud.demo.article.infrastructure.exception;

public class ArticleNotFoundException extends RuntimeException {
    ArticleNotFoundException(Long id) {
        super("Could not find article with id " + id);
    }
}
```

```java
package com.crud.demo.article.infrastructure.exception;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.bind.annotation.ResponseStatus;

@ControllerAdvice
class ArticleNotFoundAdvice {

    @ResponseBody
    @ExceptionHandler(ArticleNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    String articleNotFoundHandler(ArticleNotFoundException ex) {
        return ex.getMessage();
    }
}
```

Super ! Passons désormais à la mise en place d’une des parties les plus centrales : la BDD !

On ajoute dans le le fichier [application.properties](http://application.properties) la paramètres de configuration de notre bdd

```java
spring.datasource.url=jdbc:mysql://localhost:3307/spring_jpa_crud
spring.datasource.username=goyave
spring.datasource.password=:goy@ve1418:

spring.jpa.show-sql=true

spring.jpa.properties.hibernate.jdbc.time_zone=Europe/Paris

spring.jpa.hibernate.ddl-auto=update
```

Retournons désormais sur notre entity afin de la décorer 🏅 

```java
@Entity
public class Article {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "title", nullable = false)
    private String title;
    
    @Column(name = "description", nullable = false)
    private String description;

    @Column(name = "createdAt", nullable = false)
    private final LocalDateTime createdAt;

```

Première décoration : `@Entity`  qui va servir à dire à Spring Data : “Ecoute moi bien l’ami, cette classe correspond à une Entity qui doit être convertie en Table et ses attributs en colonnes, bien compris ?” (un petit ton de coboye histoire de mettre un peu de pression)

Deuxième décoration : `@Id et GeneratedValue` pour préciser que ID est une clé primaire et lui attribuer une stratégie de Generation automatique.

Troisième décoration : `@Column` pour donner à chaque attribut un nom d’équivalence de colonne en BDD en ajoutant quelques précisions supplémentaires

Relançons notre serveur à petit coup de `mvn spring-boot:run` et voyons voir ce qui se trame dans la console :

> Hibernate: create table article (id bigint not null auto_increment, created_at datetime(6) not null, description varchar(255) not null, title varchar(255) not null, primary key (id)) engine=InnoDB
>

Excellent, mais quelle puissance 😱 ! Si tu observes maintenant l’état de ta BDD tu verras qu’une nouvelle table ainsi que ses colonnes ont été crées !

Let’s go, un petit test s’impose :

GET  `[localhost:8080/articles](http://localhost:8080/articles)` nous donne comme résultat une response 200 avec un array vide (normal la BDD est vide 😀 ), et si jamais on essaie avec `[localhost:8080/articles/4](http://localhost:8080/articles)`  on aura une 404 avec `Could not find article with id 4` .

Super, tout fonctionne !

Maintenant un petit POST avec dans le body :

```java
{
	"title": "Sujet santé",
	"description": "la santé c'est vraiment important bro"
}
```

SUPER ! J’ai bien un article qui vient d’être crée avec une 200

```java
{
	"id": 1,
	"title": "Sujet santé",
	"description": "la santé c'est vraiment important bro",
	"createdAt": "2023-12-08T18:06:48.037851"
}
```

Tu peux également tester un PUT et refaire un GET, tu verras que tout s’est déroulé à merveille 😀

Rajoutons maintenant une entity `Comment` qui va nous servir à lier des commentaires à nos articles, pour représenter cela en POO il nous faudra faire une relation OneToMany (un article peut avoir plusieurs commentaires et plusieurs commentaires peuvent avoir un seul article).

Commençons par créer notre Entity `Comment`

```java
package com.crud.demo.article.domain.entity;

import jakarta.persistence.*;

@Entity
public class Comment {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "content", nullable = false)
    private String content;

    public void setId(Long id) {
        this.id = id;
    }

    public Long getId() {
        return id;
    }

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }
}
```

Un article peut avoir plusieurs commentaires, pour représenter celà côté `Article` il nous faut un attribut comments de type List< Comment> avec l’annotation @OneToMany et le getter/setter qui va bien.

```java
@OneToMany(mappedBy = "article", cascade= CascadeType.REMOVE)
private List<Comment> comments;

public List<Comment> getComments() {
    return comments;
}

public void setComments(List<Comment> comments) {
    this.comments = comments;
}
```

Côté Comment plusieurs choses : le @ManyToOne ainsi que @JoinColumn

```java
@ManyToOne(fetch = FetchType.LAZY, cascade = CascadeType.REFRESH, optional = false)
@JoinColumn(name = "article_id", nullable = false)
private Article article;
```

on relance tout ça avec un p’ti coup de `mvn spring-boot:run`

```java
create table comment (id bigint not null auto_increment, content varchar(255) not null, article_id bigint not null, primary key (id)) engine=InnoDB
```

désormais on remarque que sur un GET /articles que notre body contient un comments []

```java
[
	{
		"id": 1,
		"title": "Sujet santée",
		"description": "la santé c'est vraiment important brooo",
		"comments": [],
		"createdAt": "2023-12-08T18:06:48.037851"
	}
]
```

Mhhh, une question se pose désormais, comment créer un commentaire ? 2 solutions :

- Faire un PUT de `/articles` en rajoutant un objet dans le array de comments
- Créer un controller qui va nous permettre de faire un CRUD de comment

Pour ce tuto, nous allons voir le plus rapide : le `PUT` .

Créons un CommentRepository dans notre package infrastructure.repository

```java
package com.crud.demo.article.infrastructure.repository;

import com.crud.demo.article.domain.entity.Comment;
import org.springframework.data.jpa.repository.JpaRepository;

public interface CommentRepository extends JpaRepository<Comment, Long> {}
```

Nous allons en avoir besoin dans notre Controller, pour se faire, une petite 💉 dans ce dernier s’impose !

```java
private final ArticleRepository ArticleRepository;
private final CommentRepository CommentRepository;

ArticleController(ArticleRepository ArticleRepository, CommentRepository CommentRepository) {
     this.ArticleRepository = ArticleRepository;
     this.CommentRepository = CommentRepository;
}
```

Dans notre méthode `edit` ajoutons une condition pour dire que si getComments() n’est pas vide, alors on créer et ajoute les comments à l’article

```java
@PutMapping("/articles/{id}")
Article edit (@RequestBody Article newArticle, @PathVariable Long id) {
        return ArticleRepository.findById(id)
                .map(article -> {
                    article.setTitle(newArticle.getTitle());
                    article.setDescription(newArticle.getDescription());

                    if (!newArticle.getComments().isEmpty()) {
                        for (Comment comment: newArticle.getComments()) {
                            comment.setArticle(article);
                            CommentRepository.save(comment);
                        }
                    }
                    return ArticleRepository.save(article);
                })
                .orElseGet(() -> {
                    newArticle.setId(id);
                    return ArticleRepository.save(newArticle);
                });
    }
```

Mhhhh, après avoir re-run notre app, une petite erreur nous éclate à la figure 🧨 , pourtant si je vais regarder du côté de ma BDD tout est ok 🧐.
L’erreur ici est dûe à la sérialization qui n’a pas fonctionnée correctement, nous faisons face à ce que l’on appelle une *“circular reference”*  en gros, une boucle infinie.

Cette boucle infinie vient du fait que nous essayons de serializer un Article qui contient des commentaires et dont les commentaires font référence à des articles et dont les articles font références à des commentaires … 🔄🎡🤯

Quelle est la solution pour éviter cela ? Et bien une simple annotation `@JsonIgnore` dans notre entity Comment nous permettra d’éviter ce big problème :) .

```java
@ManyToOne(fetch = FetchType.LAZY, cascade = CascadeType.REFRESH, optional = false)
@JoinColumn(name = "article_id", nullable = false)
@JsonIgnore
private Article article;
```

On re-run notre app, et là TADAM ! Tout fonctionne 😄 

Je te laisse créer un controller pour les commentaires, car côté front lorsque l’on devra modifier ou supprimer un commentaire il faudrait que cela puisse se faire sans passer par `/articles` .