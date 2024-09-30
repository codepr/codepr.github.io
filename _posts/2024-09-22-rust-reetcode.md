---
layout: post
title: "Rust code platform - part 1"
description: ""
categories: rust backend database
---

Small project idea to have some fun with Rust and try to build something as
close as possible to being production-ready. This project also aims to dabble a
bit with Rust on the web backend side.

### Constraints

- Minimal requirements
- Observability

The idea I was thinking about is a simplified coding platform in the neetcode
style. The system will be seeded with a set of problems, users can register,
admin users can also upload new problems (although authentication and
authorization may be out of scope for the time being). Problems can be run
inside self-contained environments (think about Docker containers, maybe
providing a subset of pre-built images for each programming language offered,
C, C++, Python and Go can be a good starting point).

### Tech stack

Rust is the chosen language for this project, I'm not really familiar yet with
the current trends and best practices, last time I checked `actix` and `sqlx`
were the backbone of any backend application. After some recognition work, this
is what I came up with:

- Actix
- Postgres
- Prometheus
- Grafana

We're probably gonna use Docker and `docker-compose` to run the entire environment
locally in dev, it's a convenient way to emulate a staging/prod environment without
the hassle of deploying stuff to an Heroku dyno or any other cloud provider.

### Data model

The main entity in the application is represented by the problems. They will
include a title, a description and some metadata such as the category, the
difficulty and a skeleton based on the language, something like

```json
{
    title: "Two Sum",
    description: "Given an array of integers `nums` and an integer `target`, return the integers `i` and `j`  such that `nums[i] + nums[j] = target`..",
    categories: ["arrays"],
    difficulty: "easy",
    starting_snippets: {
        "c": "int *two_sum(int *nums, size_t len, int target) ..",
        "python": "def two_sum(nums, target)..."
    },
    solved: "false"
}
```

To start very simple they could be stored in a JSON or YAML static file on
a filesystem, I prefer to store them in the DB to allow easier query
capabilities and at a later stage, extensibility and upload of new problems.
Every problem can have multiple solutions clearly, so that would be another
table to be joined as a `has_many` relationship with the problem.

To cut short, a loosely detailed definition of the schemas we're going to need:

- User
    - email: string
    - nickname: string
    - **has_many**(Problem)
- Problem
    - title: string
    - description: string
    - categories: list(string)
    - difficulty: string
    - starting_snippets: map
    - test_cases: list(string)
    - solved: boolean
    - editorial: map
    - **has_many**(Solution)
- Solution
    - description: string
    - sinppet: string

Keeping things simple as a first draft, we can assume that the images
repositories with various metadata (e.g. languages versions, OS etc) can be
stored locally in a static JSON or YAML file.

### APIs

It will be a pure REST application, no frontend at all, but it should provide
all the required to be easily integrated in a JS app.

**ASSUMPION** we can start with a set of problems and editorial pre-seeded in
the system, on a later stage we can decide to increment the functionalities and
allow users roles, auth, and upload of new problems, contests maybe and notifications
to subscribed users.

#### User management
- **POST**  `/v1/users`
- **GET**   `/v1/users/:user_id`

#### Problems management
- **GET**   `/v1/problems/:problem_id`
- **GET**   `/v1/problems`
- **POST**  `/v1/problems/:problem_id`

