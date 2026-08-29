# Modelo de datos y medidas

Detalle técnico del informe. Para la lectura de negocio, ver el
[README](../README.md).

---

## Estructura del modelo

Esquema en estrella. Cinco tablas de hechos que cuelgan de dos dimensiones
compartidas, más dos tablas desconectadas que actúan como dimensión auxiliar.

### Tablas de hechos

| Tabla | Filas | Grano | Divisa |
|---|---|---|---|
| `General-Ledger` | 2.000 | Asiento contable | Por fila (5 divisas) |
| `Accounts-Receivable` | 900 | Factura de cliente | Por fila (5 divisas) |
| `Accounts-Payable` | 800 | Factura de proveedor | Por fila (5 divisas) |
| `Expense-Claims` | 1.000 | Nota de gasto | Por fila (5 divisas) |
| `Budget-Forecast` | 48 | Año × departamento × trimestre | Toda en USD |

### Dimensiones

| Tabla | Papel |
|---|---|
| `Tabla de Fechas` | Calendario 2023-2025. Columnas: Date, Año, AñoMes, AñoTrimestre, AñoTrimestreOrden |
| `TipoCambio` | Currency, TasaEUR. Además de convertir, hace de dimensión de divisa |
| `Tramo Aging` | Tramo, Orden. Dimensión de antigüedad compartida por AP y AR |
| `Eje Waterfall` | Parámetro de campo para el toggle departamento/trimestre |

### Por qué las dimensiones compartidas

Un gráfico que muestra cobros y pagos en el mismo eje no puede apoyarse en una
columna de una de las dos tablas de hechos. Si el eje fuera
`Accounts-Payable[Tramo Aging]`, la medida de cobros no encontraría por dónde
repartirse y colapsaría en un único bloque.

Por eso el tramo de antigüedad vive en `Tramo Aging`, relacionada uno a varios
con las dos tablas de facturas, y la divisa se toma de `TipoCambio` en lugar de
la columna `Currency` de una factura concreta. Es el mismo papel que cumple la
tabla de fechas para todas las tablas de hechos a la vez.

---

## Conversión a euros

Hay dos patrones distintos según si la tabla trae divisa a nivel de fila.

**Con divisa por fila** (libro mayor, facturas). Se recorre la relación con
`RELATED`, de modo que cada fila aplica la tasa de su propia moneda:

```dax
Ingresos EUR =
SUMX(
    'General-Ledger',
    'General-Ledger'[Credit] * RELATED(TipoCambio[TasaEUR])
)
```

**Sin columna de divisa** (`Budget-Forecast`, cuyos importes nacen todos en
dólares). No hay relación que recorrer, así que la tasa se busca fijando el
código de moneda:

```dax
Presupuesto EUR =
SUMX(
    'Budget-Forecast',
    'Budget-Forecast'[BudgetUSD] *
    LOOKUPVALUE(TipoCambio[TasaEUR], TipoCambio[Currency], "USD")
)
```

En ambos casos la conversión ocurre dentro de la medida, fila a fila, y no como
columna calculada. Así el modelo no guarda importes convertidos que quedarían
obsoletos si cambiaran las tasas.

---

## Medidas

### Presupuesto (páginas 1 y 2)

```dax
Presupuesto EUR =
SUMX('Budget-Forecast',
    'Budget-Forecast'[BudgetUSD] *
    LOOKUPVALUE(TipoCambio[TasaEUR], TipoCambio[Currency], "USD"))

Forecast EUR =
SUMX('Budget-Forecast',
    'Budget-Forecast'[ForecastUSD] *
    LOOKUPVALUE(TipoCambio[TasaEUR], TipoCambio[Currency], "USD"))

Real EUR =
SUMX('Budget-Forecast',
    'Budget-Forecast'[ActualUSD] *
    LOOKUPVALUE(TipoCambio[TasaEUR], TipoCambio[Currency], "USD"))

Desviación EUR = [Real EUR] - [Presupuesto EUR]

Desviación % = DIVIDE([Desviación EUR], [Presupuesto EUR])

% Ejecución = DIVIDE([Real EUR], [Presupuesto EUR])

Error Forecast EUR = [Real EUR] - [Forecast EUR]

Error Forecast % = DIVIDE([Error Forecast EUR], [Forecast EUR])

Precisión Forecast % = 1 - ABS([Error Forecast %])
```

**Convención de signo.** La desviación se presenta de forma que el negativo
sea sobrecoste y el positivo ahorro, coherente con el código de color del
gráfico de cascada (rojo y verde).

**Desviación frente a error de forecast.** Son dos preguntas distintas. La
desviación mide si se cumplió el presupuesto; el error de forecast mide si se
supo anticipar el desvío. Un departamento puede desviarse mucho del presupuesto
y aun así haberlo previsto con precisión, lo que apunta a un presupuesto mal
fijado más que a un descontrol del gasto.

### Tarjetas de hallazgo (página 2)

Devuelven texto, no números, para poder combinar el nombre de la categoría con
su valor en una sola tarjeta.

```dax
Depto Mayor Desviación =
VAR TablaDeptos =
    ADDCOLUMNS(
        VALUES('Budget-Forecast'[Departamento]),
        "@Desv", [Desviación %]
    )
VAR DeptoTop = TOPN(1, TablaDeptos, ABS([@Desv]), DESC)
RETURN
    CONCATENATEX(
        DeptoTop,
        'Budget-Forecast'[Departamento] & " (" &
        FORMAT([@Desv], "+0.0%;-0.0%") & ")"
    )
```

`Depto Mejor Precisión Forecast` y `Trimestre Mayor Desviación` siguen el mismo
patrón, cambiando la medida evaluada y el sentido de la ordenación.

```dax
Evolución Precisión Forecast =
VAR P2024 = CALCULATE([Precisión Forecast %], 'Budget-Forecast'[FiscalYear] = 2024)
VAR P2025 = CALCULATE([Precisión Forecast %], 'Budget-Forecast'[FiscalYear] = 2025)
RETURN
    FORMAT(P2025, "0.0%") & " (" & FORMAT(P2025 - P2024, "+0.0%;-0.0%") & ")"
```

### Libro mayor (página 3)

```dax
Ingresos EUR =
SUMX('General-Ledger', 'General-Ledger'[Credit] * RELATED(TipoCambio[TasaEUR]))

Gastos EUR =
SUMX('General-Ledger', 'General-Ledger'[Debit] * RELATED(TipoCambio[TasaEUR]))

Margen EUR = [Ingresos EUR] - [Gastos EUR]

Margen % = DIVIDE([Margen EUR], [Ingresos EUR])
```

Cuentas: 4000 (Sales Revenue) y 4010 (Online Sales) al Haber como ingresos;
5000 (COGS), 5010 (Travel) y 6000 (Payroll) al Debe como gastos.

### Circulante (página 4)

```dax
Pendiente Cobrar EUR =
SUMX(
    FILTER('Accounts-Receivable', ISBLANK('Accounts-Receivable'[ReceivedDate])),
    'Accounts-Receivable'[Amount] * RELATED(TipoCambio[TasaEUR])
)

Pendiente Pagar EUR =
SUMX(
    FILTER('Accounts-Payable', ISBLANK('Accounts-Payable'[PaidDate])),
    'Accounts-Payable'[Amount] * RELATED(TipoCambio[TasaEUR])
)

Posición Neta EUR = [Pendiente Cobrar EUR] - [Pendiente Pagar EUR]

% Cartera Vencida =
DIVIDE(
    CALCULATE([Pendiente Cobrar EUR], 'Accounts-Receivable'[Días Vencido] > 0),
    [Pendiente Cobrar EUR]
)

% Pendiente +90 días =
DIVIDE(
    CALCULATE([Pendiente Pagar EUR], 'Accounts-Payable'[Días Vencido] > 90),
    [Pendiente Pagar EUR]
)
```

Se considera pendiente toda factura sin fecha de cierre registrada. El `FILTER`
se aplica dentro del `SUMX` para que la conversión de divisa siga operando fila
a fila sobre el subconjunto.

---

## Columnas calculadas: antigüedad

Dos columnas en cada tabla de facturas. En `Accounts-Payable`:

```dax
Días Vencido =
VAR Corte = DATE(2025, 6, 30)
RETURN
IF(
    ISBLANK('Accounts-Payable'[PaidDate]),
    DATEDIFF('Accounts-Payable'[DueDate], Corte, DAY),
    BLANK()
)
```

```dax
Tramo Aging =
VAR d = 'Accounts-Payable'[Días Vencido]
RETURN
SWITCH(
    TRUE(),
    ISBLANK(d),  BLANK(),
    d <= 0,      "No vencido",
    d <= 30,     "1-30 días",
    d <= 60,     "31-60 días",
    d <= 90,     "61-90 días",
                 "+90 días"
)
```

Las mismas dos en `Accounts-Receivable`, sustituyendo `PaidDate` por
`ReceivedDate`. Los textos de los tramos deben coincidir carácter a carácter en
ambas tablas para que la relación con `Tramo Aging` funcione.

**La fecha de corte es fija y no `TODAY()`.** El conjunto de datos termina el
30 de junio de 2025; con la fecha real de ejecución, todo el histórico caería en
el tramo de más de noventa días y el análisis dejaría de distinguir nada.

La tabla `Tramo Aging` se crea a mano con Introducir datos, y su columna `Tramo`
se ordena por `Orden` para que el eje no salga alfabético (`+90 días` iría
primero por el carácter inicial).

| Tramo | Orden |
|---|---|
| No vencido | 1 |
| 1-30 días | 2 |
| 31-60 días | 3 |
| 61-90 días | 4 |
| +90 días | 5 |

---

## Interactividad

Toda la navegación por botones se apoya en marcadores, con dos reglas que
conviene respetar:

**Solo *Pantalla*, nunca *Datos*.** Un marcador que guarda el estado de los
datos restaura los filtros que hubiera en el momento de crearlo, de modo que al
abrir un panel se borraría la selección que el usuario acabara de hacer.

**Siempre *objetos visuales seleccionados*.** Un marcador con alcance sobre
todos los objetos memoriza la visibilidad de la página entera, y acaba
mostrando u ocultando elementos que pertenecen a otro mecanismo. Con el alcance
acotado, cada marcador toca únicamente lo suyo.

La navegación entre páginas no usa marcadores sino la acción *Navegación de
página*, que es más simple y no arrastra estados.

Los marcadores son objetos globales del informe pero capturan el estado de la
página donde se crearon, así que los conmutadores hay que recrearlos en cada
página donde se usen.

---

## Notas sobre los datos de origen

Las limitaciones del conjunto de datos están documentadas en el
[README](../README.md) y en el panel de información de cada página del informe.
