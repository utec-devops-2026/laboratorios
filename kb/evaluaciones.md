# KB: Evaluaciones del curso (Google Forms + draw.io)

Conocimiento reutilizable para armar exámenes y entregables. **No guardar aquí preguntas ni claves**: el material de evaluación vive en `examenes/`, que está en `.gitignore`.

## Alcance por semana

El sílabo (IS4024, 2026-2) no asigna temas a semanas explícitamente. Para el **parcial (semana 8)** se toman los temas 6.1 a 6.7:

| Tema | Contenido | Lab asociado |
|---|---|---|
| 6.1 | Fundamentos DevOps, DORA | — |
| 6.2 | Git, GitFlow vs TBD, Conventional Commits, Linux/Bash | `laboratorio_git_github/`, `terminal/` |
| 6.3 | Asistentes de IA, pruebas unitarias, peer review, MCP | `laboratorio_AI_assistants/` |
| 6.4 | AWS (IAM, S3, EC2, CLI) | `laboratorio_aws/` |
| 6.5 | Docker y Compose | `laboratorio_docker/`, `laboratorio_docker_compose/` (00→01→02) |
| 6.6 | Kubernetes | `laboratorio_kubernetes/laboratorio_kubernetes.md` (versión reducida) |
| 6.7 | GitHub Actions | `laboratorio_github_actions/` |

Secret scanning y GHAS se enseñaron antes del parcial, aunque DevSecOps figura como 6.10 en el sílabo.

**El sílabo no basta: contrastar con lo que realmente se dictó.** Ejemplos del parcial 2026-2:
- Kubernetes: el lab reducido cubre Deployment, ReplicaSet, Pods, Service, escalado y autorreparación. **No** cubre ConfigMap, Secret ni NodePort en profundidad, aunque el sílabo (6.6.2) liste ConfigMaps y Secrets.
- GitHub Actions: Artefactos, environments con aprobación, `if: always()` y el gate de cobertura están en el lab, pero se dejaron fuera del parcial.

## Google Forms como examen

| Tema | Regla |
|---|---|
| Puntos | **Solo enteros.** Si la nota es /20 y hay muchas preguntas, cada opción múltiple vale 1 y lo que sobra va a las preguntas manuales. |
| Modo cuestionario | Configuración → *Convertir en cuestionario*. Clave por pregunta: *Clave de respuestas* → opción correcta → puntos. |
| Tipo de pregunta | *Opción múltiple* (círculo), no *Casillas*, para que la corrección automática sea todo o nada. |
| Publicar notas | *Más tarde, después de la revisión manual* cuando hay preguntas de carga de archivos. |
| Carga de archivos | Exige inicio de sesión con Google y guarda en el Drive del docente. `.drawio` no está en la lista de tipos → desactivar *solo tipos específicos*. |
| Rúbricas | Forms no tiene rúbricas nativas: pegar la rúbrica en la descripción de la pregunta y calificar en *Respuestas → Individual*. Con 2 puntos por pregunta, usar checklist ("cumple N de M criterios → 2/1/0"). |
| Código | Forms no formatea código → adjuntar el snippet como imagen (ícono de imagen junto al título). |
| Anticopia | *Orden aleatorio de opciones* por pregunta y *Orden aleatorio de las preguntas* en Presentación. |
| Total de puntos | Revisar que el encabezado diga el total esperado. Una pregunta de datos (p. ej. "Nombres y Apellidos") recibe los puntos predeterminados: ponerla en 0. |
| Validar el Form | Copiar el texto del Form en modo edición no muestra ni la clave marcada ni las imágenes: revisarlas en el Form mismo o respondiendo en vista previa. |
| Cambiar la clave | Si mueves de lugar la opción correcta, vuelve a marcarla en la *Clave de respuestas*: la marca no se traslada sola. |

## Diagramas en draw.io

- La app está instalada en `/Applications/draw.io.app`. Exportar a PNG desde CLI para revisar el diagrama:
  ```bash
  /Applications/draw.io.app/Contents/MacOS/draw.io -x -f png -s 1 -o salida.png archivo.drawio
  ```
- Las plantillas se escriben como XML sin comprimir (`<mxfile><diagram><mxGraphModel>…`). Con `html=1`, los saltos de línea van como `&lt;br&gt;` y **no se usa `<texto>` literal** dentro de `value` (se interpreta como etiqueta HTML); usar `{sha}` en vez de `<sha>`.
- Patrón que funciona para los alumnos: dar un **ejemplo base hecho con el lab** (Flask + PostgreSQL) y pedir que lo **adapten a un caso distinto** (p. ej. una API Node.js/npm con Redis). Así saben qué presentar y no pueden entregar el ejemplo tal cual.
- Convención de pistas en las plantillas: **borde punteado** = caja que el alumno debe editar, **texto rojo** = dato que cambia, **💡 texto rojo en cursiva** = pista que explica el porqué (p. ej. "Flask escuchaba en 5000; Express en 3000 → cambia el puerto"). Las pistas guían sin dar la respuesta: se formulan como pregunta. Para un componente nuevo (p. ej. Redis), dejar una caja punteada vacía con la flecha ya conectada.
- Carriles recomendados para el flujo DevOps: Git (TBD) → GitHub Actions (jobs con `needs`) → Registry → Kubernetes. Para Compose: redes `public`/`private`, puertos publicados, volumen nombrado y secreto.

## Imágenes de código para preguntas

- Generar con HTML + Chrome headless y recortar con PIL (ver `examenes/imagenes/src/build.py`):
  ```bash
  "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new \
    --force-device-scale-factor=2 --window-size=1150,2000 --screenshot=out.png file:///ruta/pagina.html
  ```
- Estilo: ventana oscura tipo editor con el nombre del archivo en la barra (Menlo 21px).
- Si la pregunta pide predecir un resultado, **no mostrar la salida** en la imagen (usar `???` o `...`).
- Marcar como "ejemplo ilustrativo" cualquier dato inventado (CVEs, versiones vulnerables).
