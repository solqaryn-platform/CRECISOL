# CRECISOL — C05-02 · Puerta de entrega verificable y privacidad (preparación técnica, NO certificación)

Fecha: 2026-11-13. Repositorio canónico: `crecisol-platform/CRECISOL`, ID 1450414592. Base auditada: `dev` SHA `a49fe605eb25916ca554c0d6a7f69e30648ce75b` (PR #120 integrado después del #119). Esta nota NO equivale a implementación ni prueba de correo real. Única microfase habilitada: C05-02; C05-03 bloqueada.

## Fuentes y precedentes

- Documento Maestro Único Word SHA-256 `ea5afc89e74915a9b18b9753ddb832f4be8e5d00f5e66b0f008e02e03476444c`; PDF SHA-256 `0dfd3217439760540390707ea8196d6252464a501a7fe3e6eafdc49eda39da77`. Fuente C05 SHA-256 `bf7036849d6f1aad36a3a111b62121947aa12fae8c2a0551e63b0cc788d4a0aa`, huellas consignadas en `docs/MASTER_REFERENCE.md` y `docs/c05/C05_02_ESPECIFICACION.md`. No se recalcularon los hashes de bytes de Word/PDF en esta intervención; quedan como referencias del repositorio.
- PR #119: ingesta neutral Kestrel/MySQL en fixture, sin ruta productiva.
- PR #120: prueba independiente de posesión de correo con inbox **en memoria**, digest separado por propósito y CAS MySQL. Seis workflows POSTMERGE del SHA citado figuran completed/success: #43426225261, #43426225047, #43426225148, #43426225053, #43426224959, #43426224952. Éxito de las pruebas aisladas NO es prueba de un destinatario real.
- Referencia funcional: `docs/c05/INTEGRACION_PRUEBA_CORREO_C05_02.md` y los controles B01–B18 / T01–T20.

## Contrato mínimo de la próxima integración

1. **Aislar transporte real mediante puerto**, sin alterar los invariantes Domain ni permitir que el usuario elija la dirección de entrega. Extraer destino exclusivamente del intent cifrado AES-GCM ya almacenado; verificar tenant ACTIVE y host verificado en backend. No habilitar endpoint productivo ni modificar DI productiva mientras el puerto y la política de autenticación no estén probados.
2. **Outbox transaccional y sin PII en claro**: crear registro interno idempotente por tenant+intent+purpose+version, con envelope cifrado, versión de clave, estado, contador, expiración y lease de worker. Vincular escritura del challenge y evento de entrega en una transacción local; si no puede persistirse, no confirmar la emisión. Nunca persistir token bearer sin protección; un reintento no debe generar capacidades adicionales.
3. **Fallo cerrado**: los errores permanentes, destino inválido, tenant suspendido, revocación o caducidad cancelan la capacidad pendiente; un timeout del proveedor no prueba entrega ni autoriza reemisión ilimitada. No confiar en un webhook no autenticado. Los mensajes de rebote no acreditan posesión.
4. **Comprobación del destinatario**: simular un proveedor HTTP/SMTP controlado con servidor de pruebas y buzón receptor separado del emisor, luego una prueba manual autorizada de entrega a un inbox real cuando exista infraestructura de ensayo, sin habilitar servicios de producción. La recepción del mensaje por sí sola NO acredita identidad; únicamente la presentación del desafío correcto y vigente demuestra posesión del canal para ese intento.
5. **No enumeración**: misma respuesta, códigos, estructura, cabeceras y estados para correos existentes/no existentes; medir distribuciones temporales, no asumir que HTTP idéntico significa tiempos idénticos. Validar rate limits por tenant, peer, destino pseudonimizado y defensa de abuso distribuido sin bloquear indefinidamente un destinatario por acciones ajenas.
6. **Autorización del invitante**: solo un actor autenticado y autorizado por backend puede crear invitaciones con acción, tenant, sucursal/recurso y rol permitido. Revalidar permisos al consumir. Ni desafío verificado ni invitación consumida producen `membership`, cuenta, cookie, JWT, password, MFA, sesión o elevación. El contrato de alta de membresía queda pendiente de integración segura posterior dentro de C05-02, sin invadir C05-03.
7. **Operación y privacidad**: secreto y versión fuera del repositorio, rotación/revocación, retención mínima de ciphertext y digests, borrado por política, auditoría sin PII, y limpieza de outbox; no guardar direcciones ni capacidades en logs, excepciones, métricas o URLs públicas. Parámetros de cuota, caducidad y reintento deben ser configurables del lado servidor.

## Matriz negativa que exige evidencia antes de certificar

| ID | Ensayo obligatorio | Criterio |
| --- | --- | --- |
| E01 | Tenant A intenta obtener/confirmar challenge de B | Cero lectura y cero cambios cruzados |
| E02 | Cliente falsifica Host / Forwarded / tenant_id | Rechazo y ningún envío |
| E03 | Entrega falla después de reservar challenge | Sin challenge utilizable ni grant |
| E04 | Dos workers reclaman mismo outbox | Máximo una reserva válida; reintento idempotente |
| E05 | 16 confirmaciones concurrentes del mismo secreto | Exactamente un consumo y cero sesiones |
| E06 | Token incorrecto, expirado, revocado, manipulado o de otro propósito | Cero confirmaciones |
| E07 | Tenant suspendido o invitante revocado antes de confirmar | Denegación incluso tras envío exitoso |
| E08 | Email conocido vs desconocido: respuesta, trazas y latencia bajo carga | Ninguna señal explotable de enumeración |
| E09 | DSN, credenciales, headers de transporte, URLs y logs | Cero secretos/PII o grants divulgados |
| E10 | Clave rotada, cifrado dañado y reinicio de worker | Fallo cerrado o recuperación controlada sin duplicados |
| E11 | Destinatario recibe mensaje pero no conoce desafío válido | Ninguna verificación |
| E12 | Desafío válido sin invitación, consentimiento ni autorización | Ningún alta de membresía, rol o sesión |

## Puertas A/B/C

- **A (funcional/trazabilidad)**: contrastar Word, capítulo 05, B01–B18 y T01–T20, PR #119/#120; demostrar el flujo persistencia → transporte → recepción → verificación independiente, manteniendo separación de capacidades.
- **B (seguridad/ofensiva)**: ejecutar E01–E12 con .NET10 + MySQL8.4 y transportes controlados, abuso/concurrencia, cifrado, permisos SQL, revocación y rechazo entre tenants; registrar artefactos, resultados y SHA.
- **C (arquitectura/operación)**: revisar diff, integración Domain/Application/Infrastructure/API, rendimiento, configuración, rollback, no-PII, no producción y regresiones C03/C04/C05-01/02 en SHA final. Verificar que `main` y SOLQARYN permanezcan intactos.

**Estado real de esta nota:** análisis y diseño de pruebas; E01–E12 NO EJECUTADOS AQUÍ. Sin compilación .NET, MySQL, correo real, auditorías ejecutadas de la nueva integración, nuevos secretos o despliegue. Ningún bloqueo P0/P1 queda levantado mediante documentación. No certificar C05-02 ni abrir C05-03. Rollback del PR documental: revertir únicamente este archivo tras revisar dependencias.
