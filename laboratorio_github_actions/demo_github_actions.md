# Guía: GitHub Actions — Trigger Filtrado por Evento

**Archivo:** `my-event-workflow.yaml`

---

## Paso 0 — Crear el archivo del workflow

GitHub Actions solo reconoce workflows dentro de la carpeta **`.github/workflows/`** del repositorio.

```bash
mkdir -p .github/workflows
touch .github/workflows/my-event-workflow.yaml
```

Estructura esperada:

```
repo/
└── .github/
    └── workflows/
        └── my-event-workflow.yaml   ← aquí va el YAML
```

> **Importante:** La carpeta `.github/workflows` debe estar en la raíz del repositorio. Si el archivo está en otra ubicación, GitHub **no lo detecta**.

---

## Paso 1 — Trigger: `pull_request` a `main` con tipo `opened`

```yaml
name: My Event Workflow
on:
  pull_request:
    types:
      - opened
    branches:
      - 'main'
```

Los dos filtros aplican en **AND** — el workflow solo corre si se cumplen los dos a la vez.

| Filtro | Valor | Efecto |
|--------|-------|--------|
| `branches: main` | PR apunta a `main` | Ignora PRs a otras ramas |
| `types: opened` | PR recién abierto | Ignora sync/reopen/close |

> **SÍ corre:** abrir un PR desde `feature/update-workflow` → `main`  
> **NO corre:** hacer push adicional a una PR ya abierta (tipo `synchronize`)

---

## Paso 2 — Job `model-ci`

```yaml
jobs:
  model-ci:
    runs-on: ubuntu-latest
    steps:
      - name: Build Model
        run: echo "Build Model in CI Pipeline"
```

> **Print esperado:**
> ```
> ▶ model-ci
>   ✓ Set up job    2s
>   ✓ Build Model   1s
>     Build Model in CI Pipeline
>   ✓ Complete job  0s
> ```

---

## Paso 3 — Job `model-cd`

```yaml
  model-cd:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy Model
        run: echo "Deploy Model to AWS in CD Pipeline"
```

`model-cd` **no tiene `needs`** — corre en paralelo con `model-ci`.

> **Print esperado:**
> ```
> ▶ model-cd
>   ✓ Set up job     2s
>   ✓ Deploy Model   1s
>     Deploy Model to AWS in CD Pipeline
>   ✓ Complete job   0s
> ```

---

## Paso 4 — Vista resumen en Actions

> ```
> My Event Workflow  #1  ✓
>
> Jobs:
>   model-ci  ✓  3s
>   model-cd  ✓  3s
> ```

---

## Workflow completo

```yaml
name: My Event Workflow
on:
  pull_request:
    types:
      - opened
    branches:
      - 'main'
jobs:
  model-ci:
    runs-on: ubuntu-latest
    steps:
      - name: Build Model
        run: echo "Build Model in CI Pipeline"

  model-cd:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy Model
        run: echo "Deploy Model to AWS in CD Pipeline"
```

---

## Checklist

- [ ] Workflow aparece en la pestaña Actions de GitHub
- [ ] Se dispara al abrir un PR desde `feature/*` → `main`
- [ ] No se dispara al hacer push adicional a una PR ya abierta
- [ ] No se dispara si el PR apunta a una rama distinta de `main`
- [ ] `model-ci` y `model-cd` corren en paralelo
- [ ] Print `Build Model in CI Pipeline` visible en logs de `model-ci`
- [ ] Print `Deploy Model to AWS in CD Pipeline` visible en logs de `model-cd`

---

**Autor:** Wilson Julca Mejía  
Curso: *DevOps y GitHub Actions — MLOps con Python*  
Universidad de Ingeniería y Tecnología (UTEC)