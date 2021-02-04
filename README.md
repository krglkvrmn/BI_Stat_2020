# Проект №1

Проект выполнен в рамках курса  "Cтатистика и анализ данных в R" Института биоинформатики.

## Описание проекта

В проекте анализируются данные по размерам, весу, полу и возрасту моллюсков, которые были собраны различными людьми в различное время.

В ходе проекта планировалось:

* Объединить данные
* Предобработать данные
* Произвести EDA
* Выдвигать и проверять простые статистические гипотезы
* Визуализировать данные

Детальный ход работы, отражён в файле **Report.rmd**.

# Подготовка данных

Данные исследования представлены в виде 11 файлов в формате csv. Для
более удобной обработки и анализа этих данных, они были объединены в
единую таблицу с использованием пользовательской функции
*concat\_csv\_data*.

``` r
concat_csv_data <- function(path) {
  setwd(path)
  accumulated_data <- data.frame()
  researcherId <- 0
  for (filename in list.files()) {
    if (endsWith(filename, ".csv")) {
      input_df <- read.csv(filename)
      input_df$Researcher <- researcherId
      accumulated_data <- rbind(accumulated_data, input_df)
      researcherId = researcherId + 1
    }
  }
  return(accumulated_data)
}
```

Данная функция принимает на вход абсолютный путь до папки, содержащей
данные, и возвращает таблицу, включающую в себя наблюдения из всех
файлов с заданным расширением.

# Предварительный анализ данных

## Проверка данных на корректность

В данных было приведено неудобное для использования название переменной,
обозначающей пол моллюска, кроме того все загруженные данные были
представлены строковым типом. Данные были преобразованы соответствующим
образом, дополнительно были введены метки для уровней фактора переменной
*Sex*.

``` r
colnames(mollusk_data)[2] <- "Sex"
mollusk_data <- mollusk_data %>%
  transmute(across(everything(), as.numeric))
mollusk_data$Sex <- factor(mollusk_data$Sex,
                           levels = c(1, 2, 3),
                           labels = c("Самец", "Самка", "Ювенильный"))
```

Нам до конца не известно, какой моделью можно описать наши данные,
поэтому применение интерполяции для заполнения пропущенных значений
может привести к ошибкам. Возможно было также заполнить пропущенные
значения средними значениями, что тоже требует глубоких знаний об
объекте изучения. Поэтому во избежание получения ложных результатов,
наблюдения, содержащие пропущенные значения, были удалены.

## Проверка данных на наличие выбросов

Разброс значений количественных переменных, сгруппированных по
переменной *Sex*, представлен на [графике](#fig1).

<a id="fig1"></a> ![Рисунок 1. Распределение количественных переменых в
зависимости от пола
моллюска](Report_for_md_files/figure-gfm/unnamed-chunk-6-1.png)

Как видно из [графиков](#fig1), практически все переменные имеют
значительное число выбросов. Для удаления выбросов была использована
пользовательская функция *outliers.rm*.

``` r
outliers.rm <- function(x) {
  q1 <- quantile(x, 0.25, na.rm = T)
  q3 <- quantile(x, 0.75, na.rm = T)
  iqr <- IQR(x, na.rm = T)
  return(x>q1-1.5*iqr & x<q3+1.5*iqr)
}
```

В данном случае было принято решение удалить все наблюдения с выбросами,
однако стоит заметить, что можно было не удалять наблюдения с выбросами
в переменной *Rings*, так как большее число старых особей в целом
является нормальной закономерностью в природных популяциях.

## Оценка взаимосвязей и формулировка гипотез

[Корреляционная матрица](#fig2), приведённая ниже, позволяет оценить
степень взаимосвязи между переменными. <a id="fig2"></a> ![Рисунок 2.
Корреляционная матрица, отражающая взаимосвязь количественных
переменных.](Report_for_md_files/figure-gfm/unnamed-chunk-9-1.png)

А данные [графики](#fig3) позволяют сравнить средние значения для
различных переменных в завиимости от пола. <a id="fig3"></a> ![Рисунок
3. Распределение количественных переменых в зависимости от пола моллюска
(данные с удалёнными
выбросами)](Report_for_md_files/figure-gfm/unnamed-chunk-10-1.png)

Из [графиков](#fig2) можно заметить, что количественные переменные имеют
положительную взаимосвязь, а размеры самок немного больше размеров
самцов. Сформулируем следующие гипотезы:<br>
<ul>
<li>
Линейные размеры моллюсков не различаются у особей взрослых и ювенильных
особей.
<li>
Линейные размеры моллюсков не различаются у женских и мужских особей.
<li>
Существует линейная зависимость между размерами моллюска и его весом.
</ul>

# Углубленный анализ данных

## Описание количественных переменных

В приведённой ниже [таблице](#table1) отражены средние и стандартные
отклонения длин моллюсков разных полов. <a id="table1"></a>

    ## # A tibble: 3 x 4
    ##   Sex         Mean    Sd     N
    ## * <fct>      <dbl> <dbl> <dbl>
    ## 1 Самец      0.555 0.097  1353
    ## 2 Самка      0.574 0.085  1150
    ## 3 Ювенильный 0.436 0.097  1254

Из данных выяснилось, что у **79.1** процентов моллюсков значение
переменной *Height* не превышает **0.165**.

Значение переменной *Length* равное **0.66**, больше, чем у 92% от всех
наблюдений.

Переменная *Length* была стандартизована следующим образом.

``` r
Length_z_scores <- scale(mollusk_data$Length)
```

## Проверка статистических гипотез

<a id="fig4"></a> ![Рисунок 3. Распределение диаметров моллюсков с 5 и
15 кольцами](Report_for_md_files/figure-gfm/unnamed-chunk-13-1.png)

Из [графика](#fig4) видно, что моллюски с 15 кольцами имеют больший
диаметр, чем моллюски с 5 кольцами. Перед проверкой гипотезы о равенстве
средних значений для данных выборок необходимо определить подчиняются ли
распределения нормальному закону.

Здесь и далее в статистических тестах будет использоваться Р-уровень
значимости равный 0.95.

В [таблице](#table2) приведены результаты проверки распределения на
нормальность при помощи критерия Шапиро-Уилка. <a id="table2"></a>

    ## # A tibble: 2 x 6
    ##   Rings W_statistic P_value  Mean    Sd     N
    ## * <dbl>       <dbl>   <dbl> <dbl> <dbl> <int>
    ## 1     5       0.945   0      0.22  0.04   100
    ## 2    15       0.983   0.197  0.46  0.06   102

По данным теста распределение диаметров у моллюсков с 15 кольцами
значимо не отличается от нормального, а у моллюсков с 5 кольцами значимо
отличается от нормального. Следовательно для сравнения средних у данных
групп следует использовать непараметрический критерий Уилкоксона. Для
этого сформулируем альтернативную гипотезу: **Средний диаметр у
моллюсков с 5 кольцами отличается от среднего диаметра у моллюсков с 15
кольцами**.

Результаты приведены в [таблице](#table3): <a id="table3"></a>

    ## # A tibble: 1 x 9
    ##   W_statistic P_value  Altern.hypotesis Mean_5 Mean_15  Sd_5 Sd_15   N_5  N_15
    ##         <dbl> <chr>    <chr>             <dbl>   <dbl> <dbl> <dbl> <int> <int>
    ## 1           1 1.21e-34 two.sided         0.222   0.455  0.04 0.058   100   102

P-значение равно **1.21e-34**, что позволяет отвергнуть нулевую гипотезу
о равенстве средних.

Для изучения взаимосвязи переменных *Diameter* и *Whole\_weight*
необходимо сперва [визуализировать данные](#fig5). <a id="fig5"></a>
![Рисунок 5. Взаимосвязь веса и диаметра
моллюска](Report_for_md_files/figure-gfm/unnamed-chunk-17-1.png)

Из [графика](#fig5) видно, что с увеличением диаметра моллюсков
увеличивается и их вес. При этом при значениях диаметра больше **0.35**
более выражена возможная линейная взаимосвязь переменных.

Перед установлением факта наличия корреляции проверим выборки на
нормальность. Результаты теста Шапиро-Уилка приведены в
[таблице](#table4). <a id="table4"></a>

    ## # A tibble: 2 x 6
    ##   Variable     W_statistic P_value   Mean    Sd     N
    ##   <chr>              <dbl> <chr>    <dbl> <dbl> <dbl>
    ## 1 Diameter           0.971 5.54e-27 0.405 0.092  3757
    ## 2 Whole_weight       0.973 6.84e-26 0.793 0.445  3757

Видим, что распределения веса и диаметра моллюсков значимо отличаются от
нормального. Это означает, что для расчёта корреляции необходимо
воспользоваться критерием Спирмена.

Результаты приведены в [таблице](#table5): <a id="table5"></a>

    ## # A tibble: 1 x 4
    ##   S_statistic P_value   Rho Method                         
    ##         <dbl>   <dbl> <dbl> <chr>                          
    ## 1  260945824.       0 0.970 Spearman's rank correlation rho

Р-значение намного меньше 0.05, значит гипотеза о равенстве нулю
коэффициента коррелляции ложна. Значение **r = 0.97** позволяет сделать
вывод о сильной взаимосвязи данных переменных.

# Дополнительная часть

## Человеческий фактор

### Различия в замерах

В этой части будут рассмотрены различия в измерениях, проводимых
различными людьми. Для этого изначально были также загружены данные об
исследователях, проводивших замеры. Исследователи были анонимизированы,
в данных они представлены порядковыми номерами.

Для начала рассмотрим [распределение](#fig6) количественных переменных в
зависимости от исследователя, проводившего замеры. <a id="fig6"></a>
![Рисунок 6. Распределение количественных переменых в зависимости от
исследователя, проводившего
замеры](Report_for_md_files/figure-gfm/unnamed-chunk-21-1.png)

По [графикам](#fig7) видно, что распределения величин количественных
переменных среди исследователей очень схожи. Однако можно заметить, что
медианы распределений у исследователя №3 меньше, чем у других, а медианы
распределений исследователя №9 больше.<br> Выдвинем альтернативную
гипотезу о том, что **средние величины, измеренные исследователем №3
отличаются от средних величин измеренных исследователем №9**. Проверим
эту гипотезу при помощи t-теста для различных переменных.

Для начала проверим распределения на нормальность. В приведённой ниже
[таблице](#table6) представлены результаты теста на нормальность
Шапиро-Уилка.<br> P-значения приведены в ячейках таблицы для
соответствующих исследователей и переменных. <a id="table6"></a>

    ## # A tibble: 11 x 8
    ##    Researcher Length Diameter Height Whole_weight Shucked_weight Viscera_weight
    ##  *      <dbl> <chr>  <chr>    <chr>  <chr>        <chr>          <chr>         
    ##  1          0 8.44e… 1.26e-05 0.0277 1.57e-05     5.72e-06       1.82e-05      
    ##  2          1 8.74e… 4.34e-06 0.247  4.5e-05      4.74e-05       1.03e-05      
    ##  3          2 4.55e… 1.43e-07 0.005… 1.66e-08     3.97e-10       4.7e-09       
    ##  4          3 0.0126 0.00621  0.735  0.00179      0.000298       0.000252      
    ##  5          4 0.003… 0.00153  0.0137 0.0292       0.0157         0.0935        
    ##  6          5 0.000… 1.2e-05  0.224  0.0016       0.000763       0.00029       
    ##  7          6 2.94e… 4.6e-06  0.002… 0.000119     1.35e-05       3.21e-05      
    ##  8          7 5.22e… 1.46e-05 0.001… 1.19e-07     9.06e-09       8.24e-09      
    ##  9          8 4.91e… 4.15e-10 2.1e-… 3.4e-08      4.17e-08       1.34e-08      
    ## 10          9 1.29e… 4.16e-11 0.000… 4.19e-08     9.62e-09       8.12e-09      
    ## 11         10 5.68e… 7.89e-09 0.001… 1.44e-10     6.19e-12       9.89e-12      
    ## # … with 1 more variable: Shell_weight <chr>

Из [таблицы](#table6) видно, что подавляющая часть распределений не
являются нормальными, а значит для проверки гипотез воспользуемся
непараметрическим критерием Уилкоксона. Результаты тестов представлены в
[таблице](#table7).<br> Количество наблюдений исследователя №3 составило
**118**<br> Количество наблюдений исследователя №9 составило **483**
<a id="table7"></a>

    ## # A tibble: 8 x 7
    ##   Variable       W_statistic P_value Mean_3 Mean_9  Sd_3  Sd_9
    ##   <chr>                <dbl> <chr>    <dbl>  <dbl> <dbl> <dbl>
    ## 1 Rings                26810 0.314    9.24   9.49  2.42  2.27 
    ## 2 Length               25294 0.0582   0.503  0.525 0.116 0.113
    ## 3 Diameter             25576 0.0841   0.392  0.408 0.096 0.093
    ## 4 Height               24760 0.027    0.131  0.139 0.034 0.035
    ## 5 Whole_weight         25062 0.0422   0.72   0.814 0.43  0.453
    ## 6 Shucked_weight       25777 0.108    0.32   0.354 0.2   0.207
    ## 7 Viscera_weight       24704 0.0249   0.156  0.18  0.1   0.103
    ## 8 Shell_weight         25163 0.0487   0.208  0.233 0.119 0.126

Из результатов видно, что нулевую гипотезу о равенстве средних для
замеров у исследователей №3 и №9 можно отклонить для высоты моллюска,
общего веса, веса внутренностей и веса раковины. Возможно, исследователь
№3 предпочитал выбирать более плоских моллюсков, в то время как
исследователь №9 выбирал “более сферических”.

### Кто делает больше выбросов?

Интересно также посмотреть на зависимость количества выбросов от
исследователя. На нижележащей [таблице](#table8) представлены количества
выбросов среди замеров у каждого из исследователей, а также общее
количество замеров для каждого исследователя. <a id="table8"></a>

    ##    Researcher outliers   N
    ## 1           0       42 335
    ## 2           1       39 280
    ## 3           2       95 551
    ## 4           3       12 127
    ## 5           4       25 103
    ## 6           5       19 183
    ## 7           6       55 334
    ## 8           7       80 492
    ## 9           8       57 540
    ## 10          9       65 525
    ## 11         10       91 682

Однако, такие результаты трудно интерпретировать. Для большей
наглядности, нормируем количество выбросов на количество наблюдений.
<a id="table9"></a>

    ## # A tibble: 11 x 2
    ##    Researcher `% Outliers`
    ##         <dbl>        <dbl>
    ##  1          0        12.5 
    ##  2          1        13.9 
    ##  3          2        17.2 
    ##  4          3         9.45
    ##  5          4        24.3 
    ##  6          5        10.4 
    ##  7          6        16.5 
    ##  8          7        16.3 
    ##  9          8        10.6 
    ## 10          9        12.4 
    ## 11         10        13.3

По данным результатам трудно сделать какие-либо выводы, однако можно
заметить, что иследователь №3 допускал выбросы реже остальных, а
исследователь №4 чаще.

## Возрастной состав популяции

В ходе анализа данных появился закономерный вопрос. Моллюски какого пола
преобладают в разных возрастных группах?

Более-менее очевидно, что среди молодых моллюсков будут преобладать
ювенильные особи, однако как изменится распределение среди моллюсков
постарше?

Для ответа на данный вопрос построим [график](#fig7), отражающий число
моллюсков каждого пола в каждой возрастной группе. <a id="fig7"></a>
![Рисунок 7. Возрастной и половой состав популяции
моллюсков](Report_for_md_files/figure-gfm/unnamed-chunk-26-1.png)

На данном [графике](#fig7) можно заметить несколько интересных
закономерностей.

Во-первых можно заметить, что соотношение количества особей разных полов
действительно меняется в зависимости от возраста. Среди младших особей
преобладают ювенильные, среди более старших становится больше мужских
особей, далее с течением времени резко увеличивается количество женских
особей.

Во-вторых число самцов с возрастом резко снижается, что может
свидетельствовать о их меньшей средней продолжительности жизни.

Назовём “старыми” особей с количеством колец больше 10, а “молодыми”
остальных.

Сформулируем следующие альтернативные гипотезы:
<ul>
<li>
**Средний возраст отличается от среднего возраста самцов**
<li>
**Средний возраст старых самцов не равен среднему возрасту старых
самок.**
</ul>

Сначала проверим распределения на нормальность. Результаты теста
Шапиро-Уилка представлены в [таблице](#table10). <a id="table10"></a>

    ## # A tibble: 4 x 7
    ##   Sex   Age    W_statistic  P_value  Mean    Sd     N
    ##   <fct> <chr>        <dbl>    <dbl> <dbl> <dbl> <int>
    ## 1 Самец Все          0.963 5.81e-18  10.1  2.09  1353
    ## 2 Самка Все          0.965 4.09e-16  10.4  2.00  1150
    ## 3 Самец Старые       0.821 2.94e-23  12.3  1.36   502
    ## 4 Самка Старые       0.833 2.50e-22  12.2  1.30   494

Все распределения значимо отличаются от нормального. Для проверки
гипотез воспользуемся непараметрическим критерием Уилкоксона.

Результаты t-теста представлены в [таблице](#table11).
<a id="table11"></a>

    ## # A tibble: 2 x 5
    ##   Age   W_statistic P_value Altern.hypotesis     N
    ##   <chr>       <dbl>   <dbl> <chr>            <int>
    ## 1 All        841327  0.0004 two.sided         2503
    ## 2 Old        124430  0.920  two.sided          996

Итак, средний возраст самцов значимо отличается от среднего возраста
самок, но возрасты старых особей значимо не различаются. Это говорит нам
о том, что различия обусловлены молодыми особями. Это также было заметно
из последнего [графика](#fig7): молодых самцов больше, чем молодых
самок. Эта гипотеза была подтверждена в ходе одностороннего t-теста
(данные не приведены).

Получается, что с мужские особи в популяции в целом начинают появлятся
раньше, чем женские. Это может быть обусловленно биологическими
особенностями детерминации пола у данного вида моллюсков, т.е. развитие
по мужскому пути, по-видимому, происходит быстрее.

## Половой диморфизм

Посмотрим снова на [графики](#fig3) зависиимости размеров и веса
моллюсков от пола. Заметим, что значения количественных переменных,
отвечающих за размеры моллюска и его вес принимают более большие
значения для женских особей, чем для мужских.

Эти различия более наглядны на графике зависимости от возраста. Построим
подобный [график](#fig8) на примере переменной *Whole\_weight*.

<a id="fig8"></a> ![Рисунок 8. Общий вес моллюсков в зависимости от
возраста и пола](Report_for_md_files/figure-gfm/unnamed-chunk-29-1.png)

Сформулируем следующую гипотезу: **Средние значения количественных
переменных для женских особей отличаются от средних значений
количественных переменных для мужских особей** и проверим эту гипотезу
при помощи t-теста для различных переменных размеров и веса.

Проверим распределения на нормальность при помощи критерия Шапиро-Уилка.

Результаты теста приведены в [таблице](#table12). <a id="table12"></a>

    ## # A tibble: 2 x 8
    ##   Sex   Length Diameter Height Whole_weight Shucked_weight Viscera_weight
    ## * <fct> <chr>  <chr>    <chr>  <chr>        <chr>          <chr>         
    ## 1 Самец 1.46e… 3.45e-21 7.31e… 2.69e-06     4.61e-08       4.09e-07      
    ## 2 Самка 1.2e-… 6.65e-14 0.000… 1.55e-05     1.39e-07       6.2e-06       
    ## # … with 1 more variable: Shell_weight <chr>

Как видно, все распределения значимо оличаются от нормального, поэтому
для проверки гипотезы снова воспользуемся критерием Уилкоксона.

Результаты теста представлены в [таблице](#table13).
<a id="table13"></a>

    ## # A tibble: 7 x 9
    ##   Variable W_statistic P_value Mean_Male Mean_Female Sd_Male Sd_Female N_Male
    ##   <chr>          <dbl> <chr>       <dbl>       <dbl>   <dbl>     <dbl>  <int>
    ## 1 Length       700578. 1.74e-…     0.555       0.574   0.097     0.085   1353
    ## 2 Diameter     697772. 8.49e-…     0.434       0.45    0.08      0.07    1353
    ## 3 Height       692166. 1.85e-…     0.148       0.155   0.031     0.028   1353
    ## 4 Whole_w…     708512. 0.0001…     0.939       1.01    0.415     0.397   1353
    ## 5 Shucked…     731080. 0.00925     0.415       0.436   0.198     0.186   1353
    ## 6 Viscera…     697292. 7.54e-…     0.206       0.224   0.095     0.092   1353
    ## 7 Shell_w…     698036. 9.14e-…     0.265       0.286   0.113     0.11    1353
    ## # … with 1 more variable: N_Female <int>

Мы можем отвергнуть нулевую гипотезу о равенстве средних значений
размеров и веса для особей мужского и женского пола для всех переменных.
Это означает, что размеры и вес женских особей значимо отличаются от
размеров и веса мужских особей. Возможно, что для данного вида моллюсков
типичен половой диморфизм, что означает различие в анатомическом
строении мужских и женских особей. В данном случае самки в среднем
немного крупнее самцов.