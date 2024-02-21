---
sidebar_position: 1
---

# On pose les bases !

## Création du projet

Dans un premier temps nous allons créer un nouveau projet, pour ce faire vous connaissez la chanson : https://start.spring.io/

On choisit spring web JPA & mysql comme dépendences.

On ouvre le projet dans intellij IDE, on choisit le SKD 21, puis on build

lance la commande mvn spring-boot:run

## Créons l’architecture de notre APP !

Comme vous le savez, nous allons créer un back pour un blog, notre architecture à sa plus haute contiendra le nom des grandes features. Commencer par créer un package Article, qui contiendra lui ainsi que tout les futurs packages mères 3 dossiers.

Nous allonrs créer 3 dossiers, qui correspondent dans le jargon à des Layers (couches) :

- application (partie qui va nous servir à communiquer avec notre app : Request & Response)
- domain (le core de notre application qui contient notre logique métier)
- infrastructure (la logique dont nous avons besoin pour run notre application)

A l’intérieur de notre package application, nous allons y créer notre controller qui contiendra les methodes de notre CRUD

```js
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

```js
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

```js
package com.crud.demo.article.infrastructure.repository;

import com.crud.demo.article.domain.Article;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ArticleRepository extends JpaRepository<Article, Long> {
    
}
```

Nous allons nous servir de notre Repository dans notre controller, pour ce faire ⇒ une petite injection s’impose 💉 :

```js
import com.crud.demo.article.infrastructure.repository.ArticleRepository;

public class ArticleController {

    private final ArticleRepository repository;

    ArticleController(ArticleRepository repository) {
        this.repository = repository;
    }
//
```

Mettons à jour le reste de notre controller

```js
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

## Gestion d'erreurs dans le controller

`ArticleNotFoundException` mais d’où vient ce truc ?!
Nous souhaitons pouvoir gérer la manière dont les erreurs sont envoyées, pour ce faire nous allons créer deux classes : `ArticleNotFoundException` ainsi que `ArticleNotFoundAdvice`

La package infrastructure contiendra un nouveau package nommé exception qui aura le rôle d’accueillir toutes les classes en lien avec les erreurs.

```js
package com.crud.demo.article.infrastructure.exception;

public class ArticleNotFoundException extends RuntimeException {
    ArticleNotFoundException(Long id) {
        super("Could not find article with id " + id);
    }
}
```

```js
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
