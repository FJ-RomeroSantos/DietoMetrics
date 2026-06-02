# DietoMetrics

**Sistema de optimización matemática para planificación nutricional clínica**

> Herramienta de apoyo para nutriólogos que convierte el cálculo manual de planes alimenticios en un proceso preciso, reproducible y hasta 10 veces más rápido — sin reemplazar el criterio profesional, sino potenciándolo.

🚀 **Prueba la aplicación en vivo:** [DietoMetrics App](https://dietometricssuit-production.up.railway.app)

---

## ¿Qué problema resuelve?

Un nutriólogo experimentado puede tardar entre 1 y 3 horas en construir un plan alimenticio desde cero: ajustar macros a calorías, calcular equivalencias del SMAE, distribuir grupos en tiempos de comida, seleccionar alimentos específicos y cuadrar los gramos finales. Cada paso implica prueba y error, reglas de tres manuales y revisiones que se reinician si algo cambia.

**DietoMetrics automatiza ese proceso de prueba y error con optimización matemática formal**, entregando una base matemáticamente correcta en segundos. El nutriólogo conserva el control total: puede ajustar cualquier parámetro, y el sistema recalcula instantáneamente manteniendo coherencia global.

Este proyecto es también una **demostración de diseño de algoritmos aplicados a un dominio real**: cómo la programación lineal, la programación lineal entera mixta (MIP) y el análisis de sensibilidad pueden resolver problemas prácticos complejos que de otro modo se abordan solo con heurística manual.

---

## Flujo completo del sistema

```
Datos del paciente
       ↓
  Cálculo de GET/TMB (Mifflin-St Jeor)
       ↓
  [LP] Optimización de macronutrientes
       ↓
  [MIP] Distribución de equivalencias SMAE
       ↓
  [Análisis de sensibilidad] Ajuste clínico interactivo
       ↓
  [LP] Optimización de gramos por alimento real
       ↓
  Plan semanal + exportación PDF
```

---

## Módulos y algoritmos

### Módulo 1 — Evaluación antropométrica

El sistema calcula automáticamente los parámetros clínicos del paciente a partir de sus datos basales.

**Métricas calculadas:**
- Tasa Metabólica Basal (TMB) con la ecuación de **Mifflin-St Jeor** (la más validada clínicamente para población adulta)
- Gasto Energético Total (GET) según nivel de actividad física (factores de Harris-Benedict modificados)
- IMC y clasificación OMS (bajo peso / normal / sobrepeso / obesidad)
- Rango de peso ideal (IMC 18.5–24.9)
- Calorías objetivo ajustadas por meta (déficit –500 kcal para pérdida, superávit +300 kcal para ganancia)

La fórmula de Mifflin-St Jeor utilizada:

```
Hombres: TMB = 10·peso(kg) + 6.25·talla(cm) − 5·edad + 5
Mujeres: TMB = 10·peso(kg) + 6.25·talla(cm) − 5·edad − 161
GET = TMB × factor_actividad
```

---

### Módulo 2 — Optimización de macronutrientes (Programación Lineal)

**El problema:** Dado un objetivo calórico y una distribución porcentual de macronutrientes (p. ej. 20% proteína / 50% carbohidratos / 30% grasa), encontrar los gramos exactos de cada macro que satisfagan ambas restricciones simultáneamente, considerando que cada macronutriente tiene un aporte calórico distinto (proteína = 4 kcal/g, carbohidratos = 4 kcal/g, grasa = 9 kcal/g).

**Por qué no es trivial:** La restricción de porcentaje y la restricción calórica son interdependientes y el sistema puede tener rangos definidos por el nutriólogo (mínimos/máximos por macro) que crean un espacio factible no obvio. La solución manual requiere múltiples iteraciones.

**Solución — Programación Lineal con variables de desviación:**

Se minimiza una función objetivo que penaliza la desviación calórica total y las desviaciones porcentuales por macro:

```
Minimizar: ε_kcal + 0.1·(ε_P + ε_C + ε_G)

Variables: xP, xC, xG ≥ 0  (gramos de proteína, carbohidrato, grasa)

Restricciones:
  |4xP + 4xC + 9xG − kcal_objetivo| ≤ ε_kcal
  |4xP − kcal_objetivo·%P| ≤ ε_P·kcal_objetivo
  |4xC − kcal_objetivo·%C| ≤ ε_C·kcal_objetivo
  |9xG − kcal_objetivo·%G| ≤ ε_G·kcal_objetivo
  xP ≥ min_P, xP ≤ max_P  (si el nutriólogo define límites)
  xC ≥ min_C, xC ≤ max_C
  xG ≥ min_G, xG ≤ max_G
```

El solver CBC (via PuLP) encuentra la solución en milisegundos. El sistema además detecta automáticamente si la distribución está fuera de rangos fisiológicos normales (proteína 10–35%, carbohidratos 45–65%, grasa 20–35%) y lo reporta como contexto clínico especial.

**Resultado:** Gramos precisos de cada macronutriente, con desviación calórica reportada y distribución porcentual real del resultado.

---

### Módulo 3 — Distribución de equivalencias SMAE (Programación Lineal Entera Mixta)

**El problema:** Dada la cantidad de gramos de cada macronutriente, determinar cuántas porciones de cada grupo alimentario del SMAE (Sistema Mexicano de Alimentos Equivalentes del INCMNSZ) satisfacen esos requerimientos, considerando que el nutriólogo puede preferir ciertos grupos y restringir o excluir otros.

**Por qué es difícil:** Las porciones son enteras (no se pueden prescribir 2.7 porciones de cereal), hay 19 grupos posibles con diferentes combinaciones de macros por porción, y el espacio de búsqueda para combinaciones que satisfagan las restricciones simultáneamente es enorme para hacerlo manualmente.

**Solución — MIP con rangos clínicos y preferencias:**

```
Minimizar: w·(ε_P + ε_C + ε_G) + Σ d_g

Variables: x_g ∈ Z≥0  (porciones del grupo g)
           d_g ≥ 0     (desviación del rango clínico esperado)

Restricciones:
  Para cada grupo g:
    d_g ≥ x_g − rango_max(g)
    d_g ≥ rango_min(g) − x_g
    x_g ≥ porc_min_usuario(g)   (restricción dura del nutriólogo)
    x_g ≤ porc_max_usuario(g)

  Restricciones de macros (suaves, penalizadas):
    |Σ proteina_g · x_g − obj_P| ≤ ε_P
    |Σ hc_g · x_g − obj_C| ≤ ε_C
    |Σ grasa_g · x_g − obj_G| ≤ ε_G
```

Los rangos clínicos por grupo (p. ej. verduras 2–5 porciones, cereales sin grasa 4–8) definen el espacio "realista". El peso `w` se incrementa automáticamente (de 10 a 30) para casos clínicos extremos, priorizando el cumplimiento de macros sobre la preferencia de rango.

**Tabla de equivalencias SMAE embebida (19 grupos):**

| Grupo | Proteína (g) | HC (g) | Grasa (g) | kcal/porción |
|-------|:-----------:|:------:|:---------:|:------------:|
| Verduras | 2 | 4 | 0 | 24 |
| Frutas | 0 | 15 | 0 | 60 |
| Cereales sin grasa | 2 | 15 | 0 | 68 |
| Cereales con grasa | 2 | 15 | 5 | 113 |
| Leguminosas | 8 | 20 | 1 | 121 |
| AOA muy bajo aporte grasa | 7 | 0 | 1 | 37 |
| AOA bajo aporte grasa | 7 | 0 | 3 | 55 |
| AOA moderado aporte grasa | 7 | 0 | 5 | 73 |
| AOA alto aporte grasa | 7 | 0 | 8 | 100 |
| Leche descremada | 9 | 12 | 2 | 94 |
| Grasas sin proteína | 0 | 0 | 5 | 45 |
| Azúcares sin grasa | 0 | 10 | 0 | 40 |
| *(+ 7 grupos adicionales)* | | | | |

---

### Módulo 4 — Análisis de sensibilidad interactivo

**El problema:** La solución matemática del MIP puede ser algebraicamente correcta pero clínicamente poco realista (p. ej. 8 porciones de leguminosas en un día). El nutriólogo necesita poder restringir grupos individuales sin que el sistema "se rompa" y deje de encontrar soluciones, y sin tener que reiniciar el proceso desde cero.

**La clave del diseño:** En lugar de resolver el problema de nuevo cada vez que el usuario modifica una restricción (lo que puede llevar a infactibilidad), se implementa un **análisis de sensibilidad orientado por conocimiento clínico**: se evalúa cuánto se aleja la solución actual de satisfacer la nueva restricción y se calcula el ajuste mínimo necesario.

**Algoritmo de ajuste:**

Cuando el usuario define nuevas restricciones por grupo (porciones mínimas/máximas):

1. Se intenta resolver el MIP con las nuevas restricciones manteniendo las variables enteras
2. Si la solución es factible y la desviación de macros es < 5g: se acepta directamente
3. Si hay conflicto (infactibilidad): se activa el **módulo de relajación**, que introduce variables de holgura `s_g` y variables binarias de presencia `z_g`:

```
Minimizar: 100·Σ s_g − 10·Σ z_g + 50·(ε_P + ε_C + ε_G)

Con:
  x_g ≤ M·z_g      (si z_g = 0, el grupo no aparece)
  x_g ≥ z_g        (cada grupo presente tiene al menos 1 porción)
  restricciones_usuario − s_g  (relajadas por la holgura)
```

El resultado es el **ajuste mínimo** que hace factible el sistema, reportando exactamente qué grupos y cuántas porciones se modificaron y por qué.

**Ventaja computacional:** Al trabajar sobre la solución existente en lugar de resolver desde cero, se ahorra poder de cómputo y se mantiene la coherencia con el estado previo del plan.

---

### Módulo 5 — Optimización de gramos por alimento real (LP Continua)

**El problema:** Las equivalencias del SMAE son promedios por grupo. Cuando el nutriólogo selecciona alimentos específicos dentro de cada grupo (p. ej. "pechuga de pollo" dentro de AOA bajo aporte de grasa), los valores reales de macros y calorías difieren del promedio de equivalencia. El menú resultante puede cuadrar en kcal totales pero desviarse en los macros reales por comida.

**Demostración del error:** Si el sistema asigna 30g de pechuga de pollo como 1 porción de AOA bajo grasa (equivalencia: 7g proteína, 0g HC, 3g grasa = 55 kcal), pero la pechuga real aporta 5.3g grasa/100g en lugar del 3g/porción del promedio, al escalar a las porciones del día la diferencia acumulada puede llegar a 15–25g de grasa, invalidando el plan.

**Solución — LP Continua por tiempo de comida:**

Para cada tiempo de comida, se optimizan los gramos de cada alimento seleccionado simultáneamente:

```
Variables: g_i ∈ [g_min, g_max]  (gramos del alimento i, continuas)

Minimizar: 10·(ε_P + ε_C + ε_G) + 1·ε_kcal

Restricciones:
  |Σ (P_i/100)·g_i − target_P_comida| ≤ ε_P
  |Σ (HC_i/100)·g_i − target_C_comida| ≤ ε_C
  |Σ (G_i/100)·g_i − target_G_comida| ≤ ε_G
  |Σ (kcal_i/100)·g_i − target_kcal_comida| ≤ ε_kcal

  Para cada grupo dentro de la comida:
    |kcal_grupo_en_comida − presupuesto_grupo| ≤ 30%·presupuesto + slack_grupo
    (restricción blanda — penaliza alejarse del presupuesto por grupo)
```

Los macros tienen peso 10:1 sobre las calorías en la función objetivo, priorizando la precisión de macronutrientes. Las restricciones blandas por grupo permiten compensar entre alimentos del mismo grupo sin sacrificar los targets globales.

**Output:** Gramos precisos y reales de cada alimento en cada tiempo de comida, expresados también en las unidades de medida habituales del alimento (tazas, piezas, cucharadas) para facilitar la prescripción clínica.

---

### Módulo 6 — Calculadora de alimentos complementaria

Herramienta independiente (incluida en el mismo repositorio) que permite calcular el desglose completo de macronutrientes para cualquier combinación de alimentos de la base de datos. Útil para:

- Cálculos rápidos durante consulta
- Verificación de combinaciones específicas
- Comparación de opciones alimentarias
- Estimación de densidades calóricas

Accede a los mismos 2,863 alimentos de la BD SMAE del INCMNSZ con sus valores por 100g y por porción sugerida.

---

## Stack tecnológico

| Capa | Tecnología | Rol |
|------|-----------|-----|
| Optimización | **PuLP + CBC solver** | Resolutor LP/MIP para todos los módulos de optimización |
| Backend | **FastAPI** | API REST que expone los endpoints de cada módulo |
| Base de datos | **SQLite** (alimentos5.db) | 2,863 alimentos SMAE con macros completos por porción |
| Frontend | **React + Vite** | Interfaz de 5 pasos con estado persistente |
| Visualización | **Recharts** | Donut charts, barras y comparativas en tiempo real |
| Exportación | **jsPDF + autotable** | Generación de PDF profesional en cliente (sin servidor) |
| Datos | **INCMNSZ / SMAE** | Sistema Mexicano de Alimentos Equivalentes como referencia |

---

## Estructura del repositorio

```
dietometrics/
│
├── 📓 notebooks/
│   └── dietometrics_algorithms.ipynb   ← Desarrollo y explicación de todos los algoritmos
│
├── 🖥️  dietometrics-v4/                 ← Aplicación principal (planificador nutricional)
│   ├── main.py                          ← Backend FastAPI con todos los endpoints
│   ├── alimentos5.db                    ← Base de datos SMAE (2,863 alimentos)
│   ├── requirements.txt
│   ├── package.json
│   └── src/
│       ├── App.jsx                      ← Orquestador de los 5 pasos
│       ├── components/
│       │   ├── StepPaciente.jsx         ← Paso 1: datos antropométricos
│       │   ├── StepMacros.jsx           ← Paso 2: optimización de macros
│       │   ├── StepGrupos.jsx           ← Paso 3: equivalencias + sensibilidad
│       │   ├── StepMenu.jsx             ← Paso 4: selección y optimización de menú
│       │   └── StepReceta.jsx           ← Paso 5: receta final, stats y exportación
│       └── utils/
│           ├── api.js                   ← Capa de comunicación con el backend
│           └── exportPDF.js             ← Generador de PDF clínico
│
├── 🥗  dieto_metrics/                   ← Calculadora de alimentos (herramienta complementaria)
│   ├── backend/
│   │   ├── main.py                      ← FastAPI: endpoints de consulta y cálculo
│   │   └── alimentos5.db
│   └── frontend/
│       └── src/
│           ├── App.jsx
│           └── components/
│               ├── Sidebar.jsx          ← Selección y búsqueda de alimentos
│               ├── FoodTable.jsx        ← Tabla de alimentos seleccionados
│               ├── NutritionSummary.jsx ← Resumen de macros
│               └── MacroChart.jsx       ← Visualización de distribución
│
├── 📄 docs/
│   ├── ARCHITECTURE.md                  ← Decisiones de diseño y arquitectura
│   ├── ALGORITHMS.md                    ← Referencia matemática detallada
│   └── screenshots/                     ← Capturas de la aplicación
│
└── README.md
```

---

## Instalación y uso

### Requisitos

- Python 3.9+
- Node.js 18+
- pip

### Aplicación principal (planificador)

```bash
# 1. Clonar el repositorio
git clone https://github.com/tu-usuario/dietometrics.git
cd dietometrics/dietometrics-v4

# 2. Backend
pip install -r requirements.txt
uvicorn main:app --reload
# API disponible en http://localhost:8000
# Documentación interactiva en http://localhost:8000/docs

# 3. Frontend (nueva terminal)
npm install
npm run dev
# App disponible en http://localhost:5173
```

### Calculadora de alimentos

```bash
cd dietometrics/dieto_metrics
chmod +x start.sh
./start.sh
# App disponible en http://localhost:5173
```

### Notebook de algoritmos

```bash
pip install jupyter pulp numpy pandas matplotlib
jupyter notebook notebooks/dietometrics_algorithms.ipynb
```

---

## API — Endpoints principales

| Método | Ruta | Descripción |
|--------|------|-------------|
| `POST` | `/paciente/calcular` | TMB, GET, IMC y calorías objetivo |
| `POST` | `/caso/macros` | Optimización LP de macronutrientes |
| `POST` | `/caso/equivalencias` | Distribución MIP de grupos SMAE |
| `POST` | `/caso/sensibilidad` | Análisis de sensibilidad y ajuste de restricciones |
| `POST` | `/menu/optimizar` | Optimización LP de gramos por alimento real |
| `GET`  | `/alimentos` | Búsqueda en los 2,863 alimentos de la BD |
| `GET`  | `/grupos-equivalencias` | Grupos SMAE con macros por porción |
| `GET`  | `/docs` | Swagger UI interactivo |

---

## Flujo de la interfaz (5 pasos)

**Paso 1 — Paciente:** Captura datos antropométricos. Calcula automáticamente TMB, GET e IMC. Propone calorías objetivo basadas en la fórmula de Mifflin-St Jeor y el objetivo del paciente (bajar / mantener / subir). Muestra figura corporal SVG dinámica.

**Paso 2 — Macros:** El nutriólogo ajusta las calorías objetivo (pre-cargadas del paso anterior) y define los porcentajes de macronutrientes con sliders. El sistema resuelve el LP y muestra los gramos resultantes con un donut chart en tiempo real. Permite definir rangos mínimos/máximos por macro.

**Paso 3 — Grupos y sensibilidad:** Selección de grupos alimentarios SMAE a incluir. El sistema resuelve el MIP y distribuye porciones. Panel de análisis de sensibilidad: el nutriólogo puede restringir porciones mínimas/máximas por grupo y ver en tiempo real si las restricciones son compatibles (✓ excelente / ⚠ aceptable / ✗ conflicto) y cuánto margen existe en cada grupo.

**Paso 4 — Menú:** Configuración de tiempos de comida (2 a 6 por día). Para cada tiempo, selección libre de alimentos por grupo con búsqueda en los 2,863 alimentos de la BD. El sistema optimiza los gramos de cada alimento para que el menú cuadre tanto en calorías como en macros reales.

**Paso 5 — Receta y exportación:** Vista completa del plan con estadísticas (4 visualizaciones: donut por tiempo de comida, barras objetivo vs real, ranking calórico por grupo, stack de macros por tiempo). Exportación a PDF profesional con datos del paciente, macros, grupos SMAE y menú detallado. Generación de ficha médica y archivo del paciente para continuar en consultas futuras.

**Planes semanales:** Permite crear planes para diferentes días, duplicar información entre días y hacer ajustes específicos por día (descanso vs entrenamiento, días de mayor ingesta, etc.).

---

## Limitaciones conocidas y trabajo futuro

Este sistema genera **aproximaciones matemáticamente óptimas dentro de sus restricciones**, no reemplaza el juicio clínico del nutriólogo. Las principales limitaciones son:

- Las equivalencias SMAE son valores promedio por grupo; la variabilidad real entre alimentos del mismo grupo se compensa en el Módulo 5, pero pueden persistir pequeñas desviaciones
- El sistema no considera interacciones entre nutrientes, biodisponibilidad ni factores clínicos específicos (enfermedad renal, diabetes, etc.) más allá de los parámetros configurables
- La base de datos cubre alimentos del SMAE del INCMNSZ; productos manufacturados o recetas compuestas requieren carga manual

**Próximas funcionalidades planeadas:**
- [ ] Autenticación y gestión de múltiples pacientes
- [ ] Empaquetado como aplicación de escritorio (Electron)
- [ ] Módulo de patologías específicas con restricciones predefinidas (IRC, diabetes tipo 2, etc.)
- [ ] Exportación a formato compatible con otros sistemas de consulta

---

## Referencia matemática

Para una explicación detallada de cada formulación LP/MIP, incluyendo demostraciones de convergencia, análisis de complejidad y ejemplos numéricos paso a paso, ver el notebook `notebooks/dietometrics_algorithms.ipynb`.

---

## Sobre el proyecto

DietoMetrics nació de observar el proceso real de consulta nutricional y identificar dónde la optimización matemática podía aportar valor concreto: no en el diagnóstico ni en la relación terapéutica, sino en los cálculos repetitivos que consumen tiempo y concentración valiosos.

El diseño de cada módulo priorizó que el resultado fuera **interpretable y ajustable por el nutriólogo**, no una caja negra. El análisis de sensibilidad, los rangos clínicos embebidos y la capacidad de modificar cualquier parámetro en cualquier paso son decisiones de diseño deliberadas para que la herramienta complemente, no reemplace, el conocimiento profesional.

---

## Licencia

MIT — libre para uso educativo y clínico no comercial. Ver `LICENSE` para detalles.
