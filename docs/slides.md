name: title
class: middle

# Yet another package for<br/>multi-tenancy in Django

Lorenzo Peña &middot; @lorinkoz

---

## The agenda

1. Django in 2020
2. Django, multi-tenancy and you
3. The challanges ahead
    - The active tenant
    - Database, models and managers
    - Requests and URL reversing
    - The scope of everything else
4. Yet another package for this

---

class: middle
layout: false

# Django in 2020

---

## 2020 is going great so far

.center[![Dinosaur in traffic with meteors falling](images/dinosaur-traffic.jpg)]

---

layout: true

## But so is Django <small>(no... seriously)</small>

![Django motto](images/django.png)

---

-   Mature, solid and battle tested.
-   For perfectionists (like you)
-   For people with deadlines (like you)
-   Vast ecosystem - over 4k projects.ref[1]

.bottom[
.footnote[.ref[1] As listed in https://djangopackages.org]
]

---

-   Bright minds are second-guessing the modern web.ref[1]
-   People are doing email services today in vanilla monolith makers.ref[2]

.bottom[
.footnote[.ref[1] https://macwright.org/2020/05/10/spa-fatigue.html]
.footnote[.ref[2] https://twitter.com/dhh/status/1275901955995385856?s=20]
]

---

class: middle
layout: false

# Django, multi-tenancy and you

--

What do Django, multi-tenancy and you have in common?

---

## I have an hypothesis

--

The world is divided in two kinds of Djangonauts:

-   Those who **have done** multi-tenancy.
-   Those who **will be doing** multi-tenancy anytime soon.

--

.box[🔥 Hot take?]

--

.right[Well...]

---

.top[
![Reddit question about multi-tenancy in Django](images/django-mt-reddit.png)
]

---

.top[
![Reddit question about multi-tenancy in Django](images/django-mt-reddit-1.png)
]

.box[Sooner or later, you're going to have a<br/>🤑 multi-million dollar idea]

---

.top[
![Reddit question about multi-tenancy in Django](images/django-mt-reddit-2.png)
]
.box[And you're going to need some form of<br/>🏘️ multi-tenancy for it]

---

.top[
![Reddit question about multi-tenancy in Django](images/django-mt-reddit-3.png)
]

.box[And as crazy as it may sound, in order to implement it, you're going to use 🥁 Django]

---

.top[
![Reddit question about multi-tenancy in Django](images/django-mt-reddit-4.png)
]

.box[So let's face it:<br/>🔥 This post may as well have been made by you]

---

class: center middle
layout: false

## Take the .red[red] pill, Neo

![Morpheus watching as Neo decides whether or not to take the red pill](images/neo-red-pill.png)

---

## What is multi-tenancy

Software architecture in which a single instance of software runs on a server and serves multiple tenants.

A tenant is a group of users who share a common access with specific privileges to the software instance..ref[1]

.bottom[
.footnote[.ref[1] https://en.wikipedia.org/wiki/Multitenancy]
]

---

layout: true

## Types of multi-tenancy

---

Users exist **within** the context of tenants

.center[![Diagram of users inside tenants](images/diagram-users-in-tenants.png)]

---

Users exist **outside** the context of tenants

.center[![Diagram of users outside tenants](images/diagram-users-out-tenants.png)]

---

Users are equivalent to tenants (similar to single-tenancy)

.center[![Diagram of users equalling tenants](images/diagram-users-equal-tenants.png)]

---

layout: false

## How do we get to it?

Can I just start hacking my project?

--

Yes, but...

.warning[🤹 There are many things to do, and do right]
.warning[🤯 There are non trivial parts]

---

## Package first?

-   There is a number of solid packages to help with the multi-tenancy problem in Django.
-   The whole idea of this talk came from my experience forking one of those packages.

--

.box[✋ But let's not take a package-first approach]

--

Instead, let's pretend we're going to implement multi-tenancy from scratch, without the help of any package.

---

class: middle
layout: false

# The challenges ahead

---

layout: true

## The active tenant

---

We have to adjust our mindset:

.box[Most operations will now require a tenant to be considered 😎 **the active tenant**]

---

There are things that are currently tracked by Django as an **active something** independent of the request / response cycle. For instance: **timezone** and **language**.

We could use the same logic to also track the **active tenant**, so that we get at our disposal:

```python
tenant = get_active_tenant()
set_active_tenant(tenant)
```

---

.warning[🤔 What if, for some operation, there is **no active tenant**?]

We will have to answer these questions in a case by case basis:

-   Does this operation have sense in a tenant agnostic way?
-   Should we interpret the lack of tenant as an indication that the operation must be performed on all tenants?
-   Is the lack of a tenant a bug in this context?

---

Every part of the framework needs to be able to operate in the scope of the active tenant:

.left-column[

-   Database access
-   URL reversing
-   Admin interface
-   Cache
    ]

.right-column[

-   Channels (websockets)
-   Management commands
-   Celery tasks
-   File storage
    ]

And everything else...

---

class: middle
layout: false

# Database, models and managers

---

layout: true

## Database architecture

---

.left-column-66[**Isolated:** Multiple databases, one per tenant]
.right-column-33[![Diagram of isolated tenants](images/diagram-isolated.png)]

---

.left-column-66[**Shared:** One database, tenant column on (almost) every table]
.right-column-33[![Diagram of shared database](images/diagram-shared.png)]

---

.left-column-66[**Semi-isolated:** One database, one schema per tenant (PostgreSQL)]
.right-column-33[![Diagram of semi-isolated tenants](images/diagram-semi-isolated.png)]

---

layout: true

## Isolated databases

---

Multi-database configuration in Django settings

```python
DATABASES = {
    "default": {...},
    "tenant1": {...},
    "tenant2": {...},
    "tenant3": {...},
    ...
}
```

---

Queries need to define the active tenant.

```python
order = Order(...)
order.save(using="tenant1")

Order.objects.using("tenant2").filter(...)
Order.objects.db_manager("tenant2").do_something(...)
```

--

.box[🙋 How to control queries outside of your code?]

---

Django has a thing called database routers

```python
class IsolatedTenantsRouter:

    def db_for_read(self, model, **hints):
        active_tenant = get_active_tenant()
        return get_database_for_tenant(active_tenant)

    def db_for_write(self, model, **hints):
        active_tenant = get_active_tenant()
        return get_database_for_tenant(active_tenant)
```

---

**.red[Limitations]**

-   No cross-tenant relations.
-   No relation between tenants and shared data.
-   Increased costs of deployment.
-   Adding tenants require reconfiguring the project.

---

-   Do it if you expect a number of tenants in the lower tens.
-   Or if you can afford to buy a new Lamborghini for every tenant you get.

.right[![Three Lamborghinis](images/lamborghini.png)]

---

layout: true

## Shared database

---

All tenant-specific models require a FK to the model that controls the tenants:

```python
class SharedTenantModel(models.Model):

    tenant = models.ForeignKey(
        "TenantModel",
        on_delete=models.PROTECT,  # No easy tenant deletion
        related_name="%(class)ss"
    )

    class Meta:
        abstract = True
```

---

Assign active tenant before creating a model instance:

```python
order = Order(...)
order.tenant = get_active_tenant()
order.save()
```

---

Use active tenant in all queries:

```python
# In regular queries
Order.objects.create(tenant=get_active_tenant(), ...)
Order.objects.filter(tenant=get_active_tenant(), ...)

# But also in related queries
some_customer.orders.filter(
    order__tenant=get_active_tenant(),
    ...
)
```

---

Set active tenant when saving a form / serializer:

```python
# Form
instance = OrderForm.save(commit=False)
instance.tenant = get_active_tenant()
instance.save()

# DRF Serializer
instance = serializer.save(tenant=get_active_tenant())
```

---

This quickly escalates to:

.center[![Django motto with perfectionists replaced with burned out people](images/django-burned-out.png)]

--

The good news is:

.box[It's possible to automatically inject the tenant in most use cases, but this requires <u>additional wizardry</u>]

---

**.green[Recommendations]**

-   Put all your tenant specific queries in a single place.
-   Unit test each one of them, and make the test suite fail if any query is untested.

.box[🧸 Tests are a soft pillow]

---

layout: true

## Semi-isolated database

---

Use PostgreSQL schemas.ref[1] to isolate tenants within a single database.

.bottom[
.footnote[.ref[1] https://www.postgresql.org/docs/9.1/ddl-schemas.html]
]

--

.warning[⚠️ HYPE WARNING]

---

The concept of `search_path` allows for interesting combinations of isolated and shared data.

--

.left-column[
With `search_path=tenant1,shared` tables are searched in the schema of tenant 1 first, then in the shared schema
]
.right-column[![Diagram of schemas](images/schemas.png)]

---

Requires a custom database backend in order to set `search_path` based on active tenant:

```python
from django.db.backends.postgresql import base as postgresql
class DatabaseWrapper(postgresql.DatabaseWrapper):
    def _cursor(self, name=None):  # Over simplified !!!
        cursor = super()._cursor(name=name)
        tenant = get_active_tenant()
        schemas = get_schemas_from_tenant(tenant)
        search_path = ",".join(schemas)
        cursor.execute(f"SET search_path = {search_path}")
        return cursor
```

---

Requires a database router in order to control which models are migrated on which schemas.

```python
class SemiIsolatedTenantsRouter:
    def allow_migrate(self, db, app_label, model_name=None,
                      **hints):
        tenant = get_active_tenant()
        if tenant is not None:
*           return app_is_tenant_specific(app_label)
*       return app_is_shared(app_label)
```

---

**.green[Benefits]**

-   You get tenant isolation out of the box.
-   Easiest path to adapt an existing codebase.

**.red[Limitations]**

-   Extra care to define shared apps and tenant specific apps.
-   Extra care to define where to put users, sessions and content types.

---

.center[![Meme of crazy lady and cat about schemas and migrations](images/cat-lady-meme-schemas.png)]

---

layout: false

## Does it scale?

--

.warning[🔥 Yes and no!]

--

.left-column-66[Hit me in the Q/A because this deserves more than a slide...]
.right-column-33[.right[![Young boy after food fight](images/food-fight.png)]]

---

class: middle
layout: false

# Requests and URL reversing

---

layout: true

## Activating a tenant from the incoming request

---

-   Inferred from the user
-   Stored in the session
-   Specified in the headers
-   Specified in the URL
    -   Via subdomain
    -   Via subfolder
    -   Via query parameter

.box[A tenant can be activated from an incoming request via middleware]

---

```python
def TenantFromSessionMiddleware(get_response):
    def middleware(request):
*       tenant = get_tenant_from_session(request.session)
        if tenant:
            set_active_tenant(tenant)
        return get_response(request)
    return middleware
```

.box[Middleware with different retrieval methods could be chained]

---

.warning[The order of middleware is important!]

-   If it depends on the session it must go after `SessionMiddleware`.
-   If it depends on the user it must go after `AuthenticationMiddleware`.
-   If users depend on the tenant, then the tenant middleware must go before `AuthenticationMiddleware`.

---

layout: true

## Reversing tenant-aware URLs

---

A bigger challenge actually is:

.box[How to reverse URLs in Django so that the tenant is included?]

For some cases it's simply not possible:

-   Inferred from the user
-   Stored in the session
-   Specified in the headers

---

The only possible way is when the tenant is inferred from the URL itself:

-   Via subdomain
-   Via subfolder
-   Via query parameter

.warning[But it's not trivial either]

---

**Via subdomain**<br/>
Django only reverses the path, so the full domain of the tenant must be prepended.

**Via subfolder**<br/>
In order to abstract away the beginning of the path, <u>additional wizardry</u> must be performed.

**Via query parameter**<br/>
All URLs must be appended with the query parameter.

---

class: middle
layout: false

# The scope of everything else

---

## Admin site

-   For the URL it comes as part of your tenant routing scheme
-   For adjusting the models that are available to edit, you might need to create a custom admin site.

---

## Cache

You can define a tenant-specific cache key function:

```python
# settings.py
CACHES = {
    "default": {
        ...
        "KEY_FUNCTION": "myproject.cache.get_key_from_tenant",
    }
}

# myproject/cache.py
def get_key_from_tenant(key, key_prefix, version):
    tenant = get_active_tenant()
    return "{}:{}:{}:{}".format(tenant, key_prefix, version, key)
```

---

## Celery tasks

You can just pass the tenant to activate as one of your task parameters.

---

## Channels (websockets)

-   Requires (additional) custom middleware for activating tenant from request and passing tenant as part of the scope.
-   Requires naming your consumer groups including the tenant (for proper cross-tenant group isolation)

.warning[This requires some boilerplate code]

---

## Management commands

-   For custom management commands, you need to include a tenant argument, so you can activate the tenant before executing the command.
-   For existing, non tenant-aware commands, you can define a special management command that takes a tenant argument and another management command with its arguments, so that you can activate the tenant before calling the other management command.

.warning[This gets trickier the more elegant]

---

## File storage

-   You can define a custom tenant storage that organizes files per tenant.
-   If cross-tenant file access is a security problem for you, you will need a custom view to act as proxy for files and decide if the incoming request has access to the requested file.

---

class: middle
layout: false

# Yet another package for this

---

## Available packages

https://djangopackages.org/grids/g/multi-tenancy/

---

template: title
