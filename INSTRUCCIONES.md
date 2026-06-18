# INSTRUCCIONES.md — Proyecto [Nombre pendiente: "MenteNova" / "Polímata" / def. final]

> Este archivo es el contexto maestro para CUALQUIER agente de IA (Claude Code, Cursor, etc.) que trabaje en este proyecto. Debe estar referenciado o pegado al inicio de cada sesión de desarrollo. Ambos desarrolladores (Persona A y Persona B) usan este mismo archivo como fuente de verdad.

---

## 1. Qué es este proyecto (resumen de una frase)

Una plataforma que permite a una persona aprender **múltiples habilidades en paralelo** (ej. idioma + instrumento + ajedrez + ciencia) sin sobrecarga cognitiva, usando un orquestador con IA que decide qué practicar cada día, intercalando tipos de carga cognitiva para maximizar retención y minimizar interferencia entre dominios.

**Hackathon**: Google "Build a business in 90 days" — Categoría *Education & Human Potential*.
**Tesis del negocio**: la competencia (Duolingo, Coursera, etc.) está diseñada para un dominio a la vez. Nuestra propuesta de valor es la orquestación multi-dominio inteligente — eso es el moat.

---

## 2. Principios de diseño NO NEGOCIABLES

Estos principios gobiernan cualquier decisión de producto o código. Si una feature los viola, se descarta o se redefine.

1. **Catálogo abierto**: el usuario puede escribir CUALQUIER dominio que quiera aprender ("aprender apicultura", "tocar ukelele"). El sistema NUNCA debe tener una lista hardcodeada de dominios soportados. Todo pasa por el Domain Decomposer (ver sección 4).
2. **Intercalado por tipo de carga cognitiva, no por tema.** El Orquestador nunca debe agendar dos actividades consecutivas del mismo `tipo_carga_cognitiva` (ver enum en sección 5). Esto es la regla central del producto — todo el valor diferencial vive aquí.
3. **Generación dinámica, no contenido estático.** Los ejercicios se generan en el momento vía LLM (Claude API), ajustados al nivel actual del usuario. No se hardcodea contenido pedagógico salvo metadata estructural (prerequisitos, marcos de referencia como CEFR para idiomas).
4. **Outputs estructurados (JSON) en TODA llamada a IA.** Nunca se parsea texto libre del LLM para lógica de negocio. Cada agente tiene un schema JSON de salida fijo (definidos en sección 6).
5. **El usuario nunca debe sentir que usa 5 apps distintas.** Toda la UI debe reforzar la sensación de "un solo perfil de desarrollo cognitivo", no pestañas separadas por dominio.

---

## 3. Stack técnico

| Capa | Tecnología | Quién la toca |
|---|---|---|
| Frontend | Next.js 14 (App Router) + TypeScript + Tailwind | Persona B (principal), Persona A (solo endpoints que consume) |
| Backend | FastAPI (Python 3.11+) | Persona A (principal) |
| Base de datos | Supabase (PostgreSQL) | Ambos — schema acordado en Prompt de Sincronización #1 |
| Auth | Supabase Auth | Persona B |
| IA | Claude API (modelo `claude-sonnet-4-6`), structured outputs en JSON | Persona A |
| Hosting | Vercel (frontend) + Railway o Render (backend) | Persona B (deploy), ambos (config) |
| Repo | Monorepo: `/frontend` y `/backend` en la raíz | — |

**Regla de monorepo**: nunca mezclar dependencias de Python y Node en el mismo `package.json`/`requirements.txt`. Cada carpeta es autocontenida.

---

## 4. Arquitectura conceptual (las 4 capas)

```
Usuario define meta ("aprender alemán")
        ↓
[CAPA 1: Domain Decomposer] → descompone la meta en árbol de nodos de habilidad
        ↓
[CAPA 2: Orquestador] → toma TODOS los árboles activos del usuario, genera el plan del día
        ↓
[CAPA 3: Generador de Ejercicios] → por cada nodo agendado hoy, genera el ejercicio concreto
        ↓
[CAPA 4: Feedback Loop] → resultado del usuario ajusta dificultad y repetición espaciada
        ↓
        (vuelve a alimentar el Orquestador para el día siguiente)
```

Cada capa es un agente/módulo independiente en el backend. Ver sección 6 para los contratos exactos de cada uno.

---

## 5. Enum crítico: Tipo de Carga Cognitiva

Todo nodo de habilidad, sin importar el dominio, se clasifica en UNO de estos tipos. Esto es lo que permite que el catálogo sea abierto sin que el Orquestador necesite "saber" qué es ajedrez o alemán.

```python
class TipoCargaCognitiva(str, Enum):
    MOTORA_FINA = "motora_fina"           # piano, dibujo, escritura a mano
    MEMORIA_SEMANTICA = "memoria_semantica"  # vocabulario, hechos, nomenclatura
    LOGICO_MATEMATICO = "logico_matematico"  # álgebra, ecuaciones, demostraciones
    ESPACIAL = "espacial"                  # ajedrez, geometría, orientación
    AUDITIVO_LINGUISTICO = "auditivo_linguistico"  # comprensión oral, pronunciación
    CONCEPTUAL_INTUITIVO = "conceptual_intuitivo"  # interpretación física, "entender por qué"
    INTEGRADOR = "integrador"              # conversación libre, partida completa, ensayo
```

**Regla del Orquestador**: nunca agendar dos bloques consecutivos con el mismo valor de este enum, salvo que el usuario solo tenga un dominio activo ese día.

---

## 6. Contratos de API (CONTRATO FIJO — no modificar sin sincronizar con el otro dev)

> Esta sección se llena en el "Prompt de Sincronización #1". Hasta entonces, ambos desarrolladores trabajan con estos placeholders y NO deben asumir campos adicionales.

### 6.1 Endpoint: `POST /api/domains/decompose`
Input: `{ "meta_usuario": string, "tiempo_disponible_semanal_min": int }`
Output:
```json
{
  "dominio": "string",
  "nodos": [
    {
      "id": "string (uuid)",
      "nombre": "string",
      "tipo_carga_cognitiva": "enum (sección 5)",
      "prerequisitos": ["id1", "id2"],
      "tiempo_estimado_fijacion_min": int,
      "formato_practica_ideal": "flashcard | ejercicio_activo | simulacion | quiz | conversacion"
    }
  ]
}
```

### 6.2 Endpoint: `POST /api/orquestador/plan-diario`
Input: `{ "usuario_id": string, "fecha": "YYYY-MM-DD" }`
Output:
```json
{
  "bloques": [
    {
      "nodo_id": "string",
      "dominio": "string",
      "tipo_carga_cognitiva": "enum",
      "duracion_min": int,
      "orden": int
    }
  ],
  "nota_fatiga": "string (opcional, ej: 'Día ligero por baja energía reportada ayer')"
}
```

### 6.3 Endpoint: `POST /api/ejercicios/generar`
Input: `{ "nodo_id": string, "nivel_actual": int, "historial_reciente": [...] }`
Output:
```json
{
  "tipo_ejercicio": "flashcard | ejercicio_activo | simulacion | quiz | conversacion",
  "contenido": { /* estructura específica según tipo_ejercicio — ver Prompt 5 del backend */ },
  "respuesta_correcta": "string | object",
  "explicacion": "string"
}
```

### 6.4 Endpoint: `POST /api/feedback/registrar`
Input: `{ "usuario_id": string, "nodo_id": string, "acierto": bool, "tiempo_respuesta_seg": float, "fatiga_autoreportada": int (1-5) }`
Output: `{ "siguiente_repaso": "YYYY-MM-DD", "ajuste_dificultad": "subir | mantener | bajar" }`

---

## 7. Estructura de carpetas (monorepo)

```
/proyecto-raiz
  /frontend          → Next.js (Persona B)
    /app
    /components
    /lib
  /backend            → FastAPI (Persona A)
    /agentes
      domain_decomposer.py
      orquestador.py
      generador_ejercicios.py
      feedback_loop.py
    /models           → schemas Pydantic (= contratos de sección 6)
    /db
    main.py
  /docs
    INSTRUCCIONES.md   ← este archivo
    CADENA_PROMPTS_PERSONA_A.md
    CADENA_PROMPTS_PERSONA_B.md
  .env.example
  README.md
```

---

## 8. Reglas de seguridad y buenas prácticas (no negociables)

- Nunca hardcodear API keys. Todo vía `.env`, nunca commiteado (verificar `.gitignore` desde el primer commit).
- Toda llamada a Claude API debe tener manejo de error y timeout — si la IA falla, el usuario debe ver un mensaje claro, nunca un crash.
- Row Level Security (RLS) activado en Supabase desde el día 1 — cada usuario solo lee/escribe sus propios datos.
- Validar todo input de usuario con Pydantic (backend) y Zod (frontend) antes de tocar la IA o la DB.

---

## 9. Definición de "Done" para el MVP del hackathon

El MVP está listo cuando un usuario puede:
1. Registrarse y escribir 2-3 metas de aprendizaje en lenguaje libre.
2. Ver un plan diario generado automáticamente que intercala esas metas.
3. Completar al menos un ejercicio generado por IA por dominio.
4. Ver su progreso reflejado en un dashboard simple.
5. Volver al día siguiente y ver que el plan cambió según su desempeño.

Todo lo que no sea esto (gamificación avanzada, notificaciones push, app móvil) es Fase 2.
