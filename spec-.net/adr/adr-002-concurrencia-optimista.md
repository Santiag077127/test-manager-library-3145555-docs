# ADR-002 · Actualización condicional atómica sobre los ejemplares

- **Estado:** aceptada
- **Contexto:** dos bibliotecarios pueden prestar el **último ejemplar** del mismo libro al mismo tiempo (RNF-2). Si ambos leen
  `available_copies = 1` y ambos escriben, el sistema presta un libro que no tiene.
- **Decisión:** cada descuento usa una actualización condicional y atómica que solo afecta al libro activo si
  `available_copies > 0`. El adaptador comprueba que se afectó exactamente una fila. Si no, revierte la transacción completa y el
  servidor responde **409 `BOOK_NOT_AVAILABLE`**. Crear el préstamo, sus líneas y todos los descuentos ocurre en la misma
  transacción. La **última barrera** es el `CHECK` `ck_book_available_range`
  (`0 <= available_copies <= total_copies`).
- **Alternativas descartadas:** una columna de versión: no existe en el modelo físico documentado; bloqueo pesimista de la fila
  (`SELECT … FOR UPDATE`) en cada préstamo: serializa toda la biblioteca por un caso raro; contador en memoria: no sobrevive a
  más de un proceso.
- **Consecuencias:** no se agrega `version`, `updated_at` ni otra columna al modelo. Si el `CHECK` salta, **algo escribió fuera del
  adaptador**: es un defecto, no un caso de negocio. Devolver dos veces la misma línea no suma dos veces porque `returned_at` ya
  está lleno (CA-07.2).
- **Cómo se verifica:** prueba concurrente con dos peticiones simultáneas sobre un libro con un ejemplar: exactamente una recibe 201.
