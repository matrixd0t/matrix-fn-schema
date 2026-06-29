# matrix-fn-schema

Convert Python function signatures (type annotations + docstrings) into OpenAI-compatible JSON Schema (tool call format).

```python
from dataclasses import dataclass
from enum import Enum
from typing import Annotated, List, Optional, Literal
from uuid import UUID
from matrix_fn_schema import build_json_schema


class Priority(int, Enum):
    LOW = 1
    MEDIUM = 2
    HIGH = 3


@dataclass
class Attachment:
    filename: str
    url: str


def create_task(
    title: Annotated[str, "Max 200 chars"],
    priority: Priority,
    assignee_id: UUID,
    attachments: List[Attachment],
    deadline: Optional[str] = None,
) -> dict:
    """Create a new task."""
    ...


schema = build_json_schema(create_task)
```

Result:

```json
{
  "type": "function",
  "name": "create_task",
  "description": "Create a new task.",
  "strict": true,
  "parameters": {
    "type": "object",
    "properties": {
      "title":         {"type": "string"},
      "priority":      {"type": "integer", "enum": [1, 2, 3]},
      "assignee_id":   {"type": "string", "format": "uuid"},
      "attachments":   {"type": "array", "items": {"type": "object", "properties": {
                          "filename": {"type": "string"},
                          "url": {"type": "string"}
                        }, "required": ["filename", "url"], "additionalProperties": false}},
      "deadline":      {"anyOf": [{"type": "string"}, {"type": "null"}]}
    },
    "additionalProperties": false,
    "required": ["title", "priority", "assignee_id", "attachments", "deadline"]
  }
}
```

## Supported types

| Python                          | JSON Schema                                         |
|---------------------------------|-----------------------------------------------------|
| `int`                           | `{"type": "integer"}`                               |
| `float`                         | `{"type": "number"}`                                |
| `str`                           | `{"type": "string"}`                                |
| `bool`                          | `{"type": "boolean"}`                               |
| `None`                          | `{"type": "null"}`                                  |
| `Literal["a", "b"]`             | `{"enum": ["a", "b"]}`                              |
| `list[X]`                       | `{"type": "array", "items": <X>}`                   |
| `tuple[X, Y]`                   | `{"type": "array", "prefixItems": [...], "minItems": N, "maxItems": N}` |
| `tuple[X, ...]`                 | `{"type": "array", "items": <X>}`                   |
| `dict[K, V]`                    | `{"type": "object", "additionalProperties": <V>}`   |
| `Optional[X]` / `X \| None`     | `{"anyOf": [<X>, {"type": "null"}]}`                |
| `Union[X, Y, Z]` / `X \| Y \| Z` | `{"anyOf": [<X>, <Y>, <Z>]}`                     |
| `Annotated[T, ...]`             | unwrapped to `T`                                    |
| `Final[T]`                      | unwrapped to `T`                                    |
| `enum.Enum` (str values)        | `{"type": "string", "enum": [...]}`                 |
| `enum.IntEnum`                  | `{"type": "integer", "enum": [...]}`                |
| `uuid.UUID`                     | `{"type": "string", "format": "uuid"}`              |
| `datetime.datetime`             | `{"type": "string", "format": "date-time"}`         |
| `datetime.date`                 | `{"type": "string", "format": "date"}`              |
| `datetime.time`                 | `{"type": "string", "format": "time"}`              |
| `@dataclass`                    | recursive `{"type": "object", "properties": {...}}` |
| `TypedDict`                     | recursive `{"type": "object"}` with `Required`/`NotRequired` |
| `Self` / recursive types        | cycle-safe (breaks at the self-reference)           |
| `pydantic.BaseModel`            | delegated to `.model_json_schema()` (optional dep)  |

## Requirements

Python 3.10+. Dependencies: `docstring-parser>=0.16`.

Optional: `pydantic>=2` for `BaseModel` support.

Written with love by dotmatrix.