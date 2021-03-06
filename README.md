![Logo](https://github.com/openrewrite/rewrite/raw/master/doc/logo-oss.png)
### Eliminate legacy Spring patterns. Automatically.

[![Build Status](https://circleci.com/gh/openrewrite/rewrite-spring.svg?style=shield)](https://circleci.com/gh/openrewrite/rewrite-spring)
[![Apache 2.0](https://img.shields.io/github/license/openrewrite/rewrite-spring.svg)](https://www.apache.org/licenses/LICENSE-2.0)
[![Maven Central](https://img.shields.io/maven-central/v/org.openrewrite.plan/rewrite-spring.svg)](https://mvnrepository.com/artifact/org.openrewrite.plan/rewrite-spring)

## Table of Contents

- [What is this?](#what-is-this-)
- [Spring XML to annotation-based configuration](#spring-xml-to-annotation-based-configuration)
- [Prefer constructor injection](#prefer-constructor-injection)
- [Don't use `@RequestMapping` on methods](#dont-use-requestmapping-on-methods)
- [Don't specify a name argument on annotations like `@PathVariable`](#dont-specify-a-name-argument-on-annotations-like-pathvariable)
- [`public` is not necessary on `@Bean` methods](#public-is-not-necessary-on-bean-methods)

## What is this?

This project implements a [Rewrite module](https://github.com/openrewrite/rewrite) to automatically apply best practices in Java Spring Boot applications. Originally inspired by the [Reboot](https://github.com/thanus/reboot) project.

## Spring XML to annotation-based configuration

See the [demo slides](https://slides.com/rewrite/spring) here of work-in-progress features related to migration off of XML configuration to annotation-based configuration.

## Prefer constructor injection

> "Always use constructor injection. Every time you use @Autowired field injection a unit test dies."
> (Josh Long)

Starting with Spring 4.3, `@Autowired` is no longer required on constructors. This made using tools like Lombok and IDEs to generate constructors off of required (final) fields generally more useful. Field injection fell out of favor as the preferred mode of dependency injection.

### Before

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class UsersController {
    @Autowired
    private UsersService usersService;

    @Autowired(required = false)
    NameService nameService; // visibility will be reduced to private

    public void setUsersService(UsersService usersService) {
        this.usersService = usersService;
    }

    ...
}
```

### After

1. Generate `@RequiredArgsConstructor` or generate a constructor, depending on whether the Lombok option is selected.
2. Remove `@Autowired` and `@javax.inject.Inject` annotations from fields.
3. Mark injected fields as `private final`.
4. Remove setters for injectable fields.
5. When JSR-305 is on the classpath, mark generated constructor parameters and fields with `@javax.annotation.Nonnull` when `@Autowired(required=false)`.
6. Add and remove imports as necessary.

### After (if Lombok option is selected)

```java
import javax.annotation.Nonnull;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequiredArgsConstructor
public class UsersController {
    private final UsersService usersService;

    @Nonnull // added if JSR305 option is selected
    private final NameService nameService;

    ...
}
```

### After (if Lombok option is NOT selected)

```java
import javax.annotation.Nonnull;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class UsersController {
    private final UsersService usersService;

    @Nonnull // added if JSR305 option is selected
    private final NameService nameService;

    public UsersController(UsersService usersService, @Nonnull NameService nameService) {
        this.usersService = usersService;
        this.nameService = nameService;
    }

    ...
}
```

## Don't use `@RequestMapping` on methods

`@RequestMapping(method = RequestMapping.GET)` is more verbose than the equivalent `@GetMapping` (ditto for the other request methods that have corresponding annotations).

Additionally, as described in https://find-sec-bugs.github.io/bugs.htm#SPRING_CSRF_UNRESTRICTED_REQUEST_MAPPING[this] security bug pattern, methods annotated with `@RequestMapping` are by default mapped to all the HTTP request methods, leaving your application exposed to a CSRF vulnerability.

### Before

```java
import java.util.*; // even if there is only one type reference, don't unfold!
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import static org.springframework.web.bind.annotation.RequestMethod.GET;
import static org.springframework.web.bind.annotation.RequestMethod.HEAD;

@RestController
@RequestMapping("/users")
public class UsersController {
    @RequestMapping(method = HEAD)
    public ResponseEntity<List<String>> getUsersHead() {
        return null;
    }

    @RequestMapping(method = GET)
    public ResponseEntity<List<String>> getUsers() {
        return null;
    }

    @RequestMapping(path = "/{id}", method = RequestMethod.GET)
    public ResponseEntity<String> getUser(@PathVariable("id") Long id) {
        return null;
    }

    @RequestMapping
    public ResponseEntity<List<String>> getUsersNoRequestMethod() {
        return null;
    }
}
```

### After

1. Replace `@RequestMapping` with `@GetMapping`, `@PostMapping`, etc.
2. `@RequestMapping(value = "/{id}")` simplifies to `@GetMapping("/{id}")` (i.e. drop the argument key "path"/"name").
3. Works for both `@RequestMapping(method = RequestMethod.GET)` and `@RequestMapping(method = GET)` (i.e. statically imported `RequestMethod` fields).
4. When method was the only annotation argument removes the remaining parentheses, i.e. simplify `@GetMapping()` to `@GetMapping`.
5. Add and remove imports as necessary, including managing static imports on `@RequestMethod` and folding/unfolding imports into stars as necessary.

```java
import java.util.*;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import static org.springframework.web.bind.annotation.RequestMethod.HEAD;

@RestController
@RequestMapping("/users")
public class UsersController {
    @RequestMapping(method = HEAD)
    public ResponseEntity<List<String>> getUsersHead() {
        return null;
    }

    @GetMapping
    public ResponseEntity<List<String>> getUsers() {
        return null;
    }

    @GetMapping("/{id}")
    public ResponseEntity<String> getUser(@PathVariable("id") Long id) {
        return null;
    }

    @GetMapping
    public ResponseEntity<List<String>> getUsersNoRequestMethod() {
        return null;
    }
}
```

## Don't specify a name argument on annotations like `@PathVariable`

Doing so is simply more verbose than is necessary, beginning with Spring 4.3.

### Before

```java
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/users")
public class UsersController {
    @GetMapping("/{id}")
    public ResponseEntity<String> getUser(@PathVariable("id") Long id,
                                          @PathVariable(required = false) Long p2,
                                          @PathVariable(value = "p3") Long anotherName) {
        if(anotherName % 42 == 0) {
            ...
        }
        ...
    }
}
```

### After

1. Removes name/value annotation attributes for:
    * `@PathVariable`
    * `@RequestParam`
    * `@RequestHeader`
    * `@RequestAttribute`
    * `@CookieValue`
    * `@ModelAttribute`
    * `@SessionAttribute`
2. If the annotation attribute contained a value that was different than the method parameter name, rename the method parameter name and replace references to it in the method body as necessary.

```java
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/users")
public class UsersController {
    @GetMapping("/{id}")
    public ResponseEntity<String> getUser(@PathVariable Long id,
                                          @PathVariable(required = false) Long p2,
                                          @PathVariable Long p3) {
        if(p3 % 42 == 0) {
            ...
        }
        ...
    }
}
```

## `public` is not necessary on `@Bean` methods

Default visibility on `@Bean` methods is supported and simpler.

## Remove unnecessary `@Autowired`

Generally we almost never need to use `@Autowired` anymore. It isn't needed on constructors or fields any longer. Currently, this rule removes `@Autowired` from constructors.