# PolyglotWeb stage A

This file documents my process reimplementing portions of PolyglotWeb in Go, Lua, PHP, and Rust.

## Setup

First step, of course, is gathering all of the necessary repositories and setting up my editor, before moving on to the rest of the requirements.

This is accomplished by just cloning all repositories: (making use of xargs to feed the URLs to git clone one by one)
```
echo https://github.com/Arshu/PolyglotWeb https://github.com/Arshu/PolyglotWeb-Lua https://github.com/Arshu/PolyglotWeb-PHP https://github.com/Arshu/PolyglotWeb-Go https://github.com/Arshu/PolyglotWeb-Rust https://github.com/Arshu/AssembleWebApps | xargs -n 1 git clone
```

Afterwards, I set up VSCode with the `golang.go`, `rust-lang.rust`, `DEVSENSE.phptools-vscode`, and `sumneko.lua` extensions.

## Requirements

The requirements listed in the Readme are the following:

> Build a minimal API backend with swagger definitions using the help of Cursor LLM in the 4 languages mocking the API request to https://arshuweb.fly.dev
> The post request swagger definitions are available in https://arshuweb.fly.dev/Scalar/v1
> Document the prompts used in LLM to generate the bootstrap code

First off, Cursor is an IDE and not just an LLM, and we've set out to use VSCode, so this requirement might be a bit tricky to meet. I still install Cursor in the background for experimentation though.

Later down, we see that we will be using an LLM to generate the bootstrap, so I suppose I'll be testing out a few of those. For starters, I'll be running DeepSeek locally (using Ollama no GPU), and then experimenting with the hosted version once registrations are back up (2024-01-30) -- and comparing that to the result by Cursor.

The Swagger definitions are simple enough. For starters, we would like to proxy all requests for HTML to the original C# API, since we won't have those working in the first version - but still work on implementing all the other APIs.

Those APIs include:
* The Access APIs: /GetToken, /Login, and /Logoff.
    * We can't see the result of GetToken in the Swagger definition, but it should be easy to figure out once we have our hands on a valid User/Password combination
* The JsonComponent APIs: /AppApi/JsonApi/*

As we need to store things, we would have to use a database. To figure out how that is set up for the C# application, I end up running it locally. Unfortunately, I have no clue what the initial username/password might be, and while I can see that e.g. `http://localhost:5000/Account/Register` and `http://localhost:5000/` return slightly different content, it's still no registration page. So, we are a bit out of luck there, but we'll assume SQLite for the moment and ask one of the LLM models to confirm in a second.

## First language - Go

For the first language, I opt to start from something I've had recent experience with (to be able to confirm that LLM output is decent), and is likely to be most representative of what the other languages might look like.

### Bootstrapping

I fire up Cursor, and start it on the Go repository. Deciding to be bold, I start it off with the following prompt in Composer / agent mode with claude 3.5 sonnet:
```
Create a bootstrap web API application in Go with login, that will run on fly.io
```

<details>
I'll help you create a modern Go web API application with authentication that's ready to deploy on fly.io. I'll structure this with clean architecture principles and use popular, well-maintained libraries. Let's start by setting up the project structure and core files.
First, let me check if there are any existing files in the workspace.

[..snip]

Great! Let's create a modern Go web API project. I'll set up the following:
1. A Go module with proper project structure
2. Main application with fiber web framework (fast and Express-like)
3. JWT authentication
4. SQLite database with GORM
5. Environment configuration
6. Dockerfile and fly.io configuration

Let's start by creating the main application structure and essential files:

[..snip]

Great! I've created a complete Go web API application with the following features:
1. JWT Authentication with login and registration
2. SQLite database with GORM
3. Protected routes
4. Docker support
5. fly.io deployment configuration
6. Environment variable configuration
7. Comprehensive documentation

The application structure is:
* main.go - Main application setup and routing
* models.go - Database models and initialization
* auth.go - Authentication handlers and middleware
* Dockerfile - Docker configuration
* fly.toml - fly.io deployment configuration
* .env - Environment variables
* README.md - Documentation
To get started locally:
* First, install the dependencies
* Make sure to update the JWT secret in the .env file with a secure value.
* Run the application
* The API will be available at http://localhost:3000 with the following endpoints:
    * POST /auth/register - Register a new user
    * POST /auth/login - Login and get JWT token
    * GET /api/profile - Get user profile (protected route)

[...snip]

</details>

It instantly spits out a plan that includes creating an SQLite database (with Gorm - doublechecking with https://blog.logrocket.com/comparing-orm-packages-go/, it looks like a decent default choice, and their [API](https://gorm.io/docs/models.html) is sufficiently Go-like too) - perhaps my guess was correct.

It somehow decides to throw out a go.mod file with dependencies filled out, which is odd, as one would usually use `go mod tidy` here instead (Especially, given the chance that the generated package versions are outdated/insecure).
It also makes the whole thing use Gofiber, which I would rather not learn at the moment.

It also decides to rewrite the README and add JWT authentication, despite that being a bad choice in my opinion, especially as we want to be handling logout too. I throw the following question into Deepseek-r1:7b to see what it'd say, and leave it thinking in the background:
```
Should one use JWT authentication in a Fly.io app?
```
It finally spits out an answer:
<details>

```
**Should One Use JWT Authentication in a Fly.io App?**

Yes, you should consider using JWT (JSON Web Token) authentication in your Fly.io app for several reasons:

1. **Security**: JWT is highly secure due to its multi-algorithm signing approach and can be combined with OAuth 2.0 if needed.

2. **Compatibility**: Fly.io supports JWT via Node.js and Go libraries, making integration straightforward.

3. **Multi-Provider Support**: You can integrate third-party services like Google or Auth0 for authenticating users without coding authentication yourself.

4. **Token Expiration**: Implementing token expiration adds an extra layer of security by limiting access unless re-authenticated.

5. **Efficiency**: JWT is lightweight and efficient, suitable for microservices architectures typical in Fly.io.

**Considerations:**

- **Token Management**: Generate a sufficient number (e.g., 10,000) to balance cost and coverage.
  
- **Trade-offs**: While secure and flexible, JWT may require more setup than simpler methods like password-based auth or Fly.io's OAuth features.

In summary, JWT is a robust choice for authentication in Fly.io, offering security, flexibility, and compatibility with existing tools.
```

</details>

But... between the Token Management part being completely nonsensical, Multi-Provider Support not being necessary for the current task, Fly.io's OAuth applying only to logging into the Fly.io platform (and not apps on top of it), supporting multiple signature algorithms likely [weakening security](https://auth0.com/blog/critical-vulnerabilities-in-json-web-token-libraries/#Great--So--What-s-Wrong-with-That-), the whole thing is a bit wonky.
I decide to re-prompt it with 
```
The application has a logout page. Does this change your answer?
```

But it just throws more garbage back, ignoring the [wisdom that JWT-s are about as useful as session cookies](http://cryto.net/~joepie91/blog/2016/06/19/stop-using-jwt-for-sessions-part-2-why-your-solution-doesnt-work/) when you add logout/session invalidation, but are much harder to implement.

Overall, Deepseek-r1:7b is not very technically versed. I might have to retest with a larger model later on.

Just to make sure, I throw a similar question into Cursor's chat:
```
I'd like to add logout later. Are JWTs useful in this case?
```
It gives a few ideas of how to implement logout (e.g. blacklisting tokens), but also suggests just using normal sessions. Success, I suppose!

Back to Cursor's composer, I feed it the following, without accepting the changes so far:
```
Can you remove gofiber and JWTs (use regular session cookies)?
```

<details>

```
I'll help you modify the application to use standard Go `net/http` with session cookies instead of JWT tokens. I'll use the `gorilla/sessions` package for secure cookie management. Let me make these changes:

[..snip]

I've converted the application from using Fiber and JWT tokens to using standard Go net/http with session-based authentication. Here are the key changes:
* Removed Fiber and JWT dependencies, added Gorilla Mux and Sessions
* Replaced Fiber handlers with standard http.HandlerFunc
* Implemented secure session-based authentication using encrypted cookies
* Added a logout endpoint
* Updated all JSON responses to use the standard encoding/json package
* Added proper session middleware for protected routes
* Updated environment variables to use SESSION_SECRET instead of JWT_SECRET
The application now uses secure, HttpOnly cookies for session management, which is more secure against XSS attacks and provides a better user experience since the client doesn't need to manage tokens.
Key security features of the session implementation:
* HttpOnly cookies (not accessible via JavaScript)
* Secure flag (only sent over HTTPS)
* SameSite strict mode (prevents CSRF attacks)
* 7-day session expiration
* Encrypted session data
```

</details>

It does as asked, and the codebase is at least a bit better now.. So I press the accept all button.

It still doesn't store session cookie details in the database - meaning, we lose sessions on server restart (or, worse, when moving between server instances). But- that won't be something that a non-developer would recognize; and developers used to other languages might have a hard time navigating the codebase to find that particular detail. Even for me, it took me a bit of browsing around and double-checking the `gorilla/sessions` documentation to confirm that Gorm can't possibly be doing something behind-the-scenes to enable session cookie storage in the database.

(It also enables Gorm's `AutoMigrate` function on the `User` model. This is alright for now, and very likely what we want to use, but as the app transitions into production, it might be worth investigating custom migrations—especially around renaming fields and such, as otherwise `AutoMigrate` will fail to keep track of the field.)

I throw in the relevant prompt still;
```
Can you store the sessions in the database?
```

<details>

```

I'll help you modify the application to store sessions in the SQLite database using GORM. This will make the sessions persistent and allow them to survive server restarts. We'll need to:

[...snip]

I've updated the application to store sessions in the SQLite database. Here are the key changes:
1. Added a Session model to store session data in the database
2. Created a custom DBStore that implements the sessions.Store interface
3. Added session cleanup to remove expired sessions
4. Updated the main application to use the database session store
5. Improved error handling in auth handlers
6. Added proper session deletion on logout

Key features of the database session store:
1. Persistence: Sessions survive server restarts
2. Automatic cleanup: Expired sessions are removed hourly
3. Secure: Session data is still encrypted using the session secret
4. Scalable: Can be easily modified to use other databases (PostgreSQL, MySQL, etc.)

[..snip]

```

</details>

I'm a bit wary of the fact that we are implementing a whole `gorilla/sessions.Store`, especially when there exist packages like [`wader/gormstore`](https://pkg.go.dev/github.com/wader/gormstore) (found through independent research). It adds too much complexity to a codebase that's meant to be a bootstrap only. Worse yet, the implemention implemented by Cursor/Claude is very similar to the one in `wader/gormstore` - with the same periodic cleanup idea - hence, making me doubt the benefit of having our own code for this.

I decide to be cheeky and throw in the following:
```
Use wader/gormstore
```

<details>

```
I'll help you switch to using wader/gormstore which is a well-maintained library specifically designed for storing Gorilla sessions in GORM. Let's make these changes:

[...snip]

I've updated the application to use wader/gormstore instead of our custom implementation. This brings several benefits:
1. Uses a well-tested and maintained library
2. Automatic session cleanup with configurable intervals
3. Better performance with proper database indexing
4. Built-in support for session encryption
5. Automatic table creation and management
The main changes are:
1. Removed our custom session store implementation
2. Removed our custom Session model
3. Added gormstore dependency
4. Updated main.go to use gormstore
5. Added automatic session cleanup with a 1-hour interval
The session management functionality remains the same, but it's now using a more robust and maintained implementation. The gormstore library will:

[...snip]

```

</details>

It instantly figures out the plan, and even praises `wader/gormstore` as being well-maintained. Success... but, maybe I should start next language session by gathering a list of dependencies before asking for bootstrap code. Otherwise, it feels like Cursor+Claude instantly jumps to writing code that is, at least for now, too complex and too long, for the task we need it for.

I save and commit everything, except for the README.md that it vandalized—this should be a decent bootstrap for now.

There are a few things I would have done differently if I was writing the code, given my experience with Go and web applications:
* I likely wouldn't have used `gorilla/mux` and potentially `gorilla/session` either, since I wouldn't have found out about them. Instead, I might have considered storing login sessions in the user model, as it remove the need to delete expired sessions.
* I would have used `spf13/cobra` for the `main.go` file, instead of hand-coding a main function. It's much easier to manage cancellations and CLI arguments with a library like it.
* Functions like `authMiddleware` and `loggingMiddleware` would probably either use defaults or be as trivial as possible.
* `initDB` would return `error`, since I like religiously passing errors around in Go. Also, all models would be in separate files, not in a jumbo `models.go` file.
* `User.Password` would be `User.PasswordHash` as this better reflects the data stored in the field.
* The `Dockerfile` would have used cache mounts to make rebuilds (much) faster.
* The LLM forgot to create the `.env.example` file it referenced in it's generated README.

Still, I'm impressed with the LLM's use of `bcrypt`—it's a stumbling block for beginners that it completely skips due to some kind of password hashing being used in every decent example out there.

### Fixing AI's issues

Next step is getting the build fixed. I clear out the generated `go.mod` and run `go mod tidy` to get all dependencies to newest. `go mod tidy` automatically detects all required modules, so despite what LLM-s suggest, there's never a need to modify a `go.mod` file manually.

I also sort the Dockerfile out, with caching, and using https://stackoverflow.com/a/58478169 to work around CGO-related problems in Alpine.

### Getting the right API endpoints

It's now time to modify the Go code to match the API definitions from Swagger.

In this case, we have three APIs:
* /Login — sets a session cookie. Parameters are passed in the query string (? Surprising, since usually passwords are passed in the request body of a POST request)
* /Logoff — destroys the session cookie.
* /GetToken — equivalent to /Login, but returns a token of some kind instead of setting a cookie, for easier programmatic usage.

For some reason, neither of the three quite works on the original C# backend, so I set out to reimplement them just based on the Swagger documentation—a few small tweaks to the bootstrapped APIs are what follow. The current frontend uses `/AppApi/DoWebValidateUser` at the moment, which might be the correct Login API, but that will have to be clarified.

### Getting Swagger in

Earlier, while bootstrapping, I forgot that I need Swagger support to fully match the C# backend.

I ask Deepseek (now the real chatbot), "What is a good library for Swagger in Golang?", and it suggests `swaggo`, along with a few framework-specific packages for Fiber, Chi, etc.
Further asking "What package can I use for Gorilla Mux?" nets me `http-swagger`, which is documented on [`swaggo/swag`'s README](https://github.com/swaggo/swag) as well.

I go ahead and import that package, along with adding the required annotation comments. I ad dthe generated files in the `docs/` folder to git as well, as my prior experience with Go packages is that I can't run any code generation when installing the package, hence the package would break if it is imported by someone else and there are missing folders in the Git repository.

### Proxying API requests

The final step is to proxy all unknown API requests hitting our server. After a bit of fiddling with the `gorilla/mux`, I find out about it's `NotFoundHandler` functionality, and with the help of [an article](https://dev.to/b0r/implement-reverse-proxy-in-gogolang-2cp4) (and good memory of having done that before for a different project), I make a reverse proxy to the original C# backend that can catch all other requests.

Ideally, I would also proxy Login/Logoff requests, and save the session cookie for the original backend in the session of the Go backend, thus syncing up the authentication between the two backends, but that will have to wait for proper Login support in the C# backend.

# TODO

Next steps for this file:
* Finish Go Stage A API (pending on fixed /Login and /GetToken in the C# backend)
* Do PHP Stage A API
* Do Rust Stage A API
* Do Lua Stage A API