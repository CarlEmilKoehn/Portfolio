+++
date = '2026-02-14T17:15:16+01:00'
draft = false
title = 'Designing the Core Entities for Membolo'
description = 'A dev log about the early backend architecture of Membolo and the design of the core JPA entity classes.'
tags = ["java", "backend", "jpa", "hibernate", "postgres", "devlog"]
categories = ["projects", "membolo"]
+++

---

## Project Context

Membolo is a flashcard learning platform I am currently developing as part of a backend-focused project. The main idea behind the platform is to allow users to create collections of flashcards and optionally share them with others. Public collections can be browsed and used by anyone, while private collections remain personal study material.

At this stage of development the focus is purely on the backend architecture. Before implementing APIs or frontend interfaces, I wanted to design a clean domain model that represents how the application should work conceptually.

The current backend stack consists of:

* Java
* JPA with Hibernate
* PostgreSQL (running in Docker)
* Jackson for JSON serialization
* Javalin for the REST API layer (to be implemented later)

The first step in building this architecture was designing the entity classes that represent the core objects of the system.

---

## Identifying the Core Domain Objects

The functionality of the platform can largely be described through three core concepts:

* Users
* Collections
* Flashcards

A user creates collections, and each collection contains flashcards. This produces a natural hierarchical structure:

```text
User → Collection → Flashcard
```

Translating this to relational database relationships results in:

* One user can have many collections
* One collection can contain many flashcards
* Each flashcard belongs to exactly one collection

The entity classes were designed to reflect this hierarchy as directly as possible.

---

## The User Entity

The `User` entity represents an account that owns flashcard collections.

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String username;

    @Column(nullable = false, unique = true)
    private String email;

    @Column(nullable = false)
    private String password;

    @OneToMany(mappedBy = "owner", cascade = CascadeType.ALL)
    private List<Collection> collections = new ArrayList<>();

}
```

One of the first design decisions here was the use of a `List` to store collections rather than a `Map` or another structure.

Since collections are stored in a relational database table, each collection already has a unique primary key (`id`). Using a `Map` keyed by ID would duplicate functionality that the database already provides. A `List` is therefore sufficient and keeps the in-memory representation simple.

Another benefit of using a list is that ordering can later be introduced if needed (for example, allowing users to sort their collections manually or by creation date).

The relationship between users and collections is defined with `@OneToMany` and `mappedBy = "owner"`. This indicates that the `Collection` entity contains the actual foreign key column. The user entity simply represents the inverse side of the relationship.

The `cascade = CascadeType.ALL` option was chosen to simplify lifecycle management. If a user is deleted, their collections should also be removed automatically. Without cascading, this would require manual cleanup logic in the service layer.

---

## The Collection Entity

Collections act as containers for flashcards and belong to a specific user.

```java
@Entity
@Table(name = "collections")
public class Collection {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    private String description;

    @Column(name = "is_public")
    private boolean isPublic;

    @ManyToOne
    @JoinColumn(name = "owner_id", nullable = false)
    private User owner;

    @OneToMany(mappedBy = "collection", cascade = CascadeType.ALL)
    private List<Flashcard> flashcards = new ArrayList<>();

}
```

The `Collection` entity introduces the concept of **visibility** through the `isPublic` field. This will later allow the API to expose endpoints where users can search for public collections while keeping private collections restricted to their owners.

The ownership relationship is defined with `@ManyToOne`. Many collections can belong to a single user, so the database stores a foreign key (`owner_id`) pointing to the user table.

Again, flashcards are stored using a `List` rather than a `Set` or `Map`. At this stage there is no strict requirement that flashcards be unique within a collection, and maintaining insertion order can be useful for study sessions. A `List` therefore represents the most flexible choice.

Using a `Set` might enforce uniqueness, but uniqueness rules are better handled explicitly through application logic or database constraints rather than implicitly through the collection type.

Another small but important design decision is the use of `cascade = CascadeType.ALL` on the flashcard relationship. If a collection is deleted, all flashcards belonging to it should be removed as well. Cascading ensures that Hibernate performs this cleanup automatically.

One feature that may be introduced later is **collection cloning**. This would allow users to copy another user's public collection and modify it for their own use. Supporting this would likely require an additional field such as `originalCollectionId` to track where a cloned collection originated.

---

## The Flashcard Entity

Flashcards represent the individual learning items within a collection.

```java
@Entity
@Table(name = "flashcards")
public class Flashcard {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "question_text")
    private String questionText;

    @Column(name = "answer_text")
    private String answerText;

    @Column(name = "question_image_url")
    private String questionImageUrl;

    @Column(name = "answer_image_url")
    private String answerImageUrl;

    @ManyToOne
    @JoinColumn(name = "collection_id", nullable = false)
    private Collection collection;

}
```

Flashcards were designed to support both text-based and image-based learning. Instead of forcing a card to contain only text, each side of the flashcard can contain either text or an image URL.

An alternative approach would have been to store images directly in the database using binary data. However, this would significantly increase database size and complicate scaling. Storing image URLs instead keeps the database lightweight and allows images to be stored on disk or in external storage.

The relationship between flashcards and collections is implemented using `@ManyToOne`, with the foreign key stored as `collection_id`. This ensures that flashcards cannot exist independently of a collection.

Another small design consideration was keeping flashcards as a simple entity without additional abstractions. While more complex flashcard models exist (for example supporting multiple card types), the current goal is to keep the data model minimal while still allowing future extensions.

---

## Why the Entity Layer Matters

Spending time designing the entity layer early helps clarify the structure of the entire application. These classes effectively define the **domain model** of the system.

Once the entities are well structured, the rest of the backend architecture can build on top of them:

* DAO classes will handle database persistence
* Service classes will implement business logic
* Controllers will expose REST endpoints

This layered approach keeps responsibilities separated and makes the codebase easier to maintain as the project grows.

---

## Next Steps

With the entity classes defined, the next step will be implementing the **DAO layer** to handle persistence using Hibernate. These classes will provide the basic CRUD operations needed to store and retrieve users, collections, and flashcards.

Once persistence is working correctly, the project can move on to building service classes and REST endpoints with Javalin.

For now, the focus remains on ensuring the core domain model is simple, flexible, and capable of supporting the features planned for the platform.
