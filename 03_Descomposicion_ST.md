Descomposición de series de tiempo
================

  - [Transformaciones y ajustes](#transformaciones-y-ajustes)
      - [Ajustes de calendario](#ajustes-de-calendario)
      - [Ajustes poblacionales](#ajustes-poblacionales)
      - [Ajustes inflacionarios](#ajustes-inflacionarios)
      - [Transformaciones matemáticas](#transformaciones-matemáticas)
  - [Componentes de las series de
    tiempo](#componentes-de-las-series-de-tiempo)
      - [Datos desestacionalizados](#datos-desestacionalizados)
  - [Medias móviles](#medias-móviles)
      - [Medias móviles de medias
        móviles](#medias-móviles-de-medias-móviles)
      - [Medias móviles ponderadas](#medias-móviles-ponderadas)
  - [Métodos de descomposición](#métodos-de-descomposición)
      - [Descomposición clásica](#descomposición-clásica)
      - [Descomposición X11](#descomposición-x11)
      - [Descomposición SEATS](#descomposición-seats)
      - [Descomposición STL](#descomposición-stl)
  - [Tarea](#tarea)

Como hemos visto, las series de tiempo pueden presentar varios patrones,
y comúnmente es útil dividir o descomponer la serie en cada uno de esos
patrones. Recordando, una serie de tiempo puede exhibir:

  - Tendencia
  - Estacionalidad
  - Ciclos

Al realizar la descomposición de la serie, usualmente se juntan el
patrón cíclico y la tendencia en uno solo, llamado simplemente
**tendencia**. Así, tendríamos tres componentes en una serie de tiempo
cualquiera:

1.  Tendencia
2.  Estacionalidad
3.  Residuo (lo que no es parte ni de la tendencia, ni del efecto
    estacional)

# Transformaciones y ajustes

Ahora, lo que buscamos es analizar datos lo más sencillos posibles, por
lo que en ocasiones vale la pena llevar a cabo *transformaciones* o
*ajustes* de la serie de tiempo. Vamos a revisar cuatro tipos: ajustes
de calendario, por población, inflacionarios y transformaciones
matemáticas.

### Ajustes de calendario

Parte de la variación en las series de tiempo puede deberse a efectos
muy sencillos que tienen que ver con el calendario. Algunos ejemplos de
esto pueden ser variaciones en datos mensuales, debido a la cantidad de
días hábiles en cada mes. Tomemos el caso del volumen de transacciones
de Alphabet Inc., holding de Google, entre otras.

``` r
if (!require("easypackages")) install.packages("easypackages")
library("easypackages")
packages("tidyverse","quantmod","lubridate", "patchwork", "fpp2","fpp3","scales")



getSymbols("GOOG", from = "2015-01-01")

transacciones_mensuales <- GOOG %>% 
  as_tibble() %>%
  mutate(date = zoo::index(GOOG), 
         year = year(date), 
         month = month(date)) %>% 
  unite(year_mon, year, month,sep = "-") %>%
  mutate(year_mon = yearmonth(year_mon)) %>% 
  select(date, year_mon, Volumen = GOOG.Volume) %>% 
  group_by(year_mon) %>% 
  summarise(Monthly_volume = sum(Volumen),
            trading_days = n(),
            mean_volume = mean(Volumen)) %>% 
  as_tsibble(index = year_mon)
  
  

p1 <- ggplot(data = transacciones_mensuales) + 
  geom_line(aes(x = year_mon, y = Monthly_volume)) +
  ylab("Vol. total mensual") + 
  xlab("mes")

p2 <- ggplot(data = transacciones_mensuales) + 
  geom_line(aes(x = year_mon, y = mean_volume)) +
  ylab("Vol. promedio diario") +
  xlab("mes")

p1 / p2
```

![](03_Descomposicion_ST_files/figure-gfm/ajuste%20calendario%20google-1.jpeg)<!-- -->

La serie original tomando el total de transacciones mensuales presenta
mayor variación que cuando ajustamos la serie por los dias calendario
(llegando al volumen promedio diario).

### Ajustes poblacionales

Cualquier variable que se ve afectada por la población, puede ser
expresada en términos *per cápita*. Si, p. ej., con la pandemia del
COVID-19, se quisiera analizar la cantidad de médicos laborando en
México y ver su evolución histórica, tomar únicamente la cantidad de
médicos podría ser engañoso, si no se toma en consideración el
crecimiento de la población. Así, sería más recomendable observar la
variable de cantidad de *médicos por cada 100 mil habitantes*, por
ejemplo, para ver si la cantidad de médicos ha aumentado, se ha
mantenido estable o ha disminuido a lo largo del tiempo.

Otro ejemplo que se puede analizar es el del PIB de los países. Tomemos
tres casos: México, Australia e Islandia. Si consideramos el PIB para
ver qué tan bien se ha comportado la economía de cada uno de los países,
veríamos el siguiente efecto:

``` r
glimpse(global_economy)
```

    ## Rows: 15,150
    ## Columns: 9
    ## Key: Country [263]
    ## $ Country    <fct> Afghanistan, Afghanistan, Afghanistan, Afghanistan, Afgh...
    ## $ Code       <fct> AFG, AFG, AFG, AFG, AFG, AFG, AFG, AFG, AFG, AFG, AFG, A...
    ## $ Year       <dbl> 1960, 1961, 1962, 1963, 1964, 1965, 1966, 1967, 1968, 19...
    ## $ GDP        <dbl> 537777811, 548888896, 546666678, 751111191, 800000044, 1...
    ## $ Growth     <dbl> NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, ...
    ## $ CPI        <dbl> NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, ...
    ## $ Imports    <dbl> 7.024793, 8.097166, 9.349593, 16.863910, 18.055555, 21.4...
    ## $ Exports    <dbl> 4.132233, 4.453443, 4.878051, 9.171601, 8.888893, 11.258...
    ## $ Population <dbl> 8996351, 9166764, 9345868, 9533954, 9731361, 9938414, 10...

``` r
ge <- global_economy %>% 
  filter(Country == "Mexico" | Country == "Iceland" | Country == "Australia")

p3 <- ggplot(ge) + aes(x = Year, y = GDP, color = Country) + 
  geom_line()

p3
```

![](03_Descomposicion_ST_files/figure-gfm/countries%20gdp-1.jpeg)<!-- -->

¿Realmente creen que la economía de México esté a la par de la economía
de Australia e, incluso, muy superior a la de Islandia?

Lo cierto es que no. La variable del PIB no está considerando la
población de cada país; México tiene una población mucho mayor que la
de los otros dos países:

``` r
ggplot(ge) + aes(x = Year, y = Population, color = Country) +
  geom_line()
```

![](03_Descomposicion_ST_files/figure-gfm/countries%20population-1.jpeg)<!-- -->

Ahora, si ajustamos los datos del PIB para tomar en cuenta la población,
obtendríamos el **PIB per cápita**.

``` r
p4 <- ggplot(ge) + aes(x = Year, y = GDP / Population, color = Country) +
  geom_line() + ylab("GDP per capita")

p4
```

![](03_Descomposicion_ST_files/figure-gfm/countries%20gdp%20per%20capita-1.jpeg)<!-- -->

Aquí queda claro que la economía de Australia e Islandia es bastante
superior a la mexicana (lo producido en México por persona es mucho
menor que lo producido por persona en Australia e Islandia).

### Ajustes inflacionarios

Los datos afectados por el valor del dinero en el tiempo se deben
ajustar antes de modelarse. Esto se debe a la inflación: sus abuelos
podían comprar una coca y un gansito tal vez con menos de 3 pesos.
¿Cuánto dinero necesitarían hoy para comprar una coca y un gansito?
Entonces, las series de tiempo financieras se ajustan a la inflación, y
se expresan en términos de alguna moneda constante (en el tiempo). Por
ejemplo, se puede medir el valor de la vivienda en Gudalajara, con
precios constantes de 2010 (imaginando que no hubiera cambiado el valor
del dinero en el tiempo).

Para realizar el ajuste inflacionario, se toma un índice de precios. Si
\(z_t\) es el índice de precios y \(y_t\) es el valor original de la
vivienda en el tiempo \(t\), entonces el precio de la vivienda, \(x_t\),
ajustado a valor del año 2010, estaría dado por:

\[
x_t = \frac{y_t}{z_t} * z_{2010}
\] Los índices de precios son construidos, generalmente, por el
gobierno, o alguna dependencia de gobierno. En México, el más común es
el Índice Nacional de Precios al Consumidor (INPC), que es construido
por el INEGI.

Veamos el caso de la industria de periódicos y libros impresos en
Australia y su crecimiento o decrecimiento en el tiempo. Tomaremos los
datos de `aus_retail` y ajustaremos la inflación con el índice de
precios al consumidor, `CPI`, dentro de la tabla `global_economy`.

``` r
print_retail <- aus_retail %>%
  filter(Industry == "Newspaper and book retailing") %>%
  group_by(Industry) %>%
  index_by(Year = year(Month)) %>%
  summarise(Turnover = sum(Turnover))
aus_economy <- global_economy %>%
  filter(Code == "AUS")
print_retail %>%
  left_join(aus_economy, by = "Year") %>%
  mutate(Adjusted_turnover = Turnover / CPI) %>%
  gather("Type", "Turnover", Turnover, Adjusted_turnover, factor_key = TRUE) %>%
  ggplot(aes(x = Year, y = Turnover)) +
    geom_line() +
    facet_grid(vars(Type), scales = "free_y") +
    xlab("Years") + ylab(NULL) +
    ggtitle("Turnover for the Australian print media industry")
```

    ## Warning: Removed 1 row(s) containing missing values (geom_path).

![](03_Descomposicion_ST_files/figure-gfm/inflation%20adjusted%20printing%20industry-1.jpeg)<!-- -->

Al ver los datos ajustados, nos percatamos de que la venta de estos
productos impresos ha decaído mucho más de lo que los datos sin ajustar
sugieren.

### Transformaciones matemáticas

Si los datos presentan variación que aumenta o disminuye con el nivel de
la serie, se puede sugerir una transformación matemática. Las
**transformaciones logarítmicas** son muy utilizadas. Una de las razones
es porque son fácilmente interpretables: los cambios en el valor
logarítmico son relativos (o porcentuales), a cambios en la escala
original. Una desventaja de la transformación logarítmica, es que no se
puede utilizar con valores iguales a cero o negativos.

Recordando clases anteriores, donde teníamos las ventas trimestrales de
J\&J. Se observa que la variación aumenta con el nivel de la serie.
Asimismo, parece tener una forma exponencial.

``` r
data("JohnsonJohnson")
autoplot(JohnsonJohnson)+
  ggtitle("Ventas trimestrales de J&J")
```

![](03_Descomposicion_ST_files/figure-gfm/jj%20Q%20sales-1.jpeg)<!-- -->

Podemos probar aplicando una transformación logarítmica a los datos para
ver si logramos hacer que la variación sea uniforme a lo largo del
tiempo, y ver si podemos hacer que parezca tener una tendencia lineal.

``` r
log_jj <- log(JohnsonJohnson)

autoplot(log_jj)+
  ggtitle("Logaritmo de las ventas trim. de J&J")
```

![](03_Descomposicion_ST_files/figure-gfm/log%20jj-1.jpeg)<!-- -->

Otro tipo de transformaciones matemáticas son las **transformaciones de
potencia**. P. ej., sacar la raíz cuadrada o cúbica de los datos, etc.
Estas transformaciones se pueden escribir como \(w_t = y_t^p\).

Una familia de transformaciones matemáticas que incluye, tanto
logaritmos, como potencias son las **transformaciones Box-Cox**, que
dependen de un parámetro, \(\lambda\) y se definen de la siguiente
manera:

\[w_{t}=\left\{\begin{array}{ll}
\log \left(y_{t}\right) & \text { si } \lambda=0 \\
\left(y_{t}^{\lambda}-1\right) / \lambda & \text { en otro caso }
\end{array}\right.\]

Aquí, el logaritmo siempre es un logaritmo natural. Si \(\lambda = 0\),
se utiliza un logaritmo natural, de lo contrario se utiliza una
potencia, con un escalado simple.

Si \(\lambda = 1\) entonces \(w_t = y_t-1\), lo que significaría que los
datos solo se desplazarían hacia abajo, sin cambiar la forma de la serie
de tiempo. Para cualquier otro valor de \(\lambda\), la serie cambiara
de forma.

*¿Cómo escoger el valor de \(\lambda\) a utilizar?*

Un buen valor de \(\lambda\) es aquel que hace que el tamaño de la
variación estacional sea el mismo a lo largo de la serie, para que el
modelado y pronóstico sea más sencillo.

Para ejemplificar esto, tomemos datos sobre la producción de gas en
Australia.

``` r
p5a <- aus_production %>% autoplot(Gas)+ 
  ggtitle("Producción de gas (datos reales)")

p5 <- aus_production %>% autoplot(box_cox(Gas,lambda = -0.5)) + ggtitle("Box-Cox, lambda = -0.5")

p6 <- aus_production %>% autoplot(box_cox(Gas,lambda = 0)) + ggtitle("Box-Cox, lambda = 0")

p7 <- aus_production %>% autoplot(box_cox(Gas,lambda = 0.1)) + ggtitle("Box-Cox, lambda = 0.1")

p8 <- aus_production %>% autoplot(box_cox(Gas,lambda = 1)) + ggtitle("Box-Cox, lambda = 1")

p5a
```

![](03_Descomposicion_ST_files/figure-gfm/box%20cox%20transform-1.jpeg)<!-- -->

``` r
(p5 | p6) / (p7 | p8)
```

![](03_Descomposicion_ST_files/figure-gfm/box%20cox%20transform-2.jpeg)<!-- -->

``` r
# sliderInput("lambda",
#   label = "Selecciona el valor de lambda: ",
#   min = -1, max = 2, value = 0,
#   step = 0.1, animate = F
# )
# 
# renderPlot({
#   aus_production %>% 
#     autoplot(box_cox(Gas,lambda = input$lambda)) + ggtitle("Transformaciones de Box-Cox")
# })
```

La característica de `guerrero` nos ayuda a seleccionar un valor de
\(\lambda\). En este caso, escogió un `$\lambda = 0.12$`

``` r
(lambda <- aus_production %>%
  features(Gas, features = guerrero) %>%
  pull(lambda_guerrero))
```

    ## [1] 0.1204864

``` r
aus_production %>% autoplot(box_cox(Gas, lambda))
```

![](03_Descomposicion_ST_files/figure-gfm/box%20cox%20guerrero%20feature-1.jpeg)<!-- -->

# Componentes de las series de tiempo

Si empleamos una **descomposición aditiva**, se podría explicar a
nuestra serie de tiempo, \(y_t\) como:

\[
y_t = S_t + T_t + R_t
\]

Donde \(S_t\) representa el componente estacional, \(T_t\) la
tendencia-ciclo y \(R_t\) el residuo. Una **descomposición
multiplicativa** tomaría la forma:

\[
y_t = S_t * T_t * R_t
\]

*¿Cuándo utilizar la descomposición aditiva y cuándo la multiplicativa?*

La aditiva es mejor cuando la magnitud de la fluctuación estacional no
cambia con el nivel de la serie. Por el contrario, cuando la variación
del componente estacional cambia a lo largo del tiempo, se recomendaría
utilizar la multiplicativa. Esta última es muy común en series de tiempo
económicas.

Una alternativa para no utilizar la descomposición multiplicativa es
transformar los datos para evitar la fluctuación en la variación y
posteriormente aplicar la aditiva. De hecho, si se aplica una
transformación logarítmica a los datos, es equivalente a utilizar una
descomposición multiplicativa, porque:

\[y_{t}=S_{t} \times T_{t} \times R_{t} \quad \text { es equivalente a } \quad \log y_{t}=\log S_{t}+\log T_{t}+\log R_{t-}\]
Veamos el ejemplo del empleo en el sector de las ventas al menudeo en
EEUU desde 1990.

``` r
us_retail_employment <- us_employment %>%
  filter(year(Month) >= 1990, Title == "Retail Trade") %>%
  select(-Series_ID)
us_retail_employment %>%
  autoplot(Employed) +
  xlab("Year") + ylab("Persons (thousands)") +
  ggtitle("Total employment in US retail")
```

![](03_Descomposicion_ST_files/figure-gfm/retail%20employment-1.jpeg)<!-- -->
Vamos a llevar a cabo un tipo de descomposición llamado **STL** (que
analizaremos con mayor detalle posteriormente).

``` r
dcmp <- us_retail_employment %>%
  model(STL(Employed))
components(dcmp)
```

    ## # A dable:           357 x 7 [1M]
    ## # Key:               .model [1]
    ## # STL Decomposition: Employed = trend + season_year + remainder
    ##    .model            Month Employed  trend season_year remainder season_adjust
    ##    <chr>             <mth>    <dbl>  <dbl>       <dbl>     <dbl>         <dbl>
    ##  1 STL(Employed) 1990 ene.   13256. 13291.       -38.1    3.08          13294.
    ##  2 STL(Employed) 1990 feb.   12966. 13272.      -261.   -44.2           13227.
    ##  3 STL(Employed) 1990 mar.   12938. 13252.      -291.   -23.0           13229.
    ##  4 STL(Employed) 1990 abr.   13012. 13233.      -221.     0.0892        13233.
    ##  5 STL(Employed) 1990 may.   13108. 13213.      -115.     9.98          13223.
    ##  6 STL(Employed) 1990 jun.   13183. 13193.       -25.6   15.7           13208.
    ##  7 STL(Employed) 1990 jul.   13170. 13173.       -24.4   22.0           13194.
    ##  8 STL(Employed) 1990 ago.   13160. 13152.       -11.8   19.5           13171.
    ##  9 STL(Employed) 1990 sep.   13113. 13131.       -43.4   25.7           13157.
    ## 10 STL(Employed) 1990 oct.   13185. 13110.        62.5   12.3           13123.
    ## # ... with 347 more rows

Estos comandos llevan a cabo la descomposición de la serie, como se
puede ver en la tabla. La tendencia, `trend` muestra el movimiento de la
serie, sin considerar las fluctuaciones estacionales ni el residuo.
Podemos analizar la tendencia de la serie gráficamente:

``` r
us_retail_employment %>%
  autoplot(Employed, color='gray') +
  autolayer(components(dcmp), trend, color='red') +
  xlab("Year") + ylab("Persons (thousands)") +
  ggtitle("Total employment in US retail")
```

![](03_Descomposicion_ST_files/figure-gfm/employment%20trend-1.jpeg)<!-- -->

Podemos graficar los tres componentes simultáneamente con:

``` r
components(dcmp) %>% autoplot() + xlab("Year")
```

![](03_Descomposicion_ST_files/figure-gfm/STL%20decomp%20plot-1.jpeg)<!-- -->

La gráfica nos indica que se llevó a cabo una descomposición STL, de
forma aditiva, y nos grafica:

1.  La serie original.
2.  La tendencia.
3.  El componente estacional.
4.  El residuo.

Si se suma cada uno de los componentes, obtenemos la serie original.

Las barras grises en estas gráficas indican las escalas relativas de los
componentes.

### Datos desestacionalizados

En ocasiones, los datos presentados por el gobierno u otra organización
se dice que están **desestacionalizados**. Esto significa que le
quitaron el componente estacional a la serie. Si se quita, los datos
ahora están “ajustados estacionalmente”. Para la descomposición aditiva,
los datos destacionalizados están dados por \(y_t - S_t\), mientras que
en la multiplicativa estarían dados por \(\frac{y_t}{S_t}\). Veamos los
datos del empleo, desestacionalizados:

``` r
us_retail_employment %>%
  autoplot(Employed, color='gray') +
  autolayer(components(dcmp), season_adjust, color='blue') +
  xlab("Year") + ylab("Persons (thousands)") +
  ggtitle("Total employment in US retail")
```

![](03_Descomposicion_ST_files/figure-gfm/employment%20seasonally%20adjusted-1.jpeg)<!-- -->

Si la variación debido a la estacionalidad no es de interés en
particular, los datos desestacionalizados pueden ser muy útiles. Un
ejemplo común es el nivel de desempleo o crecimiento económico. El INEGI
reporta las cifras de manera desestacionalizadas. Esto nos permite ver
el estado de la economía, sin tomar en cuenta factores estacionales.

# Medias móviles

La descomposición de series de tiempo clásica utilizaba medias móviles
para definir el componente de tendencia.

Así, una suavización de media móvil de orden \(m\) estaría dada por:

\[
\hat{T}_{t}=\frac{1}{m} \sum_{j=-k}^{k} y_{t+j}
\] en donde \(m = 2k +1\). Entonces, el estimado de de la tendencia en
el periodo \(t\), \(\hat{T}_t\), se obtiene al promediar los valores de
la serie de tiempo dentro de los \(k\) periodos alrededor de \(t\). A
esto se le llama una media móvil de orden \(m\); \(m\)-**MA**.

``` r
global_economy %>%
  filter(Country == "Mexico") %>%
  autoplot(Exports) +
  xlab("Year") + ylab("% of GDP") +
  ggtitle("Total Mexican exports")
```

![](03_Descomposicion_ST_files/figure-gfm/mex%20exports%20plot-1.jpeg)<!-- -->
En la gráfica tenemos las exportaciones de México desde 1960 a 2017,
como porcentaje del PIB. Podemos calcular una media móvil de orden 5;
esto es, obtener el promedio de 5 periodos, para cada momento, \(t\),
con el centro de la “ventana” en \(t\). Así, de acuerdo a la ecuación
presentada, \(k = 2\) y \(m = 2k +1 = 5\).

``` r
mex_exports <- global_economy %>%
  filter(Country == "Mexico") %>%
  mutate(
    `5-MA` = slide_dbl(Exports, mean, .size = 5, .align = "center")
  )
```

Para ver cómo se presenta esta media móvil gráficamente, podemos hacer
lo siguiente:

``` r
gg <- mex_exports %>%
  ggplot(aes(x = Year, y = Exports)) + 
  geom_line() +
  xlab("Year") + ylab("Exports (% of GDP)")
  
gg + geom_line(aes(y = `5-MA`), color='red') +
  ggtitle("Total Mexican exports & 5-MA")
```

![](03_Descomposicion_ST_files/figure-gfm/5-MA%20plot-1.jpeg)<!-- -->

Se puede ver que la tendencia es mucho más suave que los datos
originales, captura el movimiento principal de la serie, pero deja de
lado las fluctuaciones intermedias o menores.

Qué tan suave esté la curva resultante, dependerá del orden de la media
móvil.

``` r
mex_exports <- mex_exports %>%
  mutate(
    `1-MA` = slide_dbl(Exports, mean, .size = 1, .align = "center"),
    `3-MA` = slide_dbl(Exports, mean, .size = 3, .align = "center"),
    `9-MA` = slide_dbl(Exports, mean, .size = 9, .align = "center"),
    `15-MA` = slide_dbl(Exports, mean, .size = 15, .align = "center"),
    `21-MA` = slide_dbl(Exports, mean, .size = 21, .align = "center")
  )

gg <- mex_exports %>%
  ggplot(aes(x = Year, y = Exports)) + 
  geom_line() +
  xlab("Year") + ylab("Exports (% of GDP)")

g1 <- gg +
 geom_line(aes(y = `1-MA`), color='red') +
  ggtitle("1-MA")
g3 <- gg +
 geom_line(aes(y = `3-MA`), color='red') +
  ggtitle("3-MA")
g5 <- gg +
 geom_line(aes(y = `5-MA`), color='red') +
  ggtitle("5-MA")
g9 <- gg +
 geom_line(aes(y = `9-MA`), color='red') +
  ggtitle("9-MA")
g15 <- gg +
 geom_line(aes(y = `15-MA`), color='red') +
  ggtitle("15-MA")
g21 <- gg +
 geom_line(aes(y = `21-MA`), color='red') +
  ggtitle("21-MA")

(g1 | g3 | g5) /
  (g9 | g15 | g21)
```

![](03_Descomposicion_ST_files/figure-gfm/m-MA%20plots-1.jpeg)<!-- -->

Como se puede observar, un modelo **1-MA** realmente no llevaría a cabo
ninguna suavización, y un modelo **21-MA**, en este caso, se convertiría
prácticamente en una línea recta.

### Medias móviles de medias móviles

A una suavización de media móvil se le puede aplicar una nueva
suavización de media móvil. Por ejemplo, considerando la producción de
cerveza, podemos obtener una media móvil de orden 4 y a eso sacarle la
media móvil de orden 2:

``` r
beer <- aus_production %>%
  filter(year(Quarter) >= 1992) %>%
  select(Quarter, Beer)

beer_ma <- beer %>%
  mutate(
    `4-MA` = slide_dbl(Beer, mean, .size = 4, .align = "center-left"),
    `2x4-MA` = slide_dbl(`4-MA`, mean, .size = 2, .align = "center-right")
  )

beer_ma
```

    ## # A tsibble: 74 x 4 [1Q]
    ##    Quarter  Beer `4-MA` `2x4-MA`
    ##      <qtr> <dbl>  <dbl>    <dbl>
    ##  1 1992 Q1   443    NA       NA 
    ##  2 1992 Q2   410   451.      NA 
    ##  3 1992 Q3   420   449.     450 
    ##  4 1992 Q4   532   452.     450.
    ##  5 1993 Q1   433   449      450.
    ##  6 1993 Q2   421   444      446.
    ##  7 1993 Q3   410   448      446 
    ##  8 1993 Q4   512   438      443 
    ##  9 1994 Q1   449   441.     440.
    ## 10 1994 Q2   381   446      444.
    ## # ... with 64 more rows

Al obtener la media móvil de orden 4, **4-MA**, lo que estamos haciendo
es:

\[
\text{4-MA} = \hat{T}_t = \frac{1}{4}\left(y_{t-2}+y_{t-1}+y_{t}+y_{t+1}\right)
\] y, al sacar la media móvil de orden 2 de esta media móvil, **2
\(\times\) 4 - MA**, sería:

\[
2 \times 4 \text{-MA} = \hat{T}_{t} = \frac{1}{2}\left[\frac{1}{4}\left(y_{t-2}+y_{t-1}+y_{t}+y_{t+1}\right)+\frac{1}{4}\left(y_{t-1}+y_{t}+y_{t+1}+y_{t+2}\right)\right]
\] Simplificando, quedaría:

\[
2 \times 4 \text{-MA} = \hat{T}_{t} = \frac{1}{8} y_{t-2}+\frac{1}{4} y_{t-1}+\frac{1}{4} y_{t}+\frac{1}{4} y_{t+1}+\frac{1}{8} y_{t+2}
\] Así, llegamos a ver que la media móvil de una media móvil es
simplemente una **media móvil ponderada**.

### Medias móviles ponderadas

Como acabamos de ver, la combinación de dos o más medias móviles produce
una media móvil ponderada. Esto es, una media móvil que depende en
cierta proporción de cada rezago.

Una media móvil ponderada de orden \(m\) se puede escribir como:

\[
\hat{T}_{t} = \sum_{j=-k}^{k} a_{j} y_{t+j}
\] donde \(k = (m - 1) / 2\) y los pesos o ponderaciones están dados por
\(\left[a_{-k}, \dots, a_{k}\right]\). La suma de los pesos debe sumar
1.

Podemos decir, entonces, que el caso de la media móvil simple \(m\)-MA
es un caso particular de la media móvil ponderada, donde todos sus pesos
son iguales a \(1 / m\).

``` r
us_retail_employment_ma <- us_retail_employment %>%
  mutate(
    `12-MA` = slide_dbl(Employed, mean, .size = 12, .align = "cr"),
    `2x12-MA` = slide_dbl(`12-MA`, mean, .size = 2, .align = "cl")
  )

us_retail_employment_ma %>%
  autoplot(Employed, color='gray') +
  autolayer(us_retail_employment_ma, vars(`2x12-MA`), color='red') +
  xlab("Year") + ylab("Persons (thousands)") +
  ggtitle("Total employment in US retail, 2x12-MA")
```

![](03_Descomposicion_ST_files/figure-gfm/2_12MA-1.jpeg)<!-- -->

\[
T_t = \frac{1}{2}\left(\frac{1}{12} ( y_{t-6} +  ) \right)
\]

# Métodos de descomposición

### Descomposición clásica

Hay dos tipos de descomposición clásica: aditiva y multiplicativa. En
este tipo de descomposición, se asume que el componente estacional es
constante a lo largo del tiempo.

``` r
us_retail_employment %>%
  model(classical_decomposition(Employed, type = "additive")) %>%
  components() %>%
  autoplot() + xlab("Year") +
  ggtitle("Classical additive decomposition of total US retail employment")
```

    ## Warning: Removed 6 row(s) containing missing values (geom_path).

![](03_Descomposicion_ST_files/figure-gfm/classical%20decomp%20-%20additive-1.jpeg)<!-- -->

Hoy en día, ya no se recomienda utilizar el método clásico de
descomposición, ya que existen diversos métodos mejores.

Algunas de las desventajas del método clásico son las siguientes:

  - La estimación del componente de tendencia no está disponible para
    los primeras y últimas observaciones.

  - El componente de tendencia suele suavizar de más incrementos o
    caídas rápidas en los datos.

  - Asume que el componente estacional se repite año con año, por lo que
    no captura cambios en el patrón estacional.

### Descomposición X11

Este método funciona bastante bien con datos trimestrales y mensuales.
Está basado en la descomposición clásica, pero incluye pasos adicionales
para lidiar con los problemas de ella.

Así, X11 logra obtener estimadores para todos los puntos y el componente
estacional puede variar ligeramente con el tiempo. También, cuenta con
mecanismos para lidiar con variaciones por días calendario, efectos de
días feriados, etc.

``` r
x11_dcmp <- us_retail_employment %>%
  model(x11 = feasts:::X11(Employed, type = "additive")) %>%
  components()

autoplot(x11_dcmp) + xlab("Year") +
  ggtitle("Additive X11 decomposition of US retail employment in the US")
```

![](03_Descomposicion_ST_files/figure-gfm/X11%20decomp-1.jpeg)<!-- -->
Podemos utilizar gráficas estacionales o gráficas de sub-series
estacionales para visualizar la variación en el componente estacional a
lo largo del tiempo.

``` r
x11_dcmp %>% 
  gg_season()
```

    ## Plot variable not specified, automatically selected `y = Employed`

![](03_Descomposicion_ST_files/figure-gfm/X11%20season%20&%20subseries%20plot-1.jpeg)<!-- -->

``` r
x11_dcmp %>% 
  gg_subseries(seasonal)
```

![](03_Descomposicion_ST_files/figure-gfm/X11%20season%20&%20subseries%20plot-2.jpeg)<!-- -->

### Descomposición SEATS

“SEATS” son las siglas de “Seasonal Extraction Arima Time Series”. Este
método solo funciona para datos trimestrales o mensuales. Por lo que, si
se cuenta con datos de otra periodicidad, se debe implementar otro
método.

``` r
seats_dcmp <- us_retail_employment %>%
  model(seats = feasts:::SEATS(Employed)) %>%
  components()
autoplot(seats_dcmp) + xlab("Year") +
  ggtitle("SEATS decomposition of total US retail employment")
```

![](03_Descomposicion_ST_files/figure-gfm/SEATS%20decomp-1.jpeg)<!-- -->

### Descomposición STL

STL significa “Seasonal and Trend decomposition using Loess” (“Loess” es
un método de estimación de relaciones no lineales).

STL tiene varias ventajas sobre los otros métodos:

  - Puede tratar con cualquier tipo de estacionalidad, no solo mensual o
    trimestral.

  - El componente estacional puede variar con el tiempo y el usuario
    decide la magnitud del cambio.

  - La suavización del componente de tendencia también es controlado por
    el usuario.

  - Puede ser robusto ante *outliers*, para que observaciones inusuales
    no afecten el componente de tendencia.

Las desventajas de este método son que no controla de manera automática
la variación debido a días hábiles o variaciones por calendario.
También, solo permite hacer descomposiciones aditivas.

``` r
us_retail_employment %>%
  model(STL(Employed ~ trend(window=7) + season(window='periodic'),
    robust = TRUE)) %>%
  components() %>%
  autoplot()
```

![](03_Descomposicion_ST_files/figure-gfm/STL%20decomp-1.jpeg)<!-- -->

Esta gráfica muestra una descomposición mediante STL, ajustando algunos
parámetros (el componente de tendencia es más flexible, el componente
estacional es fijo y se agregó la opción de robustez). Como muestra el
código, los dos parámetros principales a seleccionar al usar STL son la
tendencia, `trend(window = x)` y la estacionalidad, `season(window =
y)`.

Estos parámetros controlan qué tan rápido cambian los componentes de
tendencia y estacional, respectivamente. Valores más bajos provocan
cambios más rápidos. **NOTA:** los valores escogidos de los parámetros
deben ser impares. Si se desea mantener el mismo componente estacional a
lo largo del tiempo, se debería definir como periódico, `season(window =
"periodic")`, como en el caso anterior.

# Tarea

1.  Tomando el PIB de cada país, `GDP`, contenido en la tabla
    `global_economy`, grafique el PIB per cápita a lo largo del tiempo.
    ¿Cómo ha sido la evolución de la economía de los países en el
    tiempo? ¿Cuál país tiene el mayor PIB per cápita?

2.  Grafique las siguientes series de tiempo y transfórmelas y/o
    ajústelas si lo considera necesario. ¿Qué efecto tuvo la
    transformación?
    
    1)  PIB de EEUU, de `global_economy`.
    2)  PIB de México, también de `global_economy`.
    3)  Demanda de electricidad en el estado de Victoria (Australia), de
        `vic_elec`.

3.  ¿Es útil realizar una transformación de Box-Cox a los datos
    `canadian_gas`? ¿Por qué sí o por qué no?

4.  El dataset `fma::plastics` tiene información de las ventas mensuales
    (medidas en miles) del producto A para un productor de plásticos, a
    lo largo de cinco años.
    
    1)  Grafique la serie de tiempo para el producto A. ¿Identifica
        algún componente de tendencia-ciclo y/o estacional?
    2)  Utilice una descomposición clásica multiplicativa para calcular
        el componente de tendencia y estacional.
    3)  ¿Los resultados coinciden con su respuesta al inciso i)?
    4)  Calcule y grafique los datos desestacionalizados.
    5)  Cambie, manualmente, una observación para que sea un *outlier*
        (p. ej., sume 500 a una observación). Vuelva a estimar los datos
        desestacionalizados. ¿Cuál fue el efecto de ese outlier?
    6)  ¿Hace alguna diferencia que el outlier se encuentre cerca del
        final de la serie o más alrededor del centro?
