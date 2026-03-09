<div align="center">

# Habilitalo

**Tu historia es tuya. Habilitala.**

Un protocolo abierto de conectores para habilitar tu historial financiero,<br>
operativo y comercial — desde cualquier fuente, en minutos.

`Protocolo abierto` `TypeScript` `Agnóstico al dominio` `Licencia MIT`

[Website](https://habilitalo.com) · [Whitepaper](https://habilitalo.com/whitepaper.html) · [GitHub](https://github.com/habilitalo/protocolo)

</div>

---

## El problema

La integración de datos — financieros, operativos y comerciales — está rota de tres maneras:

| Problema | Qué pasa |
|----------|----------|
| **Fragmentación** | Cada fuente requiere su propia lógica de parsing, formatos de fecha, normalización de montos. Los equipos construyen esto desde cero cada vez. |
| **Lock-in** | Los agregadores controlan el formato, la entrega, el calendario de sync y el manejo de errores. Cambiar de agregador = reescribir tu capa de ingestión. |
| **Extracción de renta** | USD 0.50/conexión/mes suena razonable — hasta que tu cliente PYME tiene 3 cuentas y paga USD 5/mes. El agregador se lleva el 30%. |

**Habilitalo es el protocolo abierto.** Separa el *qué* de la integración (qué datos entran, qué eventos salen) del *cómo* (el pipeline que los procesa).

---

## Cómo funciona

Un Conector es una declaración + una función de transformación. Tres pasos:

### 1. Declara tu conector

Define un manifiesto JSON que describe tu fuente de datos y los tipos de evento que habilita.

```json
{
  "id": "fintoc-v1",
  "name": "Fintoc Open Banking",
  "version": "1.0.0",
  "auth": "api_key",
  "events": [
    "bank.movement.created",
    "bank.account.synced"
  ]
}
```

### 2. Transforma los datos crudos

Escribe una función pura que mapea datos crudos a eventos canónicos. Tu historial completo, sin perder un día.

```typescript
const transform: HabilitaloTransform = (raw, ctx) => ({
  id: crypto.randomUUID(),
  timestamp: new Date().toISOString(),
  source: 'habilitalo',
  type: 'bank.movement.created',
  version: '1.0.0',
  payload: {
    amount: raw.amount,
    currency: mapCurrency(raw.currency),
    description: raw.description,
    counterparty: raw.sender_account?.holder_name,
    date: raw.post_date,
  },
  meta: {
    connector: 'fintoc-v1',
    tenant: ctx.tenantId,
    trace_id: ctx.traceId,
  },
})
```

### 3. Emite eventos canónicos

Los eventos fluyen al ecosistema. El día uno de tu nuevo sistema tiene toda la historia del anterior.

```
Fuente de datos → Conector → Evento Canónico → Ecosistema
```

---

## Evento Canónico

Cada conector emite eventos en una forma canónica única — el sobre universal que fluye a través de todo el sistema:

```typescript
interface CanonicalEvent {
  id: string           // UUID único
  timestamp: string    // ISO 8601
  source: 'habilitalo'
  type: string         // e.g. 'bank.movement.created'
  version: string      // Versión del esquema
  payload: Record<string, unknown>
  meta: EventMeta      // connector, tenant, trace_id
}
```

---

## Pipeline de 3 etapas

El pipeline toma datos crudos de cualquier fuente y produce asientos contables balanceados, eventos emitidos y resultados de análisis:

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────┐
│  1. Ingestión │ ──→ │  2. Procesamiento │ ──→ │  3. Análisis  │
│              │     │                  │     │              │
│  Raw → Stage │     │  Stage → Ledger  │     │  Anomalías   │
│  Dedup hash  │     │  Partida doble   │     │  Recurrentes │
└──────────────┘     └──────────────────┘     └──────────────┘
```

| Etapa | Entrada | Salida | Responsabilidad |
|-------|---------|--------|-----------------|
| **Ingestión** | Datos crudos de la fuente | `RawEntry` en staging | Parsing, normalización, deduplicación por SHA-256 |
| **Procesamiento** | `RawEntry` pendientes | `JournalEntry` con partida doble | Clasificación, posteo al libro mayor, validación de balance |
| **Análisis** | Asientos del libro mayor | Anomalías y patrones | Detección de anomalías, identificación de recurrentes |

Cada etapa es secuencial y desacoplada — una falla en una etapa no corrompe las demás.

---

## Conectores

Cada conector habilita una fuente de datos al protocolo. Activos hoy o en camino:

| Conector | Tipo | Región | Estado |
|----------|------|--------|:------:|
| **Fintoc Open Banking** | Banking | Chile / LATAM | Activo |
| **Importación CSV** | Archivo | Global | Activo |
| **Entrada Manual** | Formulario | Global | Activo |
| SAT México | Impuestos | México | Próximamente |
| Stripe | Pagos | Global | Próximamente |
| QuickBooks | Contabilidad | USA | Próximamente |

> Construir un nuevo conector = escribir una función de transformación + un manifiesto JSON. El pipeline se encarga del resto.

---

## Principios de diseño

| # | Principio | |
|:-:|-----------|---|
| 1 | **Declarativo** | Un conector se define, no se programa. Manifiesto JSON + función pura. |
| 2 | **Agnóstico al dominio** | Nació en finanzas, pero cualquier fuente — operativa, comercial, clínica — puede expresarse como conector. |
| 3 | **Sin lock-in** | Protocolo abierto, MIT. Tus datos, tus reglas. Cambiar de implementación no requiere reescribir conectores. |
| 4 | **Verificable** | Deduplicación por SHA-256, partida doble balanceada, trazabilidad completa por evento. |
| 5 | **Funciones puras** | Los transforms no tienen side effects. Reciben datos crudos, retornan eventos canónicos. Testeable, reproducible. |
| 6 | **Infraestructura compartida** | Deduplicación, posteo al libro mayor, entrega de eventos, detección de anomalías — todo reutilizable entre conectores. |

---

## Arquitectura — El stack Digitalo

Habilitalo es la capa de habilitación de datos del ecosistema:

```
External World → HABILITALO → Servicialo → Compensalo → Coordinalo
(Bancos, APIs)    (Habilita)   (Ejecuta)    (Paga)       (Agenda)
```

| Capa | Protocolo | Rol |
|------|-----------|-----|
| **Habilitalo** | Open Connector Protocol | Habilita historial desde cualquier fuente |
| [Servicialo](https://servicialo.com) | Service Orchestration Protocol | Orquesta servicios profesionales |
| Compensalo | Settlement Protocol | Liquida pagos entre partes |
| Coordinalo | Scheduling Protocol | Coordina agendas y recursos |

---

## En este repositorio

```
habilitalo/
├── index.html          # Landing page (habilitalo.com)
├── whitepaper.html     # Whitepaper técnico completo
├── og-image.svg        # Open Graph image
└── README.md
```

|  | Versión | Estado |
|---|---------|--------|
| Protocolo | 0.1 | Whitepaper publicado |
| Website | — | [habilitalo.com](https://habilitalo.com) |

---

## Contribuir

El estándar de mapeo es público. Tú dictas las reglas del lenguaje.

1. Escribe un transform
2. Declara un manifiesto
3. Abre un PR

Lee el [whitepaper](https://habilitalo.com/whitepaper.html) para la especificación completa.

---

## Licencia

MIT — Habilitalo es un protocolo abierto. Cualquiera puede implementarlo.
