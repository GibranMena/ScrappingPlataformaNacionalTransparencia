Hackeando Infomex: técnicas de scrapeo de la plataforma de info pública
================

[GibranMenaA](https://twitter.com/GibranMenaA)
Marzo, 2019

¿A qué nos enfrentamos?
-----------------------

![](https://i.ibb.co/3NjPxgF/Captura-de-pantalla-2019-03-01-a-las-10-35-05-p-m.png)

![](https://i.imgflip.com/2v0lrb.jpg)

Todo lo ven pero...
-------------------

La actual plataforma es, por decir lo menos, deficiente. Para consultar 1 año de solicitudes a alguna dependencia necesitarías al menos 7 mil clics, uno para entrar al texto de cada solicitud, y otro para acceder a la respuesta. No hay manera de conocer el contenido de la pregunta sino repasando cada uno de las tortuosas solicitudes

![](https://i.ibb.co/5n69Trv/solicitudesmultiplicadas.png)

Transparentando la intransparencia
==================================

Obteniendo las URLs de las solicitudes y cómo funciona CSS Selector
-------------------------------------------------------------------

Esta es la URL 'real' de la página de la solicitud sobre la dependencia.

<https://www.infomex.org.mx/ConsultaPublicaAPF/GeneraReporte.aspx?strReporte=rptPublicoSI&host=www.infomex.org.mx&fechaCapturaDesdeS=01/01/2018%2000:00:00&fechaCapturaHastaS=01/03/2019%2023:59:59&selectEstatusS=5&selectTipoRespuestaS=6&selectDependencia=16>

¿Cómo lo sé? ¿Gracias al inspector de tu explorador web.

![](https://i.ibb.co/2kZFFTW/Captura-de-pantalla-2019-03-01-a-las-6-53-42-p-m.png)

Así se ve el inspector ya en acción

![](https://i.ibb.co/T8jz69J/Captura-de-pantalla-2019-03-01-a-las-6-55-13-p-m.png)

Al copiar y pegar la URL verdadera en otra pestaña, y abrir de nuevo el inspector del explorador, podemos encontrar esta la URL real de cada respuesta a una solicitud, donde está el texto relevante.

![](https://i.ibb.co/7bvCRd3/textorelevante.jpg)

<https://www.infomex.org.mx/ConsultaPublicaAPF/GeneraReporte.aspx?strReporte=rptPublicoSI&host=www.infomex.org.mx&fechaCapturaDesdeS=01/01/2018%2000:00:00&fechaCapturaHastaS=01/03/2019%2023:59:59&selectEstatusS=5&selectTipoRespuestaS=6&selectDependencia=16>

¿Qué cambia en cada página que me muestra el texto?

Exacto, el folio.

La página parece estar diseñada para complicar el acceso a los folios, por lo que tomamos un atajo con lo que sí permite la plataforma, que es la descarga de todas las páginas donde se encuentran los folios.

![](https://i.ibb.co/XDYh6rW/Captura-de-pantalla-2019-03-01-a-las-6-40-01-p-m.png)

Al descargar, nos econtramos con oootro obstáculo para el análisis de los datos: los logotipos en una matriz... ¡no son datos! Podemos retirar las primeras imágenes con cualquier hoja de cálculo, como LibreOffice.org

![](https://i.ibb.co/6tQ2DqK/logos.png)

Ahora viene la diversión, vamos a RStudio o el IDE que ustedes prefieran.
-------------------------------------------------------------------------

Instalación de paquetes
-----------------------

Comencemos con abrir un nuevo **proyecto** desde el menú Archivo de Rstudio. Es importante que sea un proyecto, y no simplemente un script, porque estos nos permitirá almacenar todas nuestra información en una carpeta y acceder a archivos gracias al paquete **here**.

Simplemente elijamos dónde queremos que cree Rstudio el proyecto y demos clic en Aceptar.

Los paquetes son condensaciones de funciones que la comunidad de R contribuye para tareas específicas de procesamiento, limpieza, análisis, visualización. En este caso, el paquete central para nuestra tarea de *scrapping* es **RVest**.

``` r
install.packages("rio") #Para importar varias hojas de un archivo xls
install.packages("rvest") #Para leer los htmls y hacer scrapping
install.packages("magrittr") #Para hacer conciso el código
install.packages("tibble") #Para hacer planas las listas y de hecho leerlas
install.packages("here") #Para la vida: organizar proyectos
install.packages("purrr") #Para hacer planas las listas y de hecho leerlas
install.packages("stringr") #Para detectar y eliminar registros por string
install.packages("dplyr") #Para convertir a matrices
install.packages("tidyr") #Para quitar los NAs
```

'Prender' los paquetes
----------------------

A pesar de estar instalados, es necesario llamar o activar los paquetes que requeriremos en nuestra sesión

``` r
library("rio") #Para importar varias hojas de un archivo xls
library("rvest") #Para leer los htmls y hacer scrapping
library("magrittr") #Para usar los pipes o signos %%>%% para hacer más conciso el código
library("here") #para no incorporar rutas todo el tiempo
library("purrr") #Para hacer planas las listas y de hecho leerlas
library("stringr") #Para detectar y eliminar registros por string
library("dplyr") #Para convertir a matrices
library("tidyr") #Para quitar los NAs
```

¿Cómo funciona Rvest?
---------------------

Permite obtener los contenidos estructurados de páginas web a través de 'etiquetas' de contenidos conocidas como selectores CSS

Por ejemplo, para el sitio de nuestra solicitud pública:

``` r
url <- "https://www.infomex.org.mx/gobiernofederal/moduloPublico/verDetalleSolSinRespP.action?idFolioSol=0001600003418&idTipoResp=0"
html <- read_html(url)
```

Aquí debemos encontrar el Selector CSS
--------------------------------------

Añadimos el link al final de este sitio <https://selectorgadget.com/> a los marcadores, en navegadores como Firefox, o añadir la extensión que hay para navegadores como Chrome.

![](https://i.ibb.co/q5dhqLT/Captura-de-pantalla-2019-03-01-a-las-11-02-53-p-m.png)

Vamos a la página del texto que buscamos, damos clic en el cuadro una vez, y verificamos que nada más quede seleccionado. Si algo que no deseamos queda seleccionado, con un clic en esa área pordemos retirarlo. SelectorGadget nos dará el selector css asociado con ese contenido.

![](https://i.ibb.co/Wg90rGF/Captura-de-pantalla-2019-03-01-a-las-9-18-21-p-m.png) Ahora podemos extraer nuestro texto en esta url de una sola solicitud.

``` r
nodos <- html_nodes(html, "#idComenInexInf")
texto <- html_text(nodos)

# O, sintetizado

texto2 <- html %>% html_nodes("#idComenInexInf") %>% html_text(.)
```

¡Listo! Ahora sólo debemos replicar para miles de folios XD.
------------------------------------------------------------

Cuál es la diferencia entre <https://www.infomex.org.mx/gobiernofederal/moduloPublico/verDetalleSolSinRespP.action?idFolioSol=0001600003418> y
<https://www.infomex.org.mx/gobiernofederal/moduloPublico/verDetalleSolSinRespP.action?idFolioSol=0001600023518>

Exaaacto, el folio de la solicitud, creemos una lista con los 4 mil + folios
----------------------------------------------------------------------------

Este es el archivo que descargamos de la plataforma (y que colocamos en la misma carpeta que nuestro proyecto de R. Gracias a import\_list, ¡podemos leer al mismo tiempo cada una de las 147 pestañas del archivo!

``` r
data_list <- import_list("semarnatsol.xls", setclass = "tbl", rbind = T) #Este archivo es el que tiene cambiar cada vez
data_list <- as_tibble(data_list$X__1) #Dejar sólo los folios
data_list <- drop_na(data_list) #Eliminar NAs
folios <- data_list %>% filter(!(str_detect(value, "Folio"))) #filtrar y dejar fuera los registros "Folio"

#Vamos a crear los urls

url_lista <- vector("list", length(folios)) # crear objeto con la longitud de ids

for (i in seq_along(folios)) { 
  url_lista[[i]] <- paste0("https://www.infomex.org.mx/gobiernofederal/moduloPublico/verDetalleSolSinRespP.action?idFolioSol=", folios[[i]])
}
```

Acortemos porque el tiempo es oro
=================================

Para este ejercicio, de nuestra lista de más de 4 mil urls, dejemos los primeros 100, para ahorrar tiempo.

``` r
url_lista <- flatten(url_lista)
url_lista <- url_lista[1:100]
```

Ahora sí, a hacer scrapping de solicitudes
------------------------------------------

``` r
Scraplist <- (vector("list", length(url_lista))) #Crea una lista de la longitud de folios
htmlscrap <- (vector("list", length(url_lista)))


for (i in url_lista) {
  htmlscrap[[i]] <- read_html(i)
  Scraplist[[i]] <- htmlscrap[[i]] %>% html_nodes("#idComenInexInf") %>% html_text(.)
  cat("*")
}

coso <- as_data_frame(t(rbind(Scraplist)))  # Esto se tiene que cambiar
coso2 <- as_data_frame(coso, rownames = "ids")
coso2 <- coso2[101:200,]
coso2$Scraplist <- as.character(coso2$Scraplist)
write.csv(coso2, "infomex_buscable.csv")
```

Finalmente, podemos abrir el documento csv que generamos con un editor de texto, **no con una hoja de cálculo**. Tendremos un documento con el que perfectamente podemos buscar nuestros temas. ¡Gracias por nada, sistema de consulta de solicitudes públicas de la PNT! :D
