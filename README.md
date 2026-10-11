# Documentación de Pydantic Field() Constraints

## Tabla de Contenidos

- [Introducción](#introducción)
- [Field() con constraints](#field-con-constraints)
- [Ejemplos prácticos](#ejemplos-prácticos)
- [Validaciones avanzadas](#validaciones-avanzadas)
- [Conclusión](#conclusión)

---

## Introducción

Pydantic es una biblioteca de validación de datos para Python que utiliza **Field()** con *constraints* (restricciones) para definir reglas de validación en los modelos de datos. Esta documentación cubre el uso completo de `Field()` y sus parámetros de restricción.

---

## Field() con Constraints

### Sintaxis Básica

```python
from pydantic import BaseModel, Field

class Usuario(BaseModel):
    nombre: str = Field(..., min_length=3, max_length=50)
    edad: int = Field(ge=18, le=120)
    email: str = Field(pattern=r'^[\w\.-]+@[\w\.-]+\.\w+$')
```

### Parámetros Principales de `Field()`

| Parámetro | Tipo | Descripción |
|-----------|------|-------------|
| `default` | Any | Valor por defecto si no se proporciona |
| `default_factory` | Callable | Función que devuelve el valor por defecto |
| `alias` | str | Nombre alternativo para serialización/deserialización |
| `title` | str | Título del campo (para documentación) |
| `description` | str | Descripción del campo |
| `ge` / `gt` | float/int | Mínimo mayor o igual / estrictamente mayor |
| `le` / `lt` | float/int | Máximo menor o igual / estrictamente menor |
| `multiple_of` | int/float | El valor debe ser múltiplo de este número |
| `max_digits` | int | Dígitos máximos (para decimales) |
| `decimal_places` | int | Decimales permitidos |
| `pattern` | str | Expresión regular para validación de strings |
| `min_length` / `max_length` | int | Longitud mínima/máxima del string |
| `allow_inf_nan` | bool | Permitir inf/NaN en floats (default: True) |

---

## Ejemplos Prácticos

### 1. Validación de Edad con Rango y Múltiplo

```python
from pydantic import BaseModel, Field, ValidationError
from typing import Optional

class Persona(BaseModel):
    nombre: str = Field(
        ...,
        min_length=2,
        max_length=100,
        description="Nombre completo de la persona"
    )
    edad: int = Field(
        ge=0,
        le=150,
        multiple_of=1,  # Debe ser entero
        description="Edad en años"
    )
    altura: float = Field(
        gt=0,
        lt=3.0,
        decimal_places=2,
        max_digits=4,
        description="Altura en metros"
    )

# Uso correcto
persona = Persona(nombre="Ana", edad=25, altura=1.75)
print(persona.model_dump())
```

### 2. Validación de Email con Regex

```python
from pydantic import BaseModel, Field, EmailStr

class Contacto(BaseModel):
    email: str = Field(
        ...,
        pattern=r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$',
        description="Dirección de correo electrónico válida"
    )

# Alternativa usando EmailStr (built-in)
class ContactoV2(BaseModel):
    email: EmailStr = Field(..., description="Email válido")
```

### 3. Validación de Precio con Restricciones Financieras

```python
from pydantic import BaseModel, Field

class Producto(BaseModel):
    nombre: str = Field(..., max_length=200)
    precio: float = Field(
        gt=0,
        lt=1_000_000,
        multiple_of=0.01,  # Siempre dos decimales
        ge=0.01,           # Mínimo $0.01
        le=999999.99,      # Máximo $999,999.99
        description="Precio en moneda local"
    )
    cantidad: int = Field(
        ge=1,
        le=1_000_000,
        multiple_of=1,
        description="Cantidad de unidades"
    )

producto = Producto(
    nombre="Laptop",
    precio=999.99,
    cantidad=5
)
```

### 4. Validación con Default Factory

```python
from pydantic import BaseModel, Field
from datetime import date
from typing import List

class Evento(BaseModel):
    nombre: str = Field(..., max_length=100)
    fecha: date = Field(
        default_factory=date.today,
        ge=date(2024, 1, 1),
        le=date(2030, 12, 31)
    )
    participantes: List[str] = Field(
        default_factory=list,
        min_length=0,
        max_length=500
    )

evento = Evento(nombre="Conferencia")
print(evento.fecha)  # Hoy en día
```

### 5. Validación de JSON con Schema Nested

```python
from pydantic import BaseModel, Field
from typing import Optional

class Direccion(BaseModel):
    calle: str = Field(..., min_length=1, max_length=200)
    ciudad: str = Field(..., min_length=1, max_length=100)
    pais: str = Field(default="España", max_length=50)

class UsuarioCompleto(BaseModel):
    id_usuario: int = Field(ge=1, le=9_000_000)
    nombre: str = Field(..., min_length=2, max_length=50)
    direccion: Direccion = Field(default_factory=Direccion)
    activo: bool = Field(default=True)

usuario = UsuarioCompleto(
    id_usuario=42,
    nombre="Carlos",
    direccion={"calle": "Calle Mayor 123", "ciudad": "Madrid"}
)
```

---

## Validaciones Avanzadas

### Combinación de Múltiples Constraints

```python
from pydantic import BaseModel, Field, field_validator
from typing import Optional

class Registro(BaseModel):
    codigo: str = Field(
        ...,
        min_length=8,
        max_length=12,
        pattern=r'^[A-Z0-9]{8,12}$',  # Solo letras mayúsculas y números
        description="Código alfanumérico de 8 a 12 caracteres"
    )
    version: str = Field(
        default="1.0.0",
        pattern=r'^\d+\.\d+\.\d+(-[a-zA-Z0-9]+)?$'
    )

# Validación adicional con field_validator
@field_validator('codigo')
@classmethod
def validar_codigo(cls, v: str) -> str:
    if 'A' in v and 'Z' not in v:
        raise ValueError("El código debe contener al menos una letra")
    return v.upper()
```

### Validación Condicional con `Field()` y `field_validator`

```python
from pydantic import BaseModel, Field, field_validator
from typing import Optional

class ProductoComplejo(BaseModel):
    nombre: str = Field(..., max_length=200)
    precio_unitario: float = Field(gt=0, le=1_000_000)
    cantidad: int = Field(ge=0, le=9_999_999)
    descuento: Optional[float] = Field(default=None, ge=0, le=100)

    @field_validator('precio_unitario', 'cantidad')
    @classmethod
    def validar_precio_cantidad(cls, v):
        if isinstance(v, float) and v < 0.01:
            raise ValueError("El valor debe ser >= 0.01")
        return v

    @field_validator('descuento')
    @classmethod
    def validar_descuento(cls, v, values):
        precio = values.data['precio_unitario']
        cantidad = values.data['cantidad']
        if v is not None:
            total_sin_descuento = precio * cantidad
            descuento_aplicado = total_sin_descuento - (total_sin_descuento * v / 100)
            if descuento_aplicado < 0.01:
                raise ValueError("El descuento no puede reducir el precio a cero")
        return v
```

### Validación de URLs y Domains

```python
from pydantic import BaseModel, Field, HttpUrl, AnyUrl

class SitioWeb(BaseModel):
    url_principal: HttpUrl = Field(..., description="URL principal del sitio")
    url_alternativa: Optional[AnyUrl] = Field(default=None, description="URL alternativa")
    dominio: str = Field(
        ...,
        pattern=r'^[a-zA-Z0-9][a-zA-Z0-9\-]{1,61}[a-zA-Z0-9]\.(com|org|net|es)$',
        max_length=253
    )

sitio = SitioWeb(url_principal="https://ejemplo.com")
```

---

## Errores y Manejo de Validaciones

### Captura de Errores Detallados

```python
from pydantic import BaseModel, Field, ValidationError
import json

class Formulario(BaseModel):
    nombre: str = Field(..., min_length=3, max_length=50)
    edad: int = Field(ge=18, le=65)
    email: str = Field(pattern=r'^[\w\.-]+@[\w\.-]+\.\w+$')

try:
    Formulario(nombre="A", edad=-5, email="noesemail")
except ValidationError as e:
    print(json.dumps(e.errors(), indent=2))
```

**Salida:**
```json
[
  {
    "type": "string_too_short",
    "loc": ["nombre"],
    "msg": "String should have at least 3 characters",
    "input": "A",
    "ctx": {"limit_value": 3}
  },
  {
    "type": "less_than_equal",
    "loc": ["edad"],
    "msg": "Input should be less than or equal to 65",
    "input": -5,
    "ctx": {"limit_value": 65}
  },
  {
    "type": "string_pattern_mismatch",
    "loc": ["email"],
    "msg": "String should match pattern '^\\w\\.-]+@\\w\\.-]+\\.\\w+$'",
    "input": "noesemail"
  }
]
```

---

## Conclusión

Los *constraints* de `Field()` en Pydantic proporcionan una forma declarativa, legible y potente de definir reglas de validación. La combinación con `field_validator` permite validaciones lógicas complejas que dependen del contexto completo del modelo.

### Recursos Adicionales

- [Documentación oficial de Pydantic](https://docs.pydantic.dev/latest/)
- [Pydantic v2 Migration Guide](https://docs.pydantic.dev/latest/migration/)
- [GitHub: pydantic/pydantic](https://github.com/pydantic/pydantic)

---

*Última actualización: 2024*