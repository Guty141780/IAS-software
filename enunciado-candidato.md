# Prueba técnica — Desarrollador de Software (front-end) — DS-FE-20261119-m2f8

## 1. Antes de empezar

- Tienes 120 minutos desde que abres este documento. Registra en DECISIONES.md la hora de inicio, la hora en que terminaste la Parte 1 y la hora de fin.
- Lee todo el enunciado antes de empezar. Las tres partes son obligatorias.
- Tiempo sugerido: Parte 1 30 min · Parte 2 25 min · Parte 3 20 min · DECISIONES.md 12 min.
- Si el tiempo no alcanza, trabaja en este orden: Parte 1, Parte 2, Parte 3. Entrega lo que tengas y explica en DECISIONES.md qué faltó y cómo lo resolverías.
- No se espera: back-end real; diseño visual propio de alta fidelidad; despliegue en la nube; integración con proveedores reales de autenticación; más endpoints, pantallas o entidades de los indicados.
- Si algo es ambiguo, no preguntes: decide, documenta el supuesto en DECISIONES.md y sigue.
- Se valora la solución más adecuada para el contexto descrito: rendimiento, seguridad, mantenibilidad y buenas prácticas. Que funcione no basta.

## 2. Contexto de negocio

Banco Arrayán atiende a empresas medianas con su banca en línea empresarial. El banco reemplaza su portal monolítico por aplicaciones cliente independientes que consumen un BFF (backend for frontend) publicado en el mismo dominio del portal.

El cliente piloto, Distribuidora Andina S.A.S., tiene una cuenta de nómina, una corriente donde recibe el recaudo de sus 2.300 puntos de venta, unos 50.000 movimientos al mes, y una cuenta de compensación para los pagos de sus exportaciones. Cada mañana su tesorera revisa el recaudo del mes buscando referencias de pago y desde ahí transfiere a proveedores.

El 3 de noviembre, un día después de desplegar la versión 3.4.0 del portal actual, 23 clientes reportaron transferencias duplicadas. El equipo quiere entender la causa antes de que la nueva aplicación herede el problema.

## 3. Objetivo

Construir la primera versión de la aplicación de cuentas empresariales, con dos pantallas, consulta de movimientos y transferencia a terceros, contra una API simulada desde el contrato de la sección 4.2.

## 4. Datos iniciales y glosario

Estos son los únicos insumos de la prueba. No necesitas buscar información externa.

### 4.1 Datos de ejemplo

Cuentas del cliente piloto (respuesta de GET /cuentas):

```json
{ "cuentas": [
  { "id": "c-001", "numero": "0012345678", "tipo": "AHORROS", "alias": "Nómina", "moneda": "COP",
    "saldoDisponible": "48250300.55", "operaciones": ["CONSULTA", "TRANSFERENCIA"] },
  { "id": "c-002", "numero": "0098765432", "tipo": "CORRIENTE", "alias": "Recaudo", "moneda": "COP",
    "saldoDisponible": "1873402119.10", "operaciones": ["CONSULTA", "TRANSFERENCIA"] },
  { "id": "c-003", "numero": "0077001122", "tipo": "COMPENSACION", "alias": "Exportaciones", "moneda": "USD",
    "saldoDisponible": "312450.75", "operaciones": ["CONSULTA"] }
] }
```

Movimientos del mes en curso: c-001 no tiene movimientos este mes; c-002 tiene 50.000; c-003 tiene 38. Primera página de c-002:

```json
{ "items": [
  { "id": "m-50000", "fechaHora": "2026-11-19T08:42:17-05:00", "descripcion": "RECAUDO PUNTO 0412 EFECTIVO",
    "referencia": "RC-0412-191126-0087", "tipo": "CREDITO", "monto": "1250000.00", "estado": "APLICADO" },
  { "id": "m-49999", "fechaHora": "2026-11-19T08:41:03-05:00", "descripcion": "PAGO PROVEEDOR <LACTEOS DEL VALLE> FV-2231",
    "referencia": "TRF-90112", "tipo": "DEBITO", "monto": "18450300.50", "estado": "EN_PROCESO" },
  { "id": "m-49998", "fechaHora": "2026-11-19T00:03:55-05:00", "descripcion": "COMISION RECAUDO OCTUBRE",
    "referencia": "CM-2610", "tipo": "DEBITO", "monto": "0.10", "estado": "APLICADO" }
], "totalRegistros": 50000, "siguienteCursor": "eyJpZCI6Im0tNDk5OTgifQ" }
```

Primera página de c-003:

```json
{ "items": [
  { "id": "x-38", "fechaHora": "2026-11-18T15:20:00-05:00", "descripcion": "PAGO EXPORTACION FACT EXP-0091",
    "referencia": "SWIFT-7781", "tipo": "CREDITO", "monto": "48200.00", "estado": "APLICADO" },
  { "id": "x-37", "fechaHora": "2026-11-18T15:20:00-05:00", "descripcion": "COMISION GIRO EXTERIOR",
    "referencia": "SWIFT-7781-C", "tipo": "DEBITO", "monto": "35.00", "estado": "APLICADO" }
], "totalRegistros": 38, "siguienteCursor": "eyJpZCI6IngtMzcifQ" }
```

Bancos destino: 1001 Banco Arrayán, 1002 Banco del Páramo, 1003 Banco Litoral, 1004 Cooperativa Financiera Guadua.

INC-1103 · Transferencias duplicadas · portal actual · 3 de noviembre de 2026
Fuentes: [met] métricas, [rum] monitoreo de usuarios reales, [red] trazas de red del navegador, [con] consola del navegador, [bff] registro del BFF.

```text
01 [met] 27-oct a 02-nov  transferencias duplicadas reportadas por día: 0 0 0 1 0 0 0
02 [met] 02-nov 19:05  despliegue portal 3.4.0: "cliente HTTP con reintentos automáticos para mejorar la resiliencia"
03 [met] 03-nov  transferencias duplicadas reportadas: 23 (21 entre 09:50 y 10:40)
04 [met] 03-nov 09:50-10:40  latencia del núcleo bancario p95 9,8 s (habitual 1,2 s); proceso de cierre de nómina en curso
05 [met] 03-nov  navegador en los 23 casos: Chrome 12, Edge 9, Firefox 2
06 [con] s-7f3a 10:13:55.002 info http-client 3.4.0 reintentos=2 metodos=GET,POST espera=0ms
07 [rum] s-7f3a 10:14:02.118 clic boton=transferir pantalla=transferencia
08 [rum] s-7f3a 10:14:02.120 boton=transferir deshabilitado
09 [red] s-7f3a 10:14:02.121 POST /api/v1/transferencias Idempotency-Key=9b1e47c2 inicio
10 [bff] 10:14:02.140 POST /transferencias usuario=u-2231 llave=9b1e47c2 monto=8450000.00 -> núcleo
11 [bff] 10:14:12.141 núcleo sin respuesta en 10000 ms llave=9b1e47c2 -> 504 TIEMPO_AGOTADO
12 [red] s-7f3a 10:14:12.150 POST /api/v1/transferencias Idempotency-Key=9b1e47c2 estado=504 duración=10029 ms
13 [con] s-7f3a 10:14:12.151 warn http-client reintento 1/2 POST /api/v1/transferencias motivo=504
14 [red] s-7f3a 10:14:12.153 POST /api/v1/transferencias Idempotency-Key=c44d0a91 inicio
15 [bff] 10:14:12.170 POST /transferencias usuario=u-2231 llave=c44d0a91 monto=8450000.00 -> núcleo
16 [bff] 10:14:12.902 núcleo respuesta tardía llave=9b1e47c2 estado=APLICADA id=TRF-88122 (cliente ya desconectado)
17 [bff] 10:14:13.010 núcleo llave=c44d0a91 estado=APLICADA id=TRF-88123 -> 201
18 [red] s-7f3a 10:14:13.020 POST /api/v1/transferencias Idempotency-Key=c44d0a91 estado=201 duración=867 ms
19 [rum] s-7f3a 10:14:13.025 pantalla=comprobante transferencia=TRF-88123
20 [rum] s-9c21 10:21:40.310 clic boton=transferir pantalla=transferencia
21 [rum] s-9c21 10:21:40.312 boton=transferir deshabilitado
22 [red] s-9c21 10:21:40.313 POST /api/v1/transferencias Idempotency-Key=5e02bb17 inicio
23 [bff] 10:21:40.330 POST /transferencias usuario=u-0874 llave=5e02bb17 -> núcleo
24 [rum] s-9c21 10:21:40.498 clic boton=transferir ignorado (deshabilitado)
25 [bff] 10:21:41.390 núcleo llave=5e02bb17 estado=APLICADA id=TRF-88301 -> 201
26 [red] s-9c21 10:21:41.402 POST /api/v1/transferencias Idempotency-Key=5e02bb17 estado=201 duración=1089 ms
27 [rum] s-9c21 10:21:41.405 pantalla=comprobante transferencia=TRF-88301
28 [bff] 10:30:05.611 POST /transferencias usuario=u-1560 llave=e7a3f0d4 -> núcleo
29 [bff] 10:30:15.612 núcleo sin respuesta en 10000 ms llave=e7a3f0d4 -> 504 TIEMPO_AGOTADO
30 [con] s-2d77 10:30:15.639 warn http-client reintento 1/2 POST /api/v1/transferencias motivo=504
31 [bff] 10:30:15.640 POST /transferencias usuario=u-1560 llave=0b9d6e25 -> núcleo
32 [bff] 10:30:16.020 núcleo respuesta tardía llave=e7a3f0d4 estado=APLICADA id=TRF-88415 (cliente ya desconectado)
33 [bff] 10:30:16.300 núcleo llave=0b9d6e25 estado=APLICADA id=TRF-88416 -> 201
34 [bff] 10:30:16.301 control de duplicados: llave 0b9d6e25 sin coincidencias en 24 h
35 [red] s-2d77 10:36:44.004 GET /api/v1/cuentas estado=504 duración=10011 ms
36 [con] s-2d77 10:36:44.006 warn http-client reintento 1/2 GET /api/v1/cuentas motivo=504
37 [red] s-2d77 10:36:44.390 GET /api/v1/cuentas estado=200 duración=384 ms
38 [met] 03-nov  respuestas 504 en POST /transferencias: 31; de ellas, con la transferencia aplicada después en el núcleo: 23
39 [met] 03-nov  el BFF no recibió dos veces la misma llave en ninguno de los 23 casos
40 [met] 04-nov 08:00  reintentos automáticos desactivados en el portal; duplicados reportados ese día: 0
```

### 4.2 Contrato de API o sistema bajo prueba

Base: /api/v1, servida por el BFF en el mismo dominio del portal.

Sesión: el portal crea al iniciar sesión la cookie sid (HttpOnly; Secure; SameSite=Strict); el navegador la adjunta a cada petición. Cada respuesta exitosa renueva la sesión por 5 minutos. Todas las respuestas traen el encabezado X-CSRF-Token; toda petición POST debe enviarlo de vuelta en un encabezado con el mismo nombre.

Errores, en cualquier endpoint: cuerpo { "codigo": string, "mensaje": string, "campos"?: [ { "campo": string, "codigo": string } ] }. 401 SESION_EXPIRADA. 429 LIMITE_PETICIONES y 503 NO_DISPONIBLE, ambos con el encabezado Retry-After en segundos.

1. GET /cuentas
   200: { "cuentas": [ Cuenta ] }. Cuenta: id, numero, tipo (AHORROS | CORRIENTE | COMPENSACION), alias, moneda (código ISO 4217), saldoDisponible (decimal en texto, dos decimales), operaciones (CONSULTA | TRANSFERENCIA).

2. GET /cuentas/{id}/movimientos?desde=AAAA-MM-DD&hasta=AAAA-MM-DD&q=&cursor=&limite=
   desde y hasta son obligatorios, con un rango máximo de 31 días. q busca en descripción y referencia (mínimo 3 caracteres). limite va de 1 a 200; por defecto 50. Orden fijo: fechaHora descendente.
   200: { "items": [ Movimiento ], "totalRegistros": entero, "siguienteCursor": texto o null }. Movimiento: id, fechaHora (ISO 8601 con zona), descripcion (texto libre, puede venir de terceros), referencia, tipo (DEBITO | CREDITO), monto (decimal positivo en texto, dos decimales), estado (APLICADO | EN_PROCESO).
   Latencia observada: de 80 a 900 ms por página; p95 350 ms.
   400 RANGO_INVALIDO.

3. POST /transferencias
   Encabezados obligatorios: X-CSRF-Token e Idempotency-Key (UUID).
   Cuerpo: cuentaOrigenId, bancoDestino (código), tipoCuentaDestino (AHORROS | CORRIENTE), numeroCuentaDestino (6 a 17 dígitos), tipoDocumentoTitular (CC | CE | NIT | PA), numeroDocumentoTitular, nombreTitular, monto (decimal en texto), descripcion (opcional, hasta 40 caracteres), claveDinamica (6 dígitos).
   201: { "id", "estado": "APLICADA" | "EN_PROCESO", "fechaHora", "costo" }.
   Una Idempotency-Key ya recibida en las últimas 24 horas con el mismo cuerpo devuelve la respuesta original con 200; con otro cuerpo, 409 IDEMPOTENCIA_CONFLICTO.
   422 con codigo SALDO_INSUFICIENTE, TOPE_DIARIO_EXCEDIDO, CLAVE_DINAMICA_INVALIDA, CUENTA_ORIGEN_NO_PERMITIDA o VALIDACION (con campos).
   504 TIEMPO_AGOTADO si el núcleo bancario no responde en 10 segundos.

### 4.3 Glosario del dominio

- BFF: servicio del banco que expone la API a las aplicaciones cliente.
- Cuenta de recaudo: cuenta corriente donde la empresa recibe los pagos de sus clientes.
- Cuenta de compensación: cuenta en dólares registrada ante el Banco de la República para operaciones de comercio exterior. En la banca en línea solo admite consultas.
- Referencia de pago: código que identifica un pago recibido; la tesorera la usa para conciliar.
- Movimiento en proceso: movimiento aceptado que el núcleo bancario aún no ha aplicado al saldo.
- Clave dinámica: código de 6 dígitos que genera el token del usuario; vale 60 segundos.
- Tope diario: monto máximo que un usuario puede transferir a terceros en un día: 20.000.000 pesos.
- Idempotency-Key: identificador que el cliente envía con cada transferencia; el BFF no aplica dos veces una transferencia con la misma llave.
- Núcleo bancario: sistema central que aplica las transferencias y los movimientos.


## 5. Lo que debes resolver

### 5.1 Parte 1: Aplicación y flujo de datos (30 min)

Construir desde cero la aplicación cliente, simular la API a partir del contrato de 3 endpoints y la pantalla de consulta con sus estados de carga, vacío y error, con pruebas de componentes.

La pantalla de consulta permite elegir una cuenta y muestra su saldo disponible y sus movimientos del mes, con fecha y hora, descripción, referencia, tipo, monto y estado. Débitos y créditos se distinguen a simple vista. Los montos llevan separador de miles, dos decimales y el símbolo de la moneda. Explica en el README cómo se activa la simulación. Aquí basta la primera página de movimientos; búsqueda y volumen son de la Parte 3.

Entregables de esta parte: código de la aplicación, API simulada, pruebas de componentes, README.md.

### 5.2 Parte 2: Formulario seguro y accesible (25 min)

Construir el formulario descrito, con datos sensibles, validación, manejo de la sesión, accesibilidad WCAG 2.2 AA y funcionamiento en Google Chrome, Microsoft Edge y Mozilla Firefox.

El formulario envía una transferencia a terceros con los campos de POST /transferencias. Ofrece como origen solo las cuentas que permiten la operación, con su saldo disponible. Valida en el cliente lo posible y muestra los errores del servidor junto al campo correspondiente o en un mensaje general. Al terminar muestra el comprobante con el número y el estado. Cuando la sesión deja de ser válida, la aplicación lleva al usuario a /ingreso, en el portal. Simula las respuestas 401, 409, 422 y 504.

Entregables de esta parte: código del formulario, pruebas del formulario.

### 5.3 Parte 3: Rendimiento, incidente y extensión (20 min)

Mostrar los 50000 registros descritos sin degradar la experiencia, controlar las búsquedas concurrentes, analizar el incidente descrito, registrar los riesgos técnicos y dejar una guía para que un desarrollador junior agregue una pantalla.

Lleva la consulta a la cuenta de recaudo completa: genera en la simulación los 50.000 movimientos y agrega una búsqueda por descripción o referencia. La tesorera debe poder recorrer el mes y encontrar una referencia sin ver resultados de una búsqueda anterior, incluso si cambia de cuenta mientras escribe.

INCIDENTE.md: analiza INC-1103 (sección 4.1) con hipótesis descartadas y su evidencia, causa raíz, corrección y cómo tu aplicación evita que se repita.

RIESGOS.md: riesgos técnicos de tu solución con probabilidad, impacto y mitigación.

GUIA-EXTENSION.md: lo necesario para que un desarrollador junior agregue sin ayuda una futura pantalla de destinatarios inscritos.

Entregables de esta parte: INCIDENTE.md, RIESGOS.md, GUIA-EXTENSION.md.


## 6. Requisitos no funcionales y restricciones

- Navegadores: Chrome, Edge y Firefox en sus dos últimas versiones estables, en equipos de oficina con 4 GB de RAM.
- Accesibilidad: WCAG 2.2 nivel AA.
- Consulta: primeras filas visibles en 1,5 s o menos tras elegir la cuenta; desplazarse y escribir no bloquean la interfaz más de 200 ms; resultados de búsqueda en 1 s o menos después de dejar de escribir.
- El BFF admite máximo 10 peticiones por segundo por sesión.
- El banco cierra la sesión tras 5 minutos sin actividad. Cuando la sesión termina no queda en el navegador ningún dato de la transferencia.
- La política de seguridad de contenido no permite scripts en línea ni recursos de otros dominios.
- Los montos se muestran sin alterar el valor que entrega la API.

## 7. Verificación y calidad

Tu entrega debe incluir pruebas automatizadas que se ejecuten con un solo comando documentado en el README, y al menos una verificación no funcional: una revisión automática de accesibilidad y una medición de rendimiento de la vista con más registros. Indica en DECISIONES.md qué probaste, qué no y por qué.

## 8. Registro de decisiones

DECISIONES.md es obligatorio y se evalúa tanto como la solución. Registra cada decisión que tomes, no solo las grandes: tecnología, arquitectura, patrones, diseño, datos, seguridad, rendimiento, pruebas y lo que decidiste no hacer. Para cada una explica qué alternativas consideraste y por qué elegiste la tuya en este contexto. En la sustentación tendrás que defender cada decisión registrada.

```markdown
# Registro de decisiones — DS-FE-20261119-m2f8
Inicio: <hora> · Fin de la Parte 1: <hora> · Fin: <hora>

## Supuestos
| # | Supuesto | Por qué lo asumí | Qué cambiaría si es falso |
|---|---|---|---|

## Decisiones
### D-01 <Título corto>
- **Tema:** stack | arquitectura | patrón | diseño | datos | seguridad | rendimiento | pruebas | operación | proceso | otro
- **Contexto:** qué problema o restricción obliga a decidir.
- **Alternativas consideradas:** al menos dos, con pros y contras en este contexto.
- **Decisión:** qué elegí.
- **Por qué:** la razón determinante, con datos del enunciado si aplica.
- **Consecuencias y trade-offs:** qué gano, qué sacrifico, qué deuda acepto.
- **Dónde se ve:** archivo, componente, commit o artefacto.

(Una entrada por cada decisión: no solo las grandes. Incluye lo que decidiste no hacer.)

## Lo que dejé fuera y por qué

## Si tuviera más tiempo
```

## 9. Uso de IA

Puedes usar asistentes de IA durante la prueba. No necesitas entregar tus conversaciones con la IA.
Ten presente que la evaluación no termina con la entrega. En una sustentación de 30 minutos, sin asistencia de IA, tendrás que:
- explicar tu solución de memoria y recorrerla sin ayuda;
- defender cada decisión de DECISIONES.md y las alternativas que descartaste;
- adaptar tu solución en vivo a un cambio que se te planteará;
- responder por cualquier parte de lo que entregaste.

Lo que no puedas explicar o defender en la sustentación no cuenta como evidencia, aunque esté en la entrega. Usa la IA como apoyo, pero toma tú cada decisión y asegúrate de entender todo lo que entregas.

## 10. Criterio de completitud

Tu prueba está completa cuando se cumple todo lo siguiente:
- Parte 1: La pantalla de consulta funciona contra la API simulada en sus tres estados y las pruebas de componentes pasan con un solo comando.
- Parte 2: El formulario valida, protege los datos sensibles y la sesión, y pasa la revisión automática de accesibilidad.
- Parte 3: La vista maneja el volumen sin bloquear la interfaz, INCIDENTE.md tiene causa raíz con evidencia y GUIA-EXTENSION.md permite agregar una pantalla sin ayuda.
- DECISIONES.md registra cada decisión con sus alternativas y las tres horas.

Una parte incompleta se evalúa por lo que muestra; lo que no alcanzaste a hacer se explorará en la sustentación.

## 11. Entrega

Entrega un repositorio Git con commits pequeños y descriptivos (la historia de commits se revisa) y un README con los comandos para instalar, ejecutar y probar.

Archivos obligatorios:
- `README.md`
- `DECISIONES.md`
- `INCIDENTE.md`
- `RIESGOS.md`
- `GUIA-EXTENSION.md`
