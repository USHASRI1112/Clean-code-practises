from pathlib import Path

content = r'''# Python & Django Clean Code Practices

A practical coding standard for Python, Django, and Django REST Framework projects.

The purpose of this document is to keep code readable, predictable, maintainable, testable, and consistent across the backend team. These practices apply to new code and should be followed when modifying existing code.

---

## 1. General principles

- Prefer simple, readable code over clever or unnecessarily abstract code.
- Keep each function, class, and module focused on one responsibility.
- Avoid duplicate logic. Extract shared logic only when it genuinely has multiple callers or a clear reusable purpose.
- Use meaningful names that explain intent.
- Avoid unexplained abbreviations and one-letter variable names, except for conventional short-lived loop variables.
- Keep functions small enough to understand without navigating through many unrelated blocks.
- Do not introduce a design pattern without a clear requirement.
- Do not silently ignore exceptions or return misleading default values.
- Remove unused imports, variables, commented-out code, and obsolete implementations.
- Follow PEP 8 and use automated formatting and linting.

## 2. Recommended application structure

Organise code by feature or domain. A feature module may use the following structure:

```text

project_management/
│
├── daos/
│   ├── impl/
│   │   └── project_dao_impl.py
│   └── interface/
│       └── project_dao_interface.py
│
├── dtos/
│   └── project_dto.py
│
├── serializers/
│   └── project_serializer.py
│
├── services/
│   ├── impl/
│   │   └── project_service_impl.py
│   └── interface/
│       └── project_service_interface.py
│
├── utils/
│   └── project_utils.py
│
├── constants/
│   ├── project_constants.py
│   └── error_messages.py
│
├── models/
│   ├── __init__.py
│   └── project_model.py
│
├── views/
│   └── project_view.py
│
├── apps.py
└── urls.py
```


Use only the layers that the feature actually needs. Do not create empty folders or interfaces merely to match this structure.

### Layer responsibilities

| Layer | Responsibility |
|---|---|
| Views | HTTP request/response handling, permissions, status codes, serializer orchestration |
| Serializers | Validate and transform API input/output |
| Services | Business rules and application workflows |
| DAOs / repositories | Database queries and persistence |
| DTOs | Typed data structures passed between application layers |
| Utils | Small, reusable, domain-independent helpers |

Rules:

- Views must not contain complex business logic or large SQL queries.
- Services must not depend on HTTP request objects or DRF response objects.
- DAOs must not decide API response formats.
- Serializers must not perform complex business workflows.
- Keep dependencies flowing in one direction: views → services → DAOs.
- Avoid circular imports between layers.

## 3. Django REST Framework API design

### Choosing a view type

Choose the simplest DRF view that fits the endpoint:

1. **`APIView`** — use for custom endpoints that do not naturally fit CRUD operations.
2. **`GenericAPIView` with mixins** — use when you need selected generic operations with custom behaviour.
3. **`ListAPIView`, `RetrieveAPIView`, `CreateAPIView`, etc.** — use for standard single-purpose endpoints.
4. **`ModelViewSet` / `ReadOnlyModelViewSet`** — use for conventional model CRUD APIs where router-based actions are appropriate.
5. **Custom `ViewSet`** — use when the endpoint has a set of related actions but does not map directly to model CRUD.

Guidelines:

- Do not use `ModelViewSet` automatically for every API.
- Do not use a ViewSet just to reduce the number of lines.
- Keep custom actions focused and documented.
- Declare permissions, authentication, serializers, and querysets explicitly where practical.
- Use routers consistently if the project uses ViewSets.
- Return appropriate HTTP status codes and a consistent response format.
- Avoid putting business logic directly inside `get_queryset()`, `create()`, `update()`, or custom actions.

## 4. Serializers

Use DRF serializers as the standard boundary for API input validation and output representation.

- Use `Serializer` for non-model or custom API data.
- Use `ModelSerializer` when the API representation maps naturally to a Django model.
- Validate incoming data through `is_valid(raise_exception=True)`.
- Use field-level validation for individual fields.
- Use `validate()` for validation involving multiple fields.
- Keep serializers focused on validation and representation.
- Avoid database-heavy operations and complex business logic inside serializers.
- Use explicit `read_only_fields` and `write_only=True` where appropriate.
- Do not expose internal model fields accidentally.
- Use separate input and output serializers when their responsibilities or shapes differ materially.
- Never trust request data just because it passed basic type validation.

Example:

```python
from rest_framework import serializers


class CreateProjectSerializer(serializers.Serializer):
    name = serializers.CharField(max_length=150)
    description = serializers.CharField(
        required=False,
        allow_blank=True,
    )

    def validate_name(self, value: str) -> str:
        cleaned_value = value.strip()
        if not cleaned_value:
            raise serializers.ValidationError("Name cannot be empty.")
        return cleaned_value
```

## 5. Pydantic models and DTOs

Use typed DTOs to define the data contracts between application layers.

- Prefer Python `dataclass` for simple internal data-transfer objects when validation is not required.
- Use Pydantic models when runtime validation, parsing, serialisation, or schema generation is valuable.
- Use DRF serializers for DRF request and response validation by default.
- Do not create duplicate validation in both Pydantic and DRF serializers without a clear reason.
- Do not use Pydantic merely because it is available.
- Keep DTOs independent of Django ORM models and HTTP request objects.
- Make optional fields explicit using `T | None`; do not use an ambiguous empty string or empty dictionary to represent missing data.
- Define clear conversion boundaries between models, DTOs, and serializers.

**Important:** Pydantic and DRF serializers solve overlapping but different problems. DRF serializers are normally the API boundary in a DRF application; Pydantic is useful for internal schemas, external integrations, configuration, and data pipelines.

## 6. Strict Python typing

Type annotations are required for application code.

- Add parameter and return types to functions and methods.
- Add types to class attributes and important local variables when inference is not clear.
- Avoid `Any` entirely in application code.
- Do not use `Any` to silence a type-checking error.
- Use `object` when a value is genuinely unknown and must be narrowed safely.
- Use `TypedDict` for dictionary-shaped data with a known schema.
- Use `Literal` for a fixed set of string or numeric values.
- Use `Enum` for meaningful fixed sets of domain values.
- Use `Protocol` when defining a structural interface.
- Use `TypeVar` and generics only when they improve type safety and readability.
- Use `T | None` for optional values in modern Python.
- Narrow types with `isinstance`, explicit checks, or validation before using a value.
- Avoid broad `dict` and `list` annotations when the element types are known.

Examples:

```python
def get_project(project_id: int) -> ProjectDTO | None:
    ...
```

```python
from typing import TypedDict


class ProjectData(TypedDict):
    id: int
    name: str
    is_active: bool
```

Recommended tooling:

- `mypy` or `pyright` for static type checking.
- Configure the checker centrally and run it before merging.
- Prefer strict settings and gradually enforce them in legacy modules.
- Do not add `# type: ignore` without a specific explanation and, where supported, a narrow error code.

## 7. Docstrings: Args, Logic, Returns

Write docstrings for important functions, especially business logic, calculations, data traversal, database workflows, and transformations.

Use this structure:

```python
def calculate_total_distance(
    segment_lengths: list[float],
    adjustment: float,
) -> float:
    """Calculate the total distance after applying an adjustment.

    Args:
        segment_lengths: Length of each segment in metres.
        adjustment: Distance to add to the calculated total.

    Logic:
        1. Sum the lengths of all segments.
        2. Add the supplied adjustment.
        3. Return the resulting distance.

    Returns:
        The adjusted total distance in metres.
    """
    return sum(segment_lengths) + adjustment
```

Guidelines:

- **Args:** Explain parameters, units, expected values, and important constraints.
- **Logic:** Explain the meaningful steps or business reasoning, not every obvious line of Python.
- **Returns:** Explain the returned value and its meaning.
- **Raises:** Document important exceptions when callers need to understand them.
- Do not write redundant docstrings such as `Returns: Returns the result`.
- Simple getters, setters, and self-explanatory functions do not need lengthy Logic sections.
- Keep docstrings accurate when implementation changes.

## 8. Services and business logic

- Put business rules and application workflows in service classes or focused service functions.
- Keep each service responsible for a coherent use case or domain capability.
- Services may coordinate multiple DAOs and other services.
- Keep services independent of DRF's `Request` and `Response`.
- Pass required values explicitly instead of reading hidden global state.
- Use transactions for workflows that require multiple database operations to succeed or fail together.
- Raise meaningful domain/application exceptions; translate them into HTTP responses at the API boundary.
- Avoid services that simply forward every DAO method without adding orchestration or business value.
- Avoid very large service classes that own unrelated workflows.

## 9. DAOs / repository layer and database access

- Keep database access in the DAO/repository layer when the project uses that architecture consistently.
- Give query methods names that explain what they retrieve or change.
- Return typed models, DTOs, or collections with clear contracts.
- Use `select_related()` for suitable foreign-key and one-to-one relationships.
- Use `prefetch_related()` for suitable reverse and many-to-many relationships.
- Avoid N+1 queries.
- Use `bulk_create()` and `bulk_update()` when appropriate and when their signal/validation behaviour is acceptable.
- Use `transaction.atomic()` for multi-step database changes that must be atomic.
- Never build SQL by concatenating untrusted input.
- Use ORM expressions or parameterised raw SQL.
- Keep raw SQL readable, parameterised, and documented.
- Do not fetch every row when filtering, pagination, or aggregation can be performed in the database.
- Add indexes based on query patterns and measured performance, not speculation.

## 10. Singleton and object lifecycle

Do not make every service, DAO, ViewSet, or helper a singleton.

- A normal class instance per request is usually inexpensive and is the default.
- Prefer stateless services with explicit dependencies.
- Reuse long-lived resources when they are genuinely expensive to create, such as connection pools or configured clients; use the lifecycle management supported by the library/framework.
- Django already manages database connections and application configuration. Do not create a separate singleton database connection.
- Avoid mutable global singleton objects that can leak state between requests.
- Never store request-specific data, user data, or mutable workflow state in a shared service instance.
- If a service is shared, it must be safe for concurrent requests and must not depend on per-request mutable attributes.
- Use dependency injection or explicit construction where it improves testability.

**Rule:** Use singleton scope only when there is a demonstrated lifecycle or resource-management reason. Do not use it as a general performance optimisation.

## 11. Swagger / OpenAPI documentation

Use `drf-spectacular` as the recommended OpenAPI documentation tool for DRF projects, unless the project already has an established alternative.

- Document every public API endpoint.
- Add meaningful summaries and descriptions.
- Specify request serializers and response serializers.
- Document path parameters, query parameters, and important headers.
- Document success and error response codes.
- Document authentication and permission requirements.
- Describe pagination and filtering behaviour.
- Use `@extend_schema` for custom actions and endpoints where automatic introspection is insufficient.
- Keep the API documentation aligned with actual serializer behaviour.
- Do not use vague descriptions such as "API to get data".

Example:

```python
from drf_spectacular.utils import extend_schema
from rest_framework.views import APIView
from rest_framework.response import Response


class ProjectListView(APIView):

    @extend_schema(
        summary="List projects",
        description="Returns projects accessible to the authenticated user.",
        responses={200: ProjectSerializer(many=True)},
    )
    def get(self, request: Request) -> Response:
        ...
```

## 12. Error handling and logging

- Catch only exceptions that the current layer can handle meaningfully.
- Do not use bare `except:`.
- Do not silently pass exceptions.
- Do not return success responses when an operation failed.
- Use project-defined exception classes for expected business errors.
- Keep HTTP-specific exception conversion in the view/API boundary or a central exception handler.
- Use Python logging instead of `print()`.
- Include useful context in logs, but never log passwords, tokens, secrets, or unnecessary personal data.
- Use appropriate log levels: `debug`, `info`, `warning`, `error`, and `exception`.
- Avoid logging the same exception repeatedly at every layer.
use decorators for the errror ahndlign and also 

## 13. Django models and migrations

- Keep model fields explicit, with appropriate nullability, defaults, lengths, and relationships.
- Use `related_name` where it improves reverse-relation clarity.
- Add database constraints for important data-integrity rules.
- Use model methods for behaviour that naturally belongs to one model; keep multi-entity workflows in services.
- Avoid overriding `save()` for complex workflows.
- Do not modify an already-applied migration in shared environments; create a new migration.
- Review generated migrations before committing.
- Use `on_delete` behaviour intentionally.
- Do not assume serializer validation alone guarantees database integrity.

## 14. Configuration and secrets

- Keep environment-specific values outside source code.
- Never commit credentials, API keys, private keys, or production secrets.
- Validate required settings at application startup.
- Separate development, testing, staging, and production configuration.
- Avoid scattering environment-variable reads throughout business logic.
- Use Django settings as the central configuration interface.

## 15. API security

- Apply authentication and permission classes explicitly.
- Enforce object-level access where users can access individual resources.
- Validate and constrain user-supplied filters, sorting, pagination, and file uploads.
- Do not expose internal exception messages or stack traces to clients.
- Use HTTPS in deployed environments.
- Apply throttling where endpoints are vulnerable to abuse.
- Follow Django's CSRF, session, password, and security guidance for the authentication method in use.
- Treat all client-supplied data as untrusted.

## 16. Performance

- Measure before optimising.
- Avoid unnecessary database queries and repeated serialisation.
- Paginate collection endpoints.
- Use streaming or background jobs for genuinely long-running work.
- Move slow external API calls out of request-response flows when appropriate.
- Use caching only with a clear invalidation strategy.
- Avoid premature multiprocessing, threading, or asynchronous code.
- Do not make synchronous Django code asynchronous unless the workload and supported dependencies justify it.

## 17. Background tasks and external integrations

- Use a task queue such as Celery for work that should not block an HTTP request.
- Make tasks idempotent where retries are possible.
- Set timeouts for external HTTP calls.
- Handle retries with bounded attempts and appropriate backoff.
- Validate external API responses before using them.
- Do not pass request objects or non-serialisable runtime objects into background tasks.
- Log task identifiers and useful execution context.
- Keep task functions focused; delegate business workflows to services.

## 18. Code formatting and quality tools

Recommended baseline:

| Tool | Purpose |
|---|---|
| Black | Python code formatting |
| Ruff | Linting and import sorting |
| mypy or Pyright | Static type checking |
| pre-commit | Run checks before commits |
| drf-spectacular | OpenAPI / Swagger documentation |

Team rules:

- Keep tool configuration in the repository.
- Use the same tool versions and configuration across the team.
- Run formatting, linting, and type checks before raising a PR.
- Do not disable a rule globally to fix one local issue.
- Keep exceptions narrow and explain why they are necessary.

## 19. Git and pull-request practices

- Use meaningful branch names and commit messages.
- Keep pull requests focused on one feature, fix, or refactor.
- Do not commit generated files, local settings, virtual environments, or secrets unless the project explicitly requires them.
- Resolve formatting and linting issues before requesting review.
- Explain architectural changes and database migration impact in the PR.
- Include API request/response examples for API changes.
- Mention important assumptions, edge cases, and backward-compatibility concerns.
- Do not merge unresolved critical review comments.

## 20. Pull-request checklist

Before raising a PR, verify:

- [ ] Code follows the agreed feature/module structure.
- [ ] Views contain HTTP orchestration, not complex business logic.
- [ ] Serializers validate API input and define output representation.
- [ ] DTOs and service contracts have explicit types.
- [ ] No `Any` typing has been introduced.
- [ ] Important functions have Args, Logic, and Returns docstrings.
- [ ] Database access and transactions are handled appropriately.
- [ ] No avoidable N+1 queries or unbounded collection responses exist.
- [ ] Errors are handled and logged appropriately.
- [ ] OpenAPI documentation is updated for API changes.
- [ ] No secrets, debug statements, or unused code are committed.
- [ ] Formatting, linting, and type checks pass.
- [ ] Migrations are reviewed and included when needed.

---

## 21. Team-wide non-negotiable rules

1. No `Any` typing in application code.
2. All important functions must have parameter and return type annotations.
3. Important calculation and business-logic functions must document Args, Logic, and Returns.
4. API input must be validated using serializers.
5. Business logic must not be written directly in views.
6. Database access must follow the project's chosen DAO/repository convention.
7. API changes must include OpenAPI documentation.
8. Singletons must not be introduced without a valid lifecycle or resource-management reason.
9. Exceptions must not be silently swallowed.
10. Code must be readable enough for another developer to understand without requiring the original author to explain it.

These rules are intended to improve consistency, not to introduce unnecessary abstraction. When a rule does not fit a specific case, explain the reason in the pull request and agree on the exception with the reviewer.
'''
path = Path("/mnt/data/python-django-clean-code-practices.md")
path.write_text(content, encoding="utf-8")
print(f"Created: {path}")
