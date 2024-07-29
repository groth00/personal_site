+++
title = "Projects"
description = "Projects"
date = "2024-07-29"
+++

**Go Web Forum**  
Tools: Go, PostgreSQL, htmx, Go's html/template package, Bulma CSS

- Designed and implemented a web forum allowing users to create posts and comments for topics
  - CRUD operations for topics, posts, and comments; like, dislike, and save posts and comments
  - Users register via a signup form then submit an activation token sent by email (SMTP)
  - Hierarchical data modeling of comment chains using PostgreSQL
  - Integrated with OpenTelemetry; instrumented HTTP handler functions to collect traces, metrics, and logs
  - Middleware: CSRF protection, secure HTTP response headers, CORS, user authentication and access control

**Go Interpreter and Compiler**  
References: Writing an Interpreter in Go & Writing a Compiler in Go (Thorsten Ball)  
Tools: Go

- Tools: Go
- Implemented a lexer and Pratt parser
- Implemented an evaluator for a tree-walking interpreter
- Implemented a stack-based VM and bytecode compiler

**Go REST API**  
Reference: Let's Go Further! (Alex Edwards)  
Tools: Go, PostgreSQL, pq, golang-migrate

- Implemented a REST API using Go to handle JSON HTTP requests and responses
  - CRUD operations: filter, sort, pagination, user registration, activation, and login
  - Database: PostgreSQL (pq), database migrations (golang-migrate)
  - Access control list for enforcing proper permissions for protecting routes
  - Middleware: logging, rate limiting, authentication, and authorization
  - Unit tests

**Go Website**  
Reference: Let's Go! (Alex Edwards)
Tools: Go, PostgreSQL, pq, golang-migrate

- Implemented a basic HTTPS website using Go to serve HTML, CSS, and Javascript,
- Utilized the repository pattern to separate application logic from data access
- Database: PostgresSQL, database migrations (golang-migrate)
- Middleware: logging, authentication, CSRF token handling, and setting secure headers
- Unit tests

**Rust REST API**  
References: Rust Web Development (Bastian Gruber)  
Tools: Rust, warp, PostgresSQL, sqlx

- Implemented a CRUD REST API using Rust and the warp framework
- Used PostgreSQL with the sqlx library and handled database migrations
- Rust unit testing for HTTP handlers
- Middleware: authentication with Paseto tokens and authorization

**Python Data Science APIs**  
References: Building Data Science Applications with FastAPI (François Voron)  
Tools: Python, FastAPI, sqlite3, SQLAlchemy, alembic, MongoDB, motor

- Implemented REST APIs using Python and FastAPI
  - Running a ML model for inferences
  - Storing posts and comments
  - Relational database implementation: sqlite3, SQLAlchemy, alembic
  - NoSQL database implementation: MongoDB, motor
