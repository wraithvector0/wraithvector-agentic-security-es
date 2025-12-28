# WraithVector AI Systems  
## Guía de Seguridad Agentic  
### Gobernanza en Tiempo de Ejecución para Sistemas LLM Autónomos

---

## Página 1 — Introducción

### 1.1 De los LLM a los Sistemas Agentic

Los Large Language Models ya no son generadores de texto aislados.  
Los sistemas de IA modernos operan cada vez más como **agentes**:

- mantienen estado y memoria  
- encadenan razonamiento en múltiples pasos  
- llaman a herramientas y APIs externas  
- interactúan con sistemas reales de negocio  
- operan con autonomía parcial o total  

Esta transición cambia de forma fundamental el modelo de seguridad.

### 1.2 Tesis central

> **En sistemas agentic, los fallos de seguridad no se producen por lo que el modelo dice,  
sino por lo que el sistema le permite hacer.**

La seguridad clásica basada en prompts se centra en el lenguaje.  
La seguridad agentic debe centrarse en la **autoridad, los límites de decisión y la aplicación de políticas en runtime**.

---

## Página 2 — Anatomía de un Sistema Agentic

### 2.1 Componentes principales

Una arquitectura agentic típica incluye:

- LLM (razonamiento y generación)  
- Orquestador de agentes  
- Memoria (historial de conversación, RAG, estado)  
- Interfaces de herramientas (APIs, funciones, acciones)  
- Lógica de negocio backend  
- Usuarios o sistemas externos  

### 2.2 Flujo de ejecución

```text
Input → LLM → Output
               ↓
           Memoria / Estado
               ↓
        Decisión / Llamada a herramienta
               ↓
           Acción en el mundo real
```

El límite crítico de seguridad es la transición entre el output del modelo y la acción operativa.

# Suposiciones del Modelo de Amenaza

- Esta guía asume el siguiente modelo de amenaza para sistemas de IA agentic:

- El LLM es no determinista y no puede considerarse fiable para aplicar políticas de seguridad.

- Todas las entradas externas (usuarios, herramientas, APIs, agentes upstream) son no confiables por defecto.

- Los outputs del modelo son datos no confiables, no decisiones autorizadas.

- Cualquier acción que afecte a sistemas externos representa una superficie de ataque de alto impacto.

- La memoria, la persistencia de estado y la reutilización de contexto amplifican el riesgo con el tiempo.

- Los sistemas multi-agente aumentan la probabilidad de contaminación entre contextos y agentes.

- Los controles de seguridad deben operar fuera del modelo, en tiempo de ejecución.

Este modelo trata explícitamente al LLM como un componente potencialmente defectuoso,
no como una entidad de decisión confiable.

## Página 3 — Ataques Clásicos de Prompt (Taxonomía Palo Alto)

### 3.1 Visión general

Los ataques clásicos documentados por Palo Alto y otros actores del sector incluyen:

- **Prompt Injection** (prefijo / sufijo)  
- **Roleplay** y **Storytelling**  
- **Ofuscación** y **codificación**  
- **Tokens repetidos** / *glitch tokens*  
- **Few-Shot / Many-Shot Poisoning**  
- **Prompt Leakage**

Estos ataques se centran principalmente en **manipular la interpretación lingüística del modelo**.

No buscan ejecutar acciones directamente, sino influir en:
- el razonamiento del LLM,
- el contenido generado,
- o la percepción de contexto y reglas.

---

### 3.2 Limitaciones de los ataques clásicos

Por sí solos, estos ataques pueden:

- influir en la generación de texto,
- provocar filtraciones de información,
- evadir filtros de contenido o alineación.

Sin embargo, **no generan impacto real en el mundo físico o empresarial**
a menos que sus *outputs* sean **reutilizados de forma operativa** por el sistema.

> En sistemas agentic bien diseñados,  
> el lenguaje no tiene autoridad operacional.

Los ataques clásicos solo se vuelven críticos cuando:
- el output del modelo se guarda como memoria,
- se reutiliza como estado,
- o se emplea como input para decisiones posteriores.

En ese punto, el problema deja de ser lingüístico  
y pasa a ser **arquitectónico**.

## Página 4 — Por qué los ataques clásicos siguen importando

### 4.1 Ataques clásicos como vectores de entrada

Aunque los ataques clásicos de prompt ya no suelen ser el exploit final,
siguen siendo relevantes como **vectores de entrada** en sistemas agentic.

En la práctica, se utilizan para:

- obtener acceso inicial al comportamiento del modelo,
- realizar reconocimiento del sistema,
- influir en el contexto o estado downstream,
- preparar ataques más sofisticados a nivel de agente.

En arquitecturas modernas, estos ataques **no rompen el sistema por sí solos**.
Sin embargo, actúan como el **primer eslabón** de una cadena de ataque más amplia.

---

### 4.2 Ruta típica de escalada

La siguiente secuencia muestra cómo un ataque clásico basado en prompts
puede escalar hasta convertirse en un fallo real de seguridad agentic
cuando los *outputs* del modelo se reutilizan sin gobernanza.

```text
Ofuscación o Prompt Injection
        ↓
El modelo genera output engañoso o peligroso
        ↓
El output se guarda como memoria o estado
        ↓
La memoria o el contexto se reutilizan
        ↓
Ejecución no autorizada de herramientas o decisiones
```

## Página 5 — Bucles Output → Input (el fallo central)

### 5.1 Definición

Un **bucle output → input** ocurre cuando:

- el LLM produce un output,
- ese output se guarda como memoria, estado o contexto,
- el sistema reutiliza ese output como si fuera input autorizado
  para decisiones posteriores.

En sistemas agentic, este patrón aparece de forma natural
cuando se encadenan razonamientos, agentes o pasos de ejecución.

---

### 5.2 Por qué es peligroso

El modelo puede generar:

- opiniones (“esto parece seguro”),
- clasificaciones (“riesgo bajo”),
- sugerencias (“se puede continuar”),
- aparentes autorizaciones.

Si el sistema no distingue entre:
- **opinión del modelo**, y
- **decisión autorizada**,

el output del LLM puede convertirse en **autoridad implícita**.

Este es el mecanismo por el cual un sistema permite que
el modelo se **auto-autorice accidentalmente**.

> El LLM no rompe las reglas.  
> El sistema le permite escribirlas.

---

## Página 6 — La memoria como superficie de ataque

### 6.1 Qué significa realmente “memoria”

En sistemas agentic, la memoria no es un concepto abstracto.
Incluye cualquier mecanismo que **persista información generada por el modelo**, como:

- historial de conversación,
- embeddings de RAG,
- variables de estado del agente,
- contexto compartido entre agentes,
- razonamientos o decisiones previas almacenadas.

La memoria convierte texto efímero en **estado operativo persistente**.

---

### 6.2 Ejemplo: fallo en un flujo KYC

**Sistema inseguro**

1. El usuario afirma ser un cliente legítimo.  
2. El LLM responde: “usuario de bajo riesgo”.  
3. El sistema guarda el estado: `riesgo = bajo`.  
4. Otro agente reutiliza esa memoria.  
5. El usuario es aprobado sin verificación adicional.

No se produjo ningún exploit técnico.  
No hubo bypass criptográfico ni acceso no autorizado.

El fallo fue **arquitectónico**:
el sistema trató un output no confiable como si fuera
una decisión válida.

> En sistemas agentic,  
> la memoria es una superficie de ataque tan crítica como una API.

## Página 7 — Amenazas nativas de agentes (el riesgo real)

Las amenazas nativas de agentes (**agent-native threats**) solo existen
cuando los LLM operan con **autonomía, memoria y capacidad de acción**.
No dependen de romper el lenguaje, sino de **romper la autoridad**.

### 7.1 Manipulación del uso de herramientas (Tool-Use Manipulation)

Los agentes pueden **proponer acciones reales** sobre sistemas externos, como:

- transferencias de fondos,
- aprobaciones de usuarios,
- acceso o modificación de datos,
- cambios en infraestructura.

Si estas acciones no se validan en runtime,
el resultado es una forma de **RCE lógico**.

**Severidad:** Crítica (8–9/10)

---

### 7.2 Inyección entre agentes (Cross-Agent Injection)

En sistemas multi-agente, el output de un agente
puede convertirse en input de otro.

Esto permite:

- propagación de estado falso,
- escalada de privilegios en cascada,
- contaminación de decisiones posteriores.

**Severidad:** Alta (8/10)

---

### 7.3 Envenenamiento de memoria / RAG

El contexto persistente (memoria, RAG, estado)
puede gobernar decisiones futuras durante largos periodos.

Un estado envenenado produce
**comportamiento incorrecto de forma sostenida**.

**Severidad:** Media–Alta (6–8/10)

---

### 7.4 Confusión de políticas (Multi-Tenant)

Se aplican políticas incorrectas de:

- tenant,
- caso de uso,
- o nivel de autorización.

Este fallo tiene impacto directo en
**seguridad, cumplimiento y responsabilidad legal**.

**Severidad:** Crítica (9/10)

## Página 8 — Relación entre ataques Palo Alto y riesgo agentic

La siguiente tabla correlaciona ataques clásicos basados en prompts
(documentados por Palo Alto) con su impacto real en sistemas agentic.

| Ataque Palo Alto          | Descripción breve                         | Impacto en sistemas agentic        | Nivel de riesgo |
|---------------------------|-------------------------------------------|------------------------------------|-----------------|
| Prompt Injection          | Manipulación de instrucciones             | Vector de entrada                  | Bajo            |
| Ofuscación / Encoding     | Ocultar intención mediante ruido          | Bypass de detección                | Bajo            |
| Few-Shot / Many-Shot      | Enseñar políticas falsas en contexto      | Envenenamiento de estado           | Medio           |
| Prompt Leakage            | Filtrado de contexto o capacidades        | Reconocimiento                     | Bajo–Medio      |
| Roleplay / Storytelling   | Confusión de autoridad                    | Confusión de políticas             | Bajo            |
| Payload Splitting         | Fragmentación multi-paso del payload      | Manipulación de estado             | Medio           |

**Conclusión clave:**

> Los ataques clásicos rara vez causan daño directo.  
> El riesgo real aparece cuando los *outputs* del modelo
> se reutilizan como memoria, estado o input operativo
> sin gobernanza en runtime.

## Página 9 — Principios de Seguridad WraithVector

Los siguientes principios definen la base de la seguridad agentic en WraithVector.
No son recomendaciones, sino **invariantes de diseño**.

---

### Principio 1 — Autoridad fuera del modelo

El LLM **nunca** decide permisos, autorizaciones ni niveles de acceso.

El modelo puede:
- razonar,
- proponer,
- sugerir.

Pero **no puede gobernar**.

---

### Principio 2 — Enforcement en tiempo de ejecución (RunSec)

Toda acción propuesta por un agente es
**interceptada y evaluada en runtime**.

La decisión final se toma:
- fuera del modelo,
- de forma determinista,
- basada en políticas explícitas.

---

### Principio 3 — Políticas por tenant y caso de uso

No existen políticas globales.

Cada decisión se evalúa en función de:
- el tenant,
- el caso de uso,
- el contexto operativo.

Esto evita:
- fugas entre clientes,
- reutilización indebida de permisos,
- confusión de autoridad.

---

### Principio 4 — No Output-as-Truth

El output del modelo **no es estado autorizado**.

Ninguna decisión puede basarse exclusivamente en:
- texto generado,
- clasificaciones implícitas,
- razonamientos previos del LLM.

---

### Principio 5 — Clasificación y control de memoria

La memoria debe tratarse como
una **superficie de ataque crítica**.

Es obligatorio:
- diferenciar contexto confiable y no confiable,
- controlar qué se persiste,
- limitar qué se reutiliza en decisiones futuras.

> En sistemas agentic,  
> la memoria sin gobernanza equivale a ejecución sin control.


flowchart TD
    U[Usuario / Sistema Externo] -->|Input| LLM[LLM / Razonamiento del agente]

    LLM -->|Propuesta de acción| RS[RunSec Gateway Runtime]

    RS -->|Consulta política| PDB[(Policy Store<br/>Supabase)]
    PDB --> RS

    RS -->|ALLOW| TOOL[Ejecución de Tool / API]
    RS -->|BLOCK| BLK[Acción bloqueada]
    RS -->|ESCALATE| HUM[Revisión humana]

    RS --> LOG[(Audit Log Inmutable<br/>Hash-Chained)]

    TOOL --> LOG
    BLK --> LOG
    HUM --> LOG

Propiedades clave:

- El LLM no ejecuta acciones directamente
- 
- Todas las decisiones pasan por RunSec

- Políticas por tenant y caso de uso

- Logs inmutables a nivel de decisión

- Idioma y modelo irrelevantes en enforcement

## Página 12 — Cumplimiento normativo y preparación para auditoría

Los sistemas agentic introducen nuevos retos de cumplimiento,
especialmente en entornos regulados como banca, seguros o fintech.

WraithVector permite:

- trazabilidad completa a nivel de decisión,
- registro de cada acción propuesta y su resultado,
- separación clara entre razonamiento del modelo y decisión autorizada,
- evidencias técnicas auditables y reproducibles.

### 12.1 Logs y evidencia

Cada decisión se registra con:

- identificador de tenant,
- caso de uso,
- acción propuesta,
- decisión final (ALLOW / BLOCK / ESCALATE),
- marca temporal,
- hash encadenado para inmutabilidad.

Esto permite construir **cadenas de evidencia** compatibles con
requisitos de auditoría.

### 12.2 Alineación regulatoria

La arquitectura está diseñada para ser **alineable** con marcos como:

- AI Act (UE),
- DORA,
- requisitos internos de control y supervisión.

> El cumplimiento no se logra con alineación declarativa,  
> sino con **evidencia técnica verificable**.

## Página 13 — Conclusión final

La seguridad basada únicamente en prompts es
**necesaria pero insuficiente** para sistemas agentic.

Cuando los LLM:
- mantienen estado,
- toman decisiones,
- llaman a herramientas,
- y afectan a sistemas reales,

la seguridad debe centrarse en **gobernar decisiones**, no lenguaje.

WraithVector introduce una separación clara entre:
- lo que el modelo **propone**,
- y lo que el sistema **permite**.

> En sistemas agentic,  
> la autoridad no puede residir en el modelo.

## Página 14 — Trabajo futuro

Esta guía establece un marco teórico y arquitectónico sólido.
Las siguientes líneas de trabajo se abordarán en fases posteriores:

- validación empírica mediante laboratorios controlados,
- experimentos reproducibles con ataques reales,
- métricas cuantitativas de reducción de riesgo,
- publicación académica basada en resultados experimentales,
- ampliación a nuevos patrones de agentes y herramientas.

Estos pasos permitirán evolucionar esta guía
hacia un **paper de investigación completo**.

## Alcance y no-objetivos

WraithVector está diseñado específicamente para abordar
**fallos de seguridad en sistemas agentic**.
No pretende resolver todos los riesgos asociados a la IA.

### No-objetivos explícitos

WraithVector **no** tiene como objetivo:

- prevenir todas las formas de prompt injection o jailbreaks,
- modificar, reentrenar o alinear modelos de lenguaje,
- garantizar la corrección semántica o factual de los outputs,
- sustituir la responsabilidad humana o de negocio,
- actuar como sistema de moderación de contenido o censura.

### Alcance real

El objetivo único y explícito de WraithVector es:

> **Evitar que outputs no confiables de modelos LLM  
> se conviertan en acciones autorizadas dentro de sistemas autónomos.**

---

> **La seguridad agentic no consiste en hacer modelos más seguros.  
> Consiste en hacer sistemas autónomos gobernables.**
