# AGENTS.md

## Proyecto

Aplicación para el análisis instantáneo y diferido de una estructura de puente formada por vigas prefabricadas pretensadas y una losa de hormigón ejecutada in situ.

La aplicación tendrá una interfaz web local basada en HTML, CSS y TypeScript, y un backend Python. El punto de partida es un programa existente de cálculo seccional que ya resuelve el cálculo instantáneo, pero todavía no incorpora el pretensado.

La referencia técnica inicial es el documento `EFECTOS_ESTRUCTURALES_DIFERIDAS_HORMIGON_GEHO_1998.pdf`. Debe utilizarse como fuente de formulación y validación conceptual, no como norma vigente. Las ecuaciones extraídas por OCR deben comprobarse visualmente en el PDF antes de implantarlas.

## Objetivo de la primera versión

Construir una aplicación local, verificable y modular que resuelva una sección representativa de una viga simplemente apoyada con sección evolutiva:

1. Viga prefabricada sola.
2. Transferencia del pretensado.
3. Peso propio de la viga.
4. Hormigonado de la losa, cuyo peso actúa sobre la viga sola.
5. Activación resistente de la losa.
6. Aplicación de cargas muertas sobre la sección compuesta.
7. Evolución de fluencia, retracción y relajación.

La primera versión se limita a comportamiento longitudinal, sección no fisurada, adherencia perfecta entre viga y losa, tendones adherentes o compatibles con la sección, y estructura isostática. No incluir inicialmente cortante, torsión, reparto transversal, fisuración, conectores flexibles, continuidad entre vanos ni análisis global de emparrillado.

## Decisiones tecnológicas

### Frontend

- HTML semántico.
- CSS propio, sobrio y técnico.
- TypeScript, no JavaScript sin tipos.
- Vite como entorno de desarrollo y construcción.
- Plotly para gráficos, salvo decisión posterior documentada.

### Backend

- Python 3.12 o superior.
- FastAPI para la API HTTP local.
- Pydantic para contratos de entrada y salida.
- NumPy para operaciones matriciales.
- SciPy solo cuando resulte necesario.
- pytest para ensayos automáticos.

### Ejecución inicial

La aplicación funcionará localmente:

- Backend en `localhost`.
- Frontend servido localmente y abierto en Edge o Chrome.
- Sin dependencia inicial de Azure, SharePoint ni servicios externos.
- Preparada para un despliegue posterior en un servidor interno.

## Principios obligatorios

1. No reescribir el motor seccional existente sin auditarlo primero.
2. Separar completamente cálculo, API e interfaz.
3. El motor de cálculo no puede importar FastAPI ni depender del DOM.
4. Usar internamente N, mm, MPa y días.
5. Toda conversión de unidades se realiza en los límites de entrada o salida.
6. Documentar explícitamente ejes, signos, fibras de referencia y excentricidades.
7. Cada ecuación implantada debe indicar fuente, hipótesis y unidades.
8. No copiar fórmulas del OCR sin revisión visual del PDF.
9. Toda modificación del motor debe incluir pruebas.
10. No ocultar desequilibrios numéricos mediante redondeos o tolerancias excesivas.
11. No inventar datos, fórmulas ni resultados ausentes en las fuentes.
12. Si existe una duda técnica, registrarla en `docs/OPEN_QUESTIONS.md` antes de asumir una solución.

## Arquitectura prevista

```text
bridge-composite-app/
├── AGENTS.md
├── README.md
├── pyproject.toml
├── package.json
├── docs/
│   ├── PROJECT_SPEC.md
│   ├── UNITS_AND_SIGNS.md
│   ├── EQUATIONS_REGISTER.md
│   ├── API_CONTRACT.md
│   ├── VALIDATION_CASES.md
│   ├── OPEN_QUESTIONS.md
│   └── DECISIONS.md
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── api/
│   │   ├── schemas/
│   │   └── services/
│   └── tests/
├── engine/
│   ├── existing_section_solver/
│   ├── geometry/
│   ├── materials/
│   ├── prestressing/
│   ├── construction_stages/
│   ├── rheology/
│   ├── solvers/
│   ├── results/
│   └── tests/
├── frontend/
│   ├── index.html
│   ├── src/
│   │   ├── main.ts
│   │   ├── api/
│   │   ├── models/
│   │   ├── components/
│   │   ├── views/
│   │   └── plots/
│   └── styles/
├── validation/
│   ├── analytical/
│   ├── regression/
│   └── geho_bridge_girder/
└── examples/
```

La estructura podrá adaptarse después de inspeccionar el programa existente, pero cualquier cambio relevante debe anotarse en `docs/DECISIONS.md`.

## Modelo de dominio mínimo

Definir, como mínimo, las entidades siguientes:

- `ConcreteMaterial`
- `PassiveSteelMaterial`
- `PrestressingSteelMaterial`
- `SectionComponent`
- `GirderSection`
- `SlabSection`
- `Tendon`
- `LoadEvent`
- `ConstructionStage`
- `EvaluationTime`
- `StructuralState`
- `SectionResult`
- `TimeDependentResult`

`StructuralState` debe ser inmutable o tratarse como inmutable. Cada etapa recibe un estado anterior y produce un estado nuevo, sin modificar silenciosamente el historial.

## Integración del cálculo seccional existente

La primera tarea es localizar, ejecutar y caracterizar el programa actual. No modificarlo hasta disponer de una línea base reproducible.

El objetivo es encapsularlo tras una interfaz equivalente a:

```python
def solve_section(
    geometry,
    materials,
    axial_force_n,
    bending_moment_nmm,
    imposed_strain=0.0,
    imposed_curvature_per_mm=0.0,
    initial_strains=None,
) -> SectionResult:
    ...
```

La interfaz definitiva puede diferir, pero debe aceptar acciones generalizadas y devolver deformación axial, curvatura, tensiones por componente, resultantes internas y comprobaciones de equilibrio.

Antes de añadir prestaciones nuevas, crear ensayos de regresión que congelen varios resultados conocidos del programa actual.

## Pretensado

Implantarlo en dos niveles independientes.

### Control por acciones equivalentes

Para una fuerza de pretensado `P` y excentricidad `e_p`, comprobar el efecto mediante el axil y momento equivalentes, respetando el convenio de signos adoptado.

### Modelo compatible del tendón

Representar cada tendón mediante:

- área;
- posición;
- módulo elástico;
- tensión o predeformación inicial;
- instante de activación;
- tipo de adherencia;
- ley de relajación, cuando se incorpore el cálculo diferido.

La deformación del tendón debe ser compatible con la deformación plana de la sección cuando el modelo adoptado sea adherente. Las resultantes del acero activo deben entrar explícitamente en las ecuaciones de equilibrio.

Ambos niveles deben producir resultados coherentes en los casos en que sean equivalentes.

## Fases constructivas

El esquema temporal debe modelarse mediante eventos. Cada evento debe declarar:

- instante;
- componentes activos antes y después del evento;
- acción aplicada;
- sección sobre la que actúa;
- estado de referencia de cada material;
- resultados instantáneos y diferidos asociados.

Debe distinguirse entre:

- sección que soporta una carga al aplicarse;
- sección activa durante la evolución diferida posterior;
- edad de cada hormigón;
- instante de transferencia del pretensado;
- instante de activación estructural de la losa.

## Cálculo diferido

El primer motor diferido será el método acumulativo del coeficiente de envejecimiento, aplicado a secciones evolutivas no fisuradas.

Orden de implantación:

1. Evolución del módulo de elasticidad.
2. Retracción de cada hormigón.
3. Fluencia mediante una función coherente con el módulo adoptado.
4. Coeficiente de envejecimiento.
5. Tensiones de autotensión por retracción diferencial.
6. Interacción con el tendón.
7. Relajación del acero activo.
8. Acumulación por intervalos y fases.

El cálculo paso a paso completo se reserva para una segunda fase y servirá como referencia frente al método simplificado.

## Resultados mínimos

La aplicación debe devolver:

- propiedades geométricas de la viga, la losa y la sección compuesta;
- posición del centro de gravedad e inercias;
- fuerza y excentricidad iniciales del pretensado;
- pérdidas instantáneas y diferidas;
- deformación axial y curvatura por fase;
- tensiones en fibras relevantes de viga y losa;
- tensión en cada tendón;
- flecha instantánea y diferida en centro de vano para el modelo isostático inicial;
- historial temporal de tensiones, curvatura, flecha y pretensado;
- resultantes internas reconstruidas;
- error de equilibrio de axil y momento;
- advertencias y límites de validez.

## Validación

La validación se hará por niveles:

1. Propiedades de secciones rectangulares simples.
2. Sección compuesta de dos rectángulos.
3. Equilibrio elástico para axil y momento.
4. Pretensado mediante acciones equivalentes.
5. Pretensado mediante tendón compatible.
6. Comparación entre ambos modelos.
7. Sección compuesta sin efectos diferidos.
8. Retracción diferencial aislada.
9. Fluencia aislada.
10. Relajación aislada.
11. Secuencia constructiva completa.
12. Reproducción del ejemplo de puente de vigas del documento GEHO.

En cada estado deben comprobarse las resultantes internas frente a las externas. Las tolerancias deben definirse en `docs/VALIDATION_CASES.md`, justificarse y aplicarse de forma uniforme.

## API inicial

Prever, como mínimo, estas rutas:

```text
GET  /api/health
POST /api/section/instantaneous
POST /api/section/prestressed
POST /api/analysis/staged
POST /api/analysis/time-dependent
POST /api/project/validate
```

Los contratos definitivos se documentarán en `docs/API_CONTRACT.md` antes de desarrollar la interfaz completa.

## Interfaz inicial

La interfaz se organizará en:

- General.
- Geometría.
- Materiales.
- Pretensado.
- Fases.
- Cargas.
- Resultados.
- Comprobaciones.

Debe reducir el desplazamiento vertical, mostrar unidades junto a cada entrada, validar los datos antes de enviarlos y distinguir claramente errores, advertencias e información.

No desarrollar una interfaz completa hasta que la API y el motor dispongan de ensayos básicos.

## Trabajo multiagente

El agente coordinador debe delegar trabajo independiente en subagentes o chats separados. Distribución inicial:

### Agente A. Auditoría del motor existente

Inspeccionar estructura, entradas, salidas, unidades, convenciones, dependencias y ensayos existentes. Proponer la envolvente mínima sin reescribir el núcleo.

### Agente B. Especificación técnica

Preparar `PROJECT_SPEC.md`, `UNITS_AND_SIGNS.md` y `EQUATIONS_REGISTER.md`. Revisar visualmente en el PDF toda ecuación que vaya a implantarse.

### Agente C. Pretensado

Diseñar e implantar acciones equivalentes, tendones compatibles y ensayos de comparación.

### Agente D. Fases y reología

Diseñar estados temporales, eventos constructivos y el método acumulativo del coeficiente de envejecimiento.

### Agente E. Backend y API

Construir FastAPI y Pydantic sobre contratos acordados. No introducir lógica estructural en las rutas HTTP.

### Agente F. Frontend

Construir HTML, CSS y TypeScript contra una API estable o simulada mediante contratos fijos.

### Agente G. Validación independiente

Crear casos analíticos y reproducir el ejemplo GEHO. Este agente no debe corregir directamente el motor; debe informar de discrepancias reproducibles.

### Agente H. Integración

Revisar cambios, ejecutar todos los ensayos, resolver conflictos y mantener `DECISIONS.md` y `CHANGELOG.md`.

## Reglas de trabajo en paralelo

- Usar un repositorio Git.
- Cada trabajo paralelo debe usar una rama o worktree independiente.
- Evitar que dos agentes modifiquen simultáneamente los mismos archivos.
- Antes de programar, cada agente debe indicar archivos que modificará y criterios de aceptación.
- Los cambios se integran en bloques pequeños y comprobables.
- Ningún cambio entra en la rama principal con ensayos fallidos.
- El agente integrador debe revisar interfaces, unidades y signos, no solo conflictos de texto.
- Los hallazgos del agente de validación se corrigen mediante una tarea separada.

## Criterios de aceptación de la primera entrega

La primera entrega se considera válida cuando:

1. El programa seccional existente está encapsulado sin alterar sus resultados de referencia.
2. El pretensado instantáneo funciona mediante acciones equivalentes y tendón compatible.
3. Existen ensayos automáticos de regresión y equilibrio.
4. FastAPI expone el cálculo instantáneo y pretensado.
5. Una interfaz mínima permite introducir una sección, un tendón y acciones, ejecutar el cálculo y mostrar resultados.
6. El proyecto se ejecuta localmente mediante instrucciones reproducibles.
7. No se ha iniciado todavía el cálculo diferido sin cerrar unidades, signos y validación del pretensado.

## Primera orden al coordinador

Comienza por una fase de inspección, sin reescritura masiva.

1. Recorre el repositorio completo.
2. Identifica el programa seccional existente y explica su flujo de cálculo.
3. Localiza entradas, salidas, unidades, signos, dependencias, variables globales y acoplamientos con la interfaz.
4. Ejecuta los ensayos existentes. Si no existen, identifica ejemplos reproducibles que puedan convertirse en regresiones.
5. Propón la arquitectura mínima de adaptación al esquema anterior.
6. Delega en paralelo únicamente tareas de lectura y análisis que no modifiquen los mismos archivos.
7. Entrega un informe breve con:
   - mapa real del repositorio;
   - riesgos técnicos;
   - diferencias respecto a esta arquitectura propuesta;
   - plan de cambios por etapas;
   - primeros ensayos que deben crearse;
   - preguntas técnicas bloqueantes.
8. No implantes todavía el cálculo diferido ni rehagas la interfaz.
9. No modifiques el motor hasta que se haya creado una línea base verificable.

## Estado inicial conocido

- Existe un programa de cálculo seccional instantáneo.
- El programa todavía no incorpora pretensado.
- La interfaz objetivo será web.
- El backend objetivo será Python.
- La formulación diferida inicial procederá del documento GEHO de 1998, con verificación manual de ecuaciones.

Cualquier información no confirmada debe marcarse como pendiente y no asumirse como hecho.
