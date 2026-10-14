# Perfil de implementación — Laravel, React y PostgreSQL

Este documento adapta las referencias de `spec-python/` y `spec-.net/` al stack exigido por el
reto: **PHP/Laravel**, **React** y **PostgreSQL 16**. No redefine el producto. Si contradice
`spec.md`, `data-model.md` o `api-contract.md`, prevalecen esos documentos.

## 1. Límites de responsabilidad

| Repositorio | Responsabilidad |
|---|---|
| `api` | API Laravel, migraciones, reglas de negocio y almacenamiento de portadas |
| `app` | Aplicación React que consume `/api/v1` |
| `page` | Sitio público estático; no llama a la API |
| `infra` | Contenedores, red, volumen y motor PostgreSQL vacío; no contiene DDL |
| `tool` | Datos de demostración mediante HTTP contra la API; nunca SQL directo |

## 2. Arquitectura de `api`

La implementación conserva la arquitectura hexagonal de
[`spec-.net/architecture.md`](spec-.net/architecture.md): las dependencias apuntan hacia el
dominio y solo la composición puede conocer tanto puertos como adaptadores.

Una organización compatible con Laravel es:

```text
src/Library/
├── Domain/                 # PHP puro: entidades, valores e invariantes
├── Application/
│   ├── Ports/              # interfaces consumidas por casos de uso
│   └── UseCases/           # un caso de uso por servicio
├── Adapters/
│   ├── Inbound/Http/       # rutas, controladores, validación y errores
│   └── Outbound/           # Eloquent/SQL, JWT, hash, reloj y portadas
└── Bootstrap/              # composición de puertos y adaptadores
```

Laravel y Eloquent solo pueden aparecer en adaptadores y en la composición. Las interfaces de
repositorio, reloj, hash de contraseñas, JWT y almacenamiento de portadas pertenecen a
`Application`. El proveedor de servicios de Laravel que registre esas interfaces y sus
implementaciones es el único punto de composición.

Esta estructura es una guía de implementación, no una séptima capa ni un requisito de producto.
Las rutas, columnas y código permanecen en inglés; mensajes de usuario y documentos, en español.

## 3. Migraciones y ciclo de una base vacía

PostgreSQL inicia vacío porque `infra` solo crea el motor, su usuario, red y volumen. No debe
haber `CREATE TABLE`, scripts de inicialización de esquema ni un mecanismo ORM de crear tablas al
arrancar en ese repositorio.

Las migraciones Laravel de `api` son la única fuente de DDL y deben crear el esquema `library`,
las seis tablas de negocio, restricciones, índices y las cinco categorías fijas. La operación de
despliegue ejecuta las migraciones de Laravel de forma explícita antes de declarar la API lista.
Por tanto:

1. `docker compose up` en `infra` puede levantar PostgreSQL sin tablas de negocio.
2. La ejecución de las migraciones de `api` materializa el esquema `library`.
3. El arranque de la API puede aprovisionar el administrador inicial solo después de que las
   migraciones hayan terminado correctamente; no sustituye las migraciones ni crea DDL.

Las seis tablas de negocio son exactamente `category`, `book`, `member`, `loan`, `loan_item` y
`user`, todas en `library`. La tabla de control propia de Laravel para registrar migraciones es
metadato técnico y debe quedar fuera del esquema `library` (por ejemplo, en `public`). No es una
tabla de negocio, no se consulta desde el dominio y no amplía el modelo de seis entidades descrito
en [`spec-.net/data-model.md`](spec-.net/data-model.md).

## 4. Concurrencia de ejemplares

No se agrega una columna de versión, `updated_at` ni otro atributo al modelo físico. Al registrar
un préstamo, el adaptador de persistencia realiza una actualización condicional y atómica por cada
libro activo:

- solo descuenta un ejemplar si `available_copies > 0`;
- comprueba que exactamente una fila fue afectada;
- si alguna línea no pudo descontarse, revierte toda la transacción y la API devuelve
  `409 BOOK_NOT_AVAILABLE`;
- crea el préstamo, sus líneas y los descuentos en la misma transacción;
- conserva `ck_book_available_range` como barrera final del motor.

Esta estrategia cumple RNF-2 sin introducir una columna que no existe en `data-model.md`. La
prueba de concurrencia debe demostrar que dos solicitudes simultáneas por el último ejemplar
producen exactamente un `201` y un `409`.

## 5. Portadas y baja lógica

La base almacena exclusivamente `cover_key`, una clave opaca o `null`; nunca una ruta local ni el
binario. El adaptador de almacenamiento es responsable de guardar y leer el binario. Debe validar
por contenido JPEG/PNG y el límite de 2 MB definido por el contrato.

Al eliminar una portada, primero se anula `cover_key` y se confirma la transacción de base de datos.
Solo después se solicita al adaptador borrar el binario. Si el borrado físico falla, no se revive
la referencia en la base: el binario huérfano debe quedar como incidencia operativa recuperable.

Eliminar un libro significa llenar `deleted_at`; nunca eliminar la fila. No se permite la baja si
tiene préstamos abiertos. Las consultas de catálogo y detalle filtran libros con `deleted_at` nulo.

## 6. Valores derivados e historial

`loan.status` y `days_late` no son columnas. Se calculan al leer, con un reloj UTC:

- `closed`: todas las líneas tienen `returned_at`.
- `overdue`: hay alguna línea sin devolver cuyo `due_at` es anterior al instante actual.
- `open`: hay líneas sin devolver y ninguna está vencida.

La representación numérica exacta de `days_late` no está definida por el contrato más allá de
que se calcula desde `due_at` y el instante actual UTC. No se debe persistir ni asumir una regla de
redondeo distinta sin una decisión de producto.

Al crear cada `loan_item`, la aplicación copia `book_title`, `isbn` y `category_name` del catálogo
en ese instante. El reporte agrupa por `book_id`, `book_title` y `category_name` de `loan_item`, no
por los valores actuales de `book` o `category`. Por eso un cambio posterior en el catálogo no
modifica reportes históricos.

## 7. Contrato, configuración y verificación

La API Laravel conserva sin cambios [`spec-.net/api-contract.md`](spec-.net/api-contract.md): base
`/api/v1`, JSON `snake_case`, UUID v4, fechas UTC y el cuerpo de error único. Todos los endpoints,
salvo `POST /auth/login` y `GET /health`, requieren JWT. La política CORS usa exclusivamente el
origen configurado por entorno.

Las variables de entorno requeridas son `DATABASE_URL`, `JWT_SECRET`, `ADMIN_USERNAME`,
`ADMIN_PASSWORD` y `CORS_ORIGIN`; `COVERS_DIR` es opcional. Se versiona solo `.env.example` y no se
incluyen secretos, contraseñas ni hashes precomputados.

Las pruebas de arquitectura deben comprobar las reglas de dependencia, pero la herramienta concreta
para PHP no está impuesta por la especificación. Las pruebas de dominio, aplicación, integración,
contrato y concurrencia siguen la estrategia descrita en `plan.md` y `tasks.md`.

## 8. Puntos que siguen sin definición suficiente

No se añaden decisiones de producto para los siguientes asuntos porque el contrato no las define:

- El algoritmo de redondeo para presentar `days_late` cuando el retraso no completa un día.
- Las cabeceras y representación exactas de éxito de `GET /books/{id}/cover`.
- La forma detallada de algunas respuestas sin ejemplo, como eliminaciones y la lista de
  categorías.
- La semántica precisa de frontera temporal para `from` y `to` en el listado de préstamos.

La implementación debe solicitar una decisión antes de fijar comportamientos que afecten el
contrato público.
