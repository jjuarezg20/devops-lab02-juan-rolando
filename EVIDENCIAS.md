# Evidencias

## Laboratorio 2. Integración continua

Repositorio: https://github.com/jjuarezg20/devops-lab02-juan-rolando
Pull request de integración: https://github.com/jjuarezg20/devops-lab02-juan-rolando/pull/1
Workflow: [`.github/workflows/ci.yml`](.github/workflows/ci.yml)

### 1. Pipeline final

El workflow `CI` se ejecuta únicamente ante `push` a `main` y `pull_request` con destino `main`. Declara `permissions: contents: read` (solo lectura del código; no usa secretos, tokens ni permisos de escritura).

| Job | Responsabilidad | Depende de |
|-----|-----------------|------------|
| **Lint** | Instala dependencias (con caché de pip) y ejecuta `python -m ruff check .` | — |
| **Test** | Instala dependencias (con caché de pip), ejecuta la suite con `pytest`, mide cobertura sobre `app` y genera `coverage.xml`, que se publica como workflow artifact | — |
| **Container** | Construye el `Dockerfile` suministrado y etiqueta la imagen como `incident-api:<sha-del-commit>` (no se publica en ningún registry) | Lint, Test |

- **Lint** y **Test** no declaran dependencias entre sí, por lo que se ejecutan en paralelo.
- **Container** declara `needs: [lint, test]`: solo se ejecuta si ambos finalizan satisfactoriamente; si alguno falla, queda como *skipped*.
- En eventos `pull_request` el tag de la imagen usa `github.event.pull_request.head.sha` (commit de la rama evaluada); en `push` usa `github.sha`.

Ruleset `Proteger main` (activo, rama `main`, sin bypass): exige pull request y que los status checks **Lint**, **Test** y **Container** finalicen satisfactoriamente antes del merge.

### 2. Fallo de lint

- Ejecución: https://github.com/jjuarezg20/devops-lab02-juan-rolando/actions/runs/36167395151
- Commit: `9b4e4f7179194022d0e5c5a3430374c85e7bb3c2`
- Cambio que provocó el fallo: se agregó el archivo temporal `app/tmp_lint_failure.py` con `import os` sin utilizar. Ruff reportó `F401 [*] \ imported but unused`.
- Resultado: **Lint ❌ · Test ✅ · Container ⏭️ skipped** (no se ejecutó por `needs`).
- Merge bloqueado: el PR #1 quedó en estado `BLOCKED` (`gh pr view 1 --json mergeStateStatus`) porque el check requerido Lint falló.
- Corrección: el archivo temporal se eliminó en el commit `fix: elimina violación temporal de Ruff` del mismo PR.

### 3. Fallo de pruebas

- Ejecución: https://github.com/jjuarezg20/devops-lab02-juan-rolando/actions/runs/36167575935
- Commit: `78a8e7f77171b484484e3f3384f7f6497e780257`
- Cambio que provocó el fallo: se agregó el archivo temporal `tests/test_tmp_failure.py` con `assert 1 == 2`. Resultado de pytest: `FAILED tests/test_tmp_failure.py::test_fallo_deliberado - assert 1 == 2` (1 failed, 9 passed).
- Resultado: **Lint ✅ · Test ❌ · Container ⏭️ skipped**.
- Merge bloqueado: el PR #1 quedó nuevamente en estado `BLOCKED`.
- Corrección: la prueba temporal se eliminó en el commit `fix: elimina prueba temporal fallida`. Las pruebas suministradas no se modificaron.

### 4. Ejecución satisfactoria

- Ejecución con los tres jobs satisfactorios: https://github.com/jjuarezg20/devops-lab02-juan-rolando/actions/runs/36167851554
- Resultado: **Lint ✅ · Test ✅ · Container ✅**; el PR pasó a estado `CLEAN` (merge habilitado por el ruleset).
- Pull request: https://github.com/jjuarezg20/devops-lab02-juan-rolando/pull/1
- SHA del commit final evaluado: `825f5d7381fa2b4da7c44c9aa1a7bcfbde85a17e`
- Imagen construida: `incident-api:825f5d7381fa2b4da7c44c9aa1a7bcfbde85a17e`

### 5. Artifact

- Nombre: `coverage-report-825f5d7381fa2b4da7c44c9aa1a7bcfbde85a17e` (contiene `coverage.xml`)
- ID: `10878735273`
- Generado por el job **Test** de la ejecución https://github.com/jjuarezg20/devops-lab02-juan-rolando/actions/runs/36167851554 (sección *Artifacts* del resumen de la ejecución). Retención: 14 días.

### 6. Caché

`actions/setup-python` usa `cache: pip` con `cache-dependency-path` apuntando a `requirements.txt` y `requirements-dev.txt`. La clave de la caché incluye un hash de esos archivos.

- Preparación de la caché (primera ejecución): https://github.com/jjuarezg20/devops-lab02-juan-rolando/actions/runs/36165805463 → `Cache saved with the key: setup-python-Linux-x64-24.04-Ubuntu-python-3.13.15-pip-f2ffb87e…`
- Reutilización (ejecución posterior): https://github.com/jjuarezg20/devops-lab02-juan-rolando/actions/runs/36167851554 → en Lint y Test, paso *Set up Python*: `Cache restored from key: setup-python-Linux-x64-24.04-Ubuntu-python-3.13.15-pip-f2ffb87e…` y en *Post Set up Python*: `Cache hit occurred on the primary key …, not saving cache.`
- Qué provocaría su actualización: cualquier cambio en `requirements.txt` o `requirements-dev.txt` (por ejemplo, subir la versión de `ruff` o `fastapi`) cambia el hash de la clave, produce un *cache miss* y se guarda una nueva caché. También cambiaría si cambia la versión de Python o la imagen del runner, que forman parte de la clave.
