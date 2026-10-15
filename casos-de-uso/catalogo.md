# Casos de uso: servicio de catálogo

Responsabilidades del servicio según el enunciado §4.1: mantener la copia local de categorías, profesionales y horarios semanales, conservar la versión aplicada, sincronizarla (completa e incremental) y resolver las búsquedas sobre esa copia local.

---

## CU-CAT-01: Inicializar o reconstruir el catálogo desde snapshot

- **Actor:** el propio servicio (al arrancar o cuando otro caso lo pide).
- **Disparador:** base local vacía, pedido de reconstrucción (CU-CAT-03) o estado local que no puede verificarse.
- **Flujo principal:**
  1. Obtiene el snapshot completo con `GET /api/synchronization/snapshot` usando el JWT técnico.
  2. Reemplaza categorías, profesionales y horarios en una única transacción.
  3. Guarda `snapshotVersion` como versión local, solo si las tres colecciones quedaron aplicadas.
- **Alternativas y errores:**
  - La cátedra no responde o devuelve error: la copia anterior queda intacta y el estado de sincronización informa el fallo.
  - Falla a mitad de la aplicación: rollback completo; la versión local no avanza.
- **Referencias:** enunciado §5.4, §6.1 · anexo §7.

## CU-CAT-02: Aplicar cambios incrementales

- **Actor:** servicio de la cátedra (vía Kafka).
- **Disparador:** evento `CatalogUpdated` en el topic del catálogo.
- **Flujo principal:**
  1. Lee en Redis la metadata (`currentVersion`, `oldestAvailableVersion`).
  2. Si la versión local no es menor que `oldestAvailableVersion`, recorre en orden `catedra:sync:changes:{v}` para cada versión posterior a la local.
  3. Cada cambio trae solo los IDs afectados: lee el estado vigente de cada ID desde los Hash de entidades y lo aplica.
  4. La versión local avanza solo cuando todos los cambios de esa versión quedaron aplicados.
- **Alternativas y errores:**
  - Notificación o cambio repetido: no altera el resultado (idempotencia).
  - Falta un cambio o la versión local quedó fuera de la ventana disponible: pasa a CU-CAT-03.
- **Referencias:** enunciado §5.5, §6.2 · anexo §14, §15.3.

## CU-CAT-03: Detectar discontinuidad y resincronizar

- **Actor:** el propio servicio.
- **Disparador:** durante CU-CAT-02 la versión local es menor que `oldestAvailableVersion`: falta algún incremento (por ejemplo, local `3` y `oldestAvailableVersion=4`).
- **Flujo principal:**
  1. No intenta aplicar cambios sueltos.
  2. Ejecuta CU-CAT-01 para reemplazar la copia local por un snapshot.
- **Referencias:** enunciado §5.6, §6.2 · anexo §18.2.

## CU-CAT-04: Buscar profesionales con filtros locales

- **Actor:** usuario final (desde la app KMP).
- **Disparador:** el usuario busca o filtra profesionales.
- **Flujo principal:**
  1. Recibe filtros por categoría, nombre, estado habilitado y disponibilidad, con paginación.
  2. Resuelve la consulta **solo** sobre la copia local.
  3. Devuelve la página de resultados.
- **Alternativas y errores:**
  - Filtros inválidos: `400` con `application/problem+json`.
  - Sin JWT válido: `401`.
- **Referencias:** enunciado §4.1, §5.7, §6.2 (no consultar a la cátedra por cada búsqueda).

## CU-CAT-05: Proveer profesional y agenda vigente a Turnos

- **Actor:** Backend-Turnos.
- **Disparador:** Turnos necesita validar un profesional o calcular disponibilidad (CU-TUR-03, CU-TUR-04).
- **Flujo principal:**
  1. Recibe el pedido con JWT válido.
  2. Devuelve el profesional y sus horarios semanales vigentes según la copia local.
- **Alternativas y errores:**
  - Profesional inexistente o deshabilitado: error explícito para que Turnos no inicie la operación.
  - Token ausente o inválido: `401`.
- **Referencias:** enunciado §3, §4.2, §9 (comunicación protegida entre servicios).

## CU-CAT-06: Informar el estado de sincronización

- **Actor:** usuario final o operador.
- **Disparador:** consulta del estado.
- **Flujo principal:** devuelve la versión local aplicada, la fecha de la última sincronización y el último error, si lo hubo.
- **Referencias:** enunciado §4.1 ("informar el estado y los errores de sincronización").
