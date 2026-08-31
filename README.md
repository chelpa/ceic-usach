# Bitácora de obra — CEIC USACH

Plan técnico para terminar [ceicusach.cl](https://ceicusach.cl), sección por sección. Generado a partir de una revisión en vivo del sitio y del código de BuscaCursos (31 de agosto de 2026).

Marca las casillas con un commit a medida que cierres tareas — así el progreso queda en el historial de git, no solo en la cabeza de quien lo hizo. Hay también una [versión interactiva](https://claude.ai/code/artifact/45480f3e-cc3b-4f17-8f9b-8495a48181a7) con checklist que se guarda solo en el navegador, si prefieres esa para uso personal.

---

## Antes que nada: el sitio 404 en cualquier ruta que no sea "/"

> [!CAUTION]
> Esto no es contenido sin construir — es una configuración de hosting rota, y bloquea todo lo demás.

Confirmado de dos formas independientes: pedir `https://ceicusach.cl/wikiprofes` directo devuelve **HTTP 404 del servidor** (no un 404 dibujado por React), y al abrirlo en un navegador la página queda en blanco con errores 404 en consola. El sitio es una SPA (React + Vite, ruteo en el cliente) — funciona si navegas haciendo clic desde "/", pero **cualquier acceso directo** a una ruta interna (recargar, un link compartido, un resultado de Google, un marcador) muere en un 404 real. Nadie puede compartir un link a WikiProfes, nadie puede recargar sin perder la página, y Google no puede indexar nada más que el Inicio.

**El arreglo:** decirle al hosting que sirva `index.html` para cualquier ruta que no sea un archivo estático. Primero identifica el proveedor — DevTools → Network → recarga cualquier ruta → mira los headers de la respuesta (`x-vercel-id`, `x-nf-request-id`, `cf-ray` te dicen cuál es) — y aplica el snippet que corresponda:

<details>
<summary><strong>Vercel</strong> — crear <code>vercel.json</code> en la raíz</summary>

```json
{
  "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }]
}
```
</details>

<details>
<summary><strong>Netlify</strong> — crear <code>public/_redirects</code></summary>

```
/*  /index.html  200
```
</details>

<details>
<summary><strong>Cloudflare Pages</strong> — mismo archivo que Netlify</summary>

```
/*  /index.html  200
```
</details>

<details>
<summary><strong>Nginx (VPS propio)</strong> — dentro del bloque <code>server {}</code></summary>

```nginx
location / {
  try_files $uri $uri/ /index.html;
}
```
</details>

**Cómo verificar:** recarga `ceicusach.cl/wikiprofes` directo en una pestaña nueva. Si carga el contenido en vez de quedar en blanco, está arreglado.

- [ ] Identificar el proveedor de hosting actual
- [ ] Aplicar el fallback de SPA correspondiente
- [ ] Verificar recargando una ruta interna directo

---

## Antes de construir: dos decisiones de arquitectura

La base (React + Vite + Tailwind, ruteo en cliente) está bien elegida — sigan con eso.

**1. Un solo componente para el contenido de tipo lista.** Noticias, Actas, Transparencia, Documentación y FAQ son, en el fondo, la misma estructura: una lista de entradas con fecha, título y a veces un PDF adjunto. Constrúyelo una sola vez como componente "lista de documentos" leyendo de un manifiesto de datos (JSON o carpeta Markdown), y las cinco secciones se vuelven configuración, no código nuevo. Importa además porque quien actualice Noticias el próximo semestre probablemente no sea quien programó el sitio.

**2. Ya existe un dataset de ramos — reúsenlo.** El HTML de BuscaCursos trae un arreglo `SECTIONS` embebido con **226 secciones → 54 ramos únicos**, mención Administración (códigos 352xxx), horario 2026-2, con nivel, créditos SCT y área por ramo. No hay que "definir" un catálogo nuevo para Malla Interactiva — hay que deduplicar por código y reusarlo, en vez de escribir los mismos ramos dos veces en dos proyectos distintos.

---

## El plan, en orden de prioridad

### 1. Malla Interactiva — `/malla`

![estado](https://img.shields.io/badge/estado-ambas_menciones_confirmadas-a8761f)

Malla curricular de Ingeniería Comercial navegable — **mención Administración y mención Economía**. Confirmado: no existe ningún diagrama de prerrequisitos ya hecho, así que eso se construye desde cero; los datos de ramos, no.

**Qué existe hoy:** nada en `/malla` todavía, pero hay dos fuentes oficiales identificadas: [fae.usach.cl/cica](https://fae.usach.cl/cica/) (mención Administración) y [fae.usach.cl/cice](https://fae.usach.cl/cice/) (mención Economía) — cada una lista los ramos por semestre (10 semestres) y enlaza al PDF de programa de cada ramo, más un PDF completo descargable por mención. Revisé el PDF completo de Administración: es una tabla por semestre, **sin diagrama de prerrequisitos**. BuscaCursos ya trae los 54 ramos de Administración con nivel y SCT reales del horario 2026-2.

**Enfoque técnico:**
1. Deduplicar `SECTIONS` de BuscaCursos por código → catálogo base de Administración; armar el equivalente para Economía desde fae.usach.cl/cice (esa mención no está en BuscaCursos todavía).
2. Los prerrequisitos no están en ninguna fuente consolidada — solo salen de cada PDF de programa de asignatura individual (campo "requisitos"), ~100 PDFs entre ambas menciones. Es scripteable, no manual uno por uno, pero es trabajo real que hay que presupuestar.
3. Grilla por semestre y mención, resalta prerrequisitos al hacer clic — sin librería de grafos para la v1.

> [!IMPORTANT]
> Los códigos de ramo exactos que cito arriba (fae.usach.cl/cica y /cice) los saqué con una herramienta que **resume el HTML con un modelo chico**, no con un scrape literal — sirven para diagnosticar y planificar, pero hay riesgo real de que un dígito esté transcrito mal. **No escribas código contra estos números todavía**: hay que volver a sacarlos con un scrape real antes de construir el dataset de la malla.

- [ ] Deduplicar `SECTIONS` de BuscaCursos → catálogo base mención Administración
- [ ] Armar catálogo de ramos mención Economía desde fae.usach.cl/cice
- [ ] Verificar códigos de ramo con un scrape real (no el resumen de la IA)
- [ ] Script para extraer "requisitos" de los ~100 PDF de programa
- [ ] Grilla por semestre y mención + resaltado de prerrequisitos

### 2. Noticias · Actas · Transparencia · Documentación

![estado](https://img.shields.io/badge/estado-sin_construir-6b6f7a)

Cuatro secciones, un solo componente: listas de entradas fechadas, algunas con PDF adjunto. Comunican que la mesa es activa y transparente — pero una página vacía comunica lo contrario, así que no conviene publicarlas sin contenido real.

**Qué existe hoy:** solo links en el footer, sin contenido visible.

**Enfoque técnico:** un componente "lista de documentos" (fecha, título, resumen, PDF opcional) alimentado por Markdown o un manifiesto JSON. Noticias = feed corto; Actas = minuta + PDF; Transparencia = tabla presupuesto vs. gasto + PDF por periodo; Documentación = estatutos y reglamentos, mayormente estáticos.

**Necesito de la mesa:** PDFs de actas existentes, cifras/reportes de transparencia, estatutos vigentes, y al menos 3–5 noticias para no lanzar la sección vacía. También definir si Actas/Transparencia son públicas o solo para socios (cambia si se necesita login).

- [ ] Definir si Actas/Transparencia requieren acceso restringido
- [ ] Reunir PDFs de actas y cifras de transparencia existentes
- [ ] Construir el componente "lista de documentos" reutilizable
- [ ] Redactar 3–5 noticias para el lanzamiento

### 3. WikiEmpresas · Convenios

![estado](https://img.shields.io/badge/estado-sin_construir-6b6f7a)

El mismo patrón de WikiProfes (ficha + buscador), aplicado a empresas: convenios de práctica, descuentos y beneficios.

**Qué existe hoy:** solo links en el footer.

**Enfoque técnico:** WikiProfes ya resolvió ficha + buscador + 202 registros. Reutiliza ese componente de lista/ficha y cambia el modelo de datos (empresa, rubro, tipo de convenio, beneficio, contacto) en vez de reescribir la UI desde cero.

**Necesito de la mesa:** el listado de convenios/empresas que ya se maneja (probablemente existe en una planilla interna).

- [ ] Levantar el listado actual de convenios/empresas
- [ ] Adaptar el componente de WikiProfes al modelo de empresas

### 4. Calendario · FAQ

![estado](https://img.shields.io/badge/estado-sin_construir-6b6f7a)

Las dos páginas más rápidas de cerrar — buena ganancia rápida entre tareas más grandes.

**Enfoque técnico:** Calendario, para la v1, incrusta el Google Calendar interno si ya existe uno (se actualiza solo). FAQ: acordeón simple, sin backend.

- [ ] Recopilar preguntas frecuentes reales
- [ ] Definir fuente del calendario (¿Google Calendar existente?)

### 5. WikiProfes — pulido

![estado](https://img.shields.io/badge/estado-construido-3f7554)

Ya existe y funciona — el Inicio la describe con 202 fichas de profesores. Se prioriza igual, tratada como pasada de pulido, no como construcción nueva.

**Qué revisar:** rendimiento del buscador con 202+ fichas en móvil, moderación de reseñas nuevas, y si falta un formulario de "sugerir profesor o corrección".

- [ ] Reunir feedback concreto de qué falla o falta hoy
- [ ] Revisar moderación de reseñas

---

## Inventario abierto

El bug de ruteo impidió cargar el contenido real de varias rutas, y no se pudieron abrir los desplegables del menú. Vale la pena un chequeo manual de 10 minutos:

| Ítem del menú | Dónde aparece | Estado |
|---|---|---|
| Inicio | Nav principal | ✅ Construido |
| WikiProfes | Nav principal + tile Inicio | ✅ Construido |
| Apuntes | Nav principal | ❓ Sin verificar |
| Comunidad ▾ | Nav principal (desplegable) | ❓ No se pudo abrir el menú |
| El CEIC ▾ | Nav principal (desplegable) | ❓ No se pudo abrir el menú |
| Nosotros | Tile Inicio + footer | 🟡 Descrito en Inicio, contenido sin confirmar |
| Programa | Tile Inicio + footer | 🟡 Descrito en Inicio, contenido sin confirmar |
| WikiEmpresas, Malla, Noticias, Convenios, FAQ, Calendario, Actas, Transparencia, Documentación | Solo footer | ⬜ Sin evidencia de contenido |

---

## Cómo seguimos

1. **Acceso al repo** — sin esto se puede seguir planificando, pero no escribir un commit.
2. **Arreglar el 404 primero** — 15 minutos de trabajo, desbloquea todo lo demás.
3. **Componente compartido de "lista de documentos"** — rinde 4 páginas por el precio de una.
4. **Seguir el orden de arriba** — Malla → contenidos institucionales → WikiEmpresas → Calendario/FAQ → pulido de WikiProfes.
5. **En paralelo:** cada sección tiene un "necesito de la mesa" — son piezas que solo el CEIC tiene (actas, cifras, convenios, malla oficial). Conviene ir juntándolas mientras se programa.
