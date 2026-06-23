# CADENA DE PROMPTS — PERSONA B (Frontend + Auth + UI)

> Cómo usar esto: copia y pega INSTRUCCIONES.md completo al inicio de tu
> chat con Claude Code o Cursor (pégalo, no lo describas, pégalo
> literalmente). Después ejecuta cada prompt UNO POR UNO, en el orden que
> aparecen. No saltes prompts. Después de cada uno, prueba que funcione
> en tu navegador antes de pasar al siguiente — si algo no funciona,
> dile a la IA exactamente qué error ves (copia el mensaje de error
> completo) en vez de intentar arreglarlo a mano.

---

## PROMPT 0 — Setup inicial del frontend

```
Lee INSTRUCCIONES.md completo antes de hacer nada.

Crea un proyecto Next.js 14 nuevo en /frontend usando App Router y
TypeScript y Tailwind CSS.

Después:
1. Instala y configura el cliente de Supabase (@supabase/supabase-js y
   @supabase/ssr).
2. Crea un archivo .env.local.example con placeholders para
   NEXT_PUBLIC_SUPABASE_URL y NEXT_PUBLIC_SUPABASE_ANON_KEY, y
   NEXT_PUBLIC_API_URL (esta última apuntando a http://localhost:8000,
   que es donde correrá el backend de mi compañero).
3. Crea una página de inicio simple en /app/page.tsx que solo diga
   "Plataforma de Aprendizaje Multi-Dominio" para confirmar que todo
   corre.
4. Dame el comando exacto para correr esto localmente y en qué puerto
   se verá (debería ser localhost:3000).

Explícame en español simple, paso a paso, qué hiciste y por qué, como si
yo no supiera nada de Next.js todavía.
```

---

## PROMPT 1 — Autenticación con Supabase

```
Implementa autenticación completa de usuarios usando Supabase Auth:

1. Página /app/login/page.tsx con formulario de email + contraseña, y un
   link a "crear cuenta".
2. Página /app/registro/page.tsx con formulario de registro (email,
   contraseña, nombre).
3. Lógica de sesión: si el usuario no está autenticado y trata de entrar
   a cualquier página que no sea login/registro, redirígelo
   automáticamente a /login (usa un middleware de Next.js para esto).
4. Un botón de "Cerrar sesión" visible en una barra de navegación simple
   en la parte superior, visible en todas las páginas excepto login/registro.
5. Crea la tabla de usuarios en Supabase si Supabase Auth no la crea
   automáticamente — necesito guardar al menos: nombre,
   tiempo_disponible_semanal_min (lo pediremos en el onboarding del
   Prompt 2).

IMPORTANTE: actívame Row Level Security (RLS) en Supabase para esta
tabla desde ya, explícame en 2 frases qué hace RLS y por qué es
importante, en términos simples.

Después de implementar esto, dime exactamente qué pasos debo hacer YO en
el panel de Supabase (la página web de supabase.com) para que esto
funcione, paso a paso, como si nunca hubiera entrado a Supabase antes.
```

---

## 🔄 PROMPT DE SINCRONIZACIÓN #1 (ejecutar junto con Persona A)

```
Necesito definir el schema completo de base de datos en Supabase, porque
mi compañero (Persona A) va a construir tablas que dependen de la tabla
de usuarios que yo ya creé.

Aquí está mi tabla de usuarios actual: [PEGAR AQUÍ el SQL de la tabla
de usuarios que se generó en el Prompt 1].

Genera el archivo /backend/db/schema.sql completo (aunque esta carpeta
la maneja mi compañero, yo necesito entender qué tablas van a existir)
con TODAS las tablas necesarias para todo el proyecto, basándote en los
4 contratos de API de la sección 6 de INSTRUCCIONES.md:

- Tabla de usuarios (la mía, no la dupliques, solo referénciala)
- Tabla de dominios/metas del usuario
- Tabla de nodos de habilidad (árboles del Domain Decomposer)
- Tabla de estado de repetición espaciada por nodo
- Tabla de historial de ejercicios completados

Explícame en español simple qué representa cada tabla y cómo se conectan
entre sí (qué es una foreign key, en una frase, por si no lo recuerdo).

Después, compárteme este archivo con mi compañero para que él lo revise
desde su lado.
```

---

## PROMPT 2 — Onboarding: definir metas de aprendizaje

```
Crea la página /app/onboarding/page.tsx. Esta es la primera pantalla que
ve un usuario nuevo después de registrarse.

Debe:
1. Preguntar: "¿Qué quieres aprender?" con un campo de texto libre donde
   el usuario puede escribir CUALQUIER cosa (ej. "aprender alemán",
   "tocar piano", "entender las ecuaciones de Maxwell"). Debe permitir
   agregar de 1 a 5 metas (un botón "+ agregar otra meta").
2. Preguntar: "¿Cuánto tiempo tienes disponible por semana para
   aprender?" con un selector simple (ej. 2-3 horas, 4-6 horas, 7-10
   horas, más de 10 horas) — esto se traduce internamente a minutos para
   guardarlo en la base de datos.
3. Al confirmar, llamar al endpoint POST /api/domains/decompose de mi
   compañero (una vez por cada meta que el usuario escribió) y mostrar
   un loading state amigable mientras la IA procesa ("Estamos diseñando
   tu plan de aprendizaje..." con un spinner), porque esto puede tardar
   unos segundos por cada meta.
4. Guardar el resultado (los árboles de nodos que devuelve el backend)
   en Supabase asociado al usuario.
5. Redirigir al dashboard principal (/app/dashboard) al terminar.

Usa el contrato exacto de la sección 6.1 de INSTRUCCIONES.md para saber
qué forma tiene la respuesta del backend — NO inventes campos que no
estén ahí.

Si mi compañero todavía no tiene el backend listo, créame datos de
prueba (mock) con la misma estructura exacta, para que yo pueda seguir
construyendo el frontend sin esperarlo, y déjame un comentario claro en
el código diciendo "ESTO ES MOCK, REEMPLAZAR CUANDO EL BACKEND ESTÉ
LISTO".
```

---

## PROMPT 3 — Dashboard principal: el plan del día

```
Crea /app/dashboard/page.tsx, la pantalla más importante de toda la app
— es lo que el usuario ve cada día.

Debe:
1. Al cargar, llamar a POST /api/orquestador/plan-diario (contrato 6.2)
   para obtener los bloques de práctica del día.
2. Mostrar los bloques como una lista visual de "tarjetas", una por
   bloque, en el ORDEN que indica el campo "orden" de la respuesta. Cada
   tarjeta muestra: el nombre del dominio, un ícono o color distinto
   según tipo_carga_cognitiva (esto ayuda al usuario a "sentir" la
   variedad sin tener que leer la etiqueta técnica), y la duración
   estimada.
3. Si la respuesta incluye "nota_fatiga", mostrar un mensaje amigable
   arriba de las tarjetas (ej. "Hoy te preparamos un día más ligero 🌱").
4. Cada tarjeta debe ser clickeable y llevar a /app/practica/[nodo_id]
   (la construiremos en el Prompt 4).
5. Diseño visual: sigue una estética minimalista tipo nórdico/editorial
   — fondo claro, tipografía limpia, espaciado generoso, paleta de
   colores reducida (2-3 colores principales + neutros). Evita el look
   de "app gamificada infantil" (sin emojis excesivos, sin colores
   neón) — el usuario que queremos atraer es alguien serio sobre su
   desarrollo personal, no un niño.

No te preocupes por animaciones complejas todavía, prioriza que
funcione y se vea limpio.
```

---

## PROMPT 4 — Pantalla de práctica (el ejercicio en sí)

```
Crea /app/practica/[nodo_id]/page.tsx — la pantalla donde el usuario
realmente hace el ejercicio.

Debe:
1. Al cargar, llamar a POST /api/ejercicios/generar (contrato 6.3) para
   obtener el ejercicio concreto para ese nodo.
2. Renderizar el ejercicio de forma DIFERENTE según tipo_ejercicio:
   - flashcard: mostrar la pregunta, un botón "Mostrar respuesta", y
     después dos botones "Lo sabía" / "No lo sabía".
   - quiz: mostrar la pregunta con 4 botones de opción, resaltar en
     verde/rojo según si acertó al hacer click.
   - ejercicio_activo y simulacion: mostrar el planteamiento, un campo
     de texto libre para que el usuario escriba su respuesta o
     razonamiento, y un botón "Verificar" que compara con
     respuesta_correcta y muestra la explicación.
   - conversacion: mostrar el prompt, un campo de texto libre amplio
     tipo chat, sin "respuesta correcta" rígida — solo mostrar la
     explicación/feedback general después de que el usuario responda.
3. Al terminar (sea que acertó o no), capturar: si acertó (bool), cuánto
   tiempo tardó en responder (mide esto desde que se mostró el ejercicio
   hasta que el usuario confirmó su respuesta), y preguntarle "¿qué tan
   cansado te sientes ahora?" con 5 botones simples (1 a 5).
4. Enviar todo esto a POST /api/feedback/registrar (contrato 6.4).
5. Después de enviar el feedback, mostrar un mensaje breve de
   confirmación y un botón "Volver al plan del día" que regresa a
   /app/dashboard.

Mantén el mismo estilo visual minimalista del Prompt 3.
```

---

## PROMPT 5 — Perfil de progreso / "Panel de talento"

```
Crea /app/perfil/page.tsx, un dashboard de progreso histórico (distinto
al dashboard diario del Prompt 3).

Debe mostrar:
1. Una vista resumen con los dominios activos del usuario y, para cada
   uno, una barra o indicador simple de progreso (% de nodos del árbol
   ya dominados vs total).
2. Una racha de días consecutivos usando la app (esto se puede calcular
   en el frontend a partir del historial, o pedirle a mi compañero que
   lo agregue al backend si es más fácil para él — pregúntame antes de
   decidir cuál camino tomar).
3. Un gráfico simple de actividad por tipo_carga_cognitiva en los
   últimos 7 días (puede ser un gráfico de barras horizontal sencillo,
   no necesita ser sofisticado) — esto refuerza visualmente la idea de
   "estás desarrollando varios tipos de habilidad, no solo memorizando".

Usa una librería simple de gráficos (recharts está bien) y mantén el
mismo estilo visual minimalista de las pantallas anteriores.

Esta pantalla es importante para la demo del hackathon porque es donde
"se ve" la propuesta de valor del producto, así que dedica cuidado
extra a que se vea profesional y claro de un vistazo.
```

---

## PROMPT 6 — Pulido visual y responsive

```
Revisa TODAS las pantallas que hemos construido (login, registro,
onboarding, dashboard, práctica, perfil) y:

1. Confirma que se ven bien en pantallas de celular (responsive) — la
   mayoría de la gente probará esto desde el teléfono durante la demo.
2. Añade estados de carga (loading spinners o skeletons) en TODOS los
   lugares donde se espera respuesta de la IA o de la base de datos —
   nunca debe verse una pantalla en blanco mientras carga.
3. Añade manejo de errores visible: si una llamada al backend falla,
   muestra un mensaje amigable ("Algo salió mal, intenta de nuevo") en
   vez de que la página se rompa o quede en blanco.
4. Revisa que la navegación entre páginas sea consistente (misma barra
   superior, mismo estilo de botones en todas las pantallas).

Dame una lista de cualquier pantalla que NO hayas podido terminar de
pulir, para yo saber qué priorizar antes de la demo.
```

---

## PROMPT 7 — Preparación para deploy

```
Prepara el frontend para desplegarlo en Vercel:

1. Confirma que todas las variables de entorno (.env.local) están
   correctamente referenciadas y no hay ninguna hardcodeada en el
   código.
2. Verifica que NEXT_PUBLIC_API_URL se pueda cambiar fácilmente entre
   localhost (desarrollo) y la URL real del backend desplegado
   (producción) sin tocar código.
3. Corre un build de producción localmente (dame el comando) y
   solucióname cualquier error o warning que aparezca.
4. Dame instrucciones paso a paso, como si nunca hubiera usado Vercel,
   de cómo conectar este repo de GitHub a Vercel y hacer el primer
   deploy.
```
