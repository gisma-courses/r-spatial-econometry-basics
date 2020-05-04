---
title: "Sitzung 2: -  Region, Distanz und räumlicher Einfluß"
author: "Chris Reudenbach"
date: "1 Mai 2020"
output: 
  html_document: 
    fig_caption: yes
    fig_height: 6.5
    fig_width: 10
    keep_md: yes
editor_options: 
  chunk_output_type: console
---



Geodaten sind prinzipiell wie gewöhnliche Daten. Allerdings sind die Aspekte, die bei der Verwendung von Geodaten zu beachten sind auf Fragen der Skala, der Zonierung (aggregierte  Flächeneinheiten) der Topologie (der Lage im Verhältnis zu anderen Entitäten) und am einfachsten der Entfernung zueinander. <!--more-->

Dies berührt fragen der der räumlichen Autokorrelation und Inhomogenität, also letztlich der Konzeption auf welcher Skala hat Raum einen Einfluss.

Die klassischen Bereiche der räumlichen Statistik sind Punktmusteranalyse, Regression und Inferenz mit räumlichen Daten, dann die Geostatistik (Interpolation z.B. mit Kriging) sowie  Methoden zur lokalen und globalen Regression und Klassifikation mit räumlichen Daten.


## Lernziele

Die Lernziele der zweiten Übung sind:

---

  * Verständnis für die konzeptionellen Hintergründe der Begriffe Region, Distanz und räumlicher Einfluss bezogen auf die jeweilig verfügbaren Datenmodelle
  * 

 
---


## Einrichten der Umgebung



```r
rootDIR="~/Schreibtisch/spatialstatSoSe2020/"


rm(list=ls())

## laden der benötigten libraries
# wir definieren zuerst eine liste mit den Paketnamen und 
# nutzen dann eine for  schleife die jedes element aus der  liste nimmt 
# und schaut ob es bereits installiert ist utils::installed.packages() 
# falls nicht wird es installiert 
libs= c("sf","mapview","tmap","ggplot2","RColorBrewer","jsonlite","tidyverse","spdep","spatialreg","ineq","rnaturalearth", "rnaturalearthhires", "tidygeocoder","usedist","raster","kableExtra")
for (lib in libs){
  if(!lib %in% utils::installed.packages()){
    utils::install.packages(lib)
  }}
# nicht wundern lapply()ist eine integrierte for Schleife die alle im vector libs
# enthaltenen packages lädt indem sie den package Namen als character string an die 
# function library übergibt
invisible(lapply(libs, library, character.only = TRUE))
```




## Regionalisierung oder Aggregationsräume

Die Analyse der räumlichen Daten erfolgt oft in regionaler oder Aggregierung. Auch wenn alle am Liebsten Daten auf einer möglichst hoch aufgelösten Ebene verfügbar hätten (im besten Fall Einzelpersonen, Haushalte, Grundstücke, Briefkästen etc.) ist der Regelfall, dass es sich um räumlich (und zeitlich) aggregierte  Daten handelt. Statt tägliche Daten über den Cornfllakes-Kosum in jeden Haushalt haben wir den  Jahresmittelwert aller verkauften Zerealien in einem Bundesland. So geht das mit den meisten Daten, die zudem oft unterschiedlich aggregiert sind wo in Europa z.B. nationale und subnationale Einheiten (z.B. NUTS1, NUTS3, NUTS3, AMR etc.) vorliegen. Häufig gibt es auch räumliche Datensätze die in Form von Rasterzellenwerten quasi-kontinuierlich vorliegen.

Bei der visuellen Exploration aber auch bei der statistischen Analyse ist es von erheblichem Einfluss wie die Gebiete zur Aggregation der Daten geschnitten sind. Da dieser Zusammenhang eher willkürlich (auch oft historisch oder durch wissenschaftlich begründet) ist, sind die Muster, die wir sehen äußerst subjektiv. Dieses sogenannte *Problem der veränderbaren Gebietseinheit* (Modifiable Areal Unit Problem, MAUP) bezeichnet. Der Effekt von höheren Einheiten auf niedrigere zu schließen ist hingegen als *ökologische Inferenz* (Ecological Inference) bekannt.


Betrachten wir diese Zusammenhänge einmal ganz praktisch mit unserem Datensatz.

Um den Zusammenhang von Zonierung und Aggregation grundsätzlich zu verstehen erzeugen wir einen synthesischen Datensatz, der eine *Region* (angenommen wird die gesamte Ausdehnung Deutschlands) mit 10000 *Haushalten* enthält. Für jeden Haushalt ist der Ort und sein Jahreseinkommen bekannt. In einem nächsten Schritt werden die Daten in unterschiedliche Zonen aggregiert. Zunächst werden die `nuts3_kreise` eingeladen. Sie dienen der Georefrenzierung der Beispiel-Daten. Zunächst wir ein Datensatz von 10k zufällig über dem Gebiet Deutschlands verteilter Koordinaten erzeugt. Diesem werden dann zufällige Einkommensdaten zu-gewürfelt.



```r
rootDIR="~/Schreibtisch/spatialstatSoSe2020/"
# einlesen der nuts3_kreise 
nuts3_kreise = readRDS(file.path(rootDIR,"nuts3_kreise.rds"))

# für die Reproduzierbarkeit der Ergebnisse muss ein beliebiger `seed` gesetzt werden
set.seed(0) 


# Normalverteilte Erzeugung von zufälligen der Koordinatenpaaren
# in der Spannweite  der Ausdehnung der nuts3_kreise Daten
xy <- cbind(x=runif(10000, extent(nuts3_kreise)[1], extent(nuts3_kreise)[3]), y=runif(10000, extent(nuts3_kreise)[2], extent(nuts3_kreise)[4]))

# Normalverteilte Erzeugung Einkommensdaten
income <- (runif(10000) * abs((xy[,1] - (extent(nuts3_kreise)[1] - extent(nuts3_kreise)[3])/2) * (xy[,2] - (extent(nuts3_kreise)[2] - extent(nuts3_kreise)[4])/2))) / 500000000
```

Nachdem die Daten erzeugt wurden, schauen wir und die akkumulierte, klassifizierte und räumliche Verteilung der Daten an.


```r
# Festlegen der Grafik-Ausgabe
par(mfrow=c(1,3), las=1)
# Plot der sortieren Einkommen
plot(sort(income), col=rev(terrain.colors(500)), pch=20, cex=.75, ylab='income')

# Histogramm der Einkommensverteilung 
hist(income, main='', col=rev(terrain.colors(10)),  xlim=c(0,150000), breaks=seq(0,150000,10000))

# Räumlicher Plot der Haushalte, Farbe und Größe markieren das Einkommen
plot(xy, xlim=c(extent(nuts3_kreise)[1], extent(nuts3_kreise)[3]), ylim=c(extent(nuts3_kreise)[2], extent(nuts3_kreise)[4]), cex=income/100000, col=rev(terrain.colors(50))[(income+1)/1200], xlab="Rechtwert",ylab="Hochwert" )
```

<img src="{{ site.baseurl }}/assets/images/unit04/zone_result-1.png" width="1000px" height="550px" />

*Abbildung 04-02-01: Die sortierte, aggregierte und räumliche Einkommensverteilung*


Gini-Koeffizient und Lorenz-Kurve sind eine brauchbare Einschätzung um die Ungleichheit, bzw. relative Konzentrationen oder Ungleichheit von Verteilungen zu zeigen.


```r
# Berechnung Gini Koeffizient
ineq(income,type="Gini")
```

```
## [1] 0.3993752
```

```r
## [1] 0.3993752

# Plot der Lorenz Kurve
par(mfrow=c(1,1), las=1)
plot(Lc(income),col="darkred",lwd=2)
```

<img src="{{ site.baseurl }}/assets/images/unit04/lorenz-1.png" width="400px" height="400px" />

*Abbildung 04-02-02: Lorenz-Kurve der Einkommensverteilung*


Um die Bedeutung unterschiedlicher Regionen in Bezug auf die aggregierten Daten zu zeigen werden mit Hilfe des `raster` Pakets neun unterschiedliche  Regionalisierungen mit der Ausdehnung und Georeferenzierung von Deutschland erzeugt. 


```r
# create different sized and numbered regions
r1 <- raster(ncol=1, nrow=4, xmn=extent(nuts3_kreise)[1], xmx=extent(nuts3_kreise)[3], ymn=extent(nuts3_kreise)[2], ymx=extent(nuts3_kreise)[4], crs=NA)
r1 <- rasterize(xy, r1, income, mean)
r2 <- raster(ncol=4, nrow=1, xmn=extent(nuts3_kreise)[1], xmx=extent(nuts3_kreise)[3], ymn=extent(nuts3_kreise)[2], ymx=extent(nuts3_kreise)[4], crs=NA)
r2 <- rasterize(xy, r2, income, mean)
r3 <- raster(ncol=2, nrow=2, xmn=extent(nuts3_kreise)[1], xmx=extent(nuts3_kreise)[3], ymn=extent(nuts3_kreise)[2], ymx=extent(nuts3_kreise)[4], crs=NA)
r3 <- rasterize(xy, r3, income, mean)
r4 <- raster(ncol=3, nrow=3, xmn=extent(nuts3_kreise)[1], xmx=extent(nuts3_kreise)[3], ymn=extent(nuts3_kreise)[2], ymx=extent(nuts3_kreise)[4], crs=NA)
r4 <- rasterize(xy, r4, income, mean)
r5 <- raster(ncol=5, nrow=5, xmn=extent(nuts3_kreise)[1], xmx=extent(nuts3_kreise)[3], ymn=extent(nuts3_kreise)[2], ymx=extent(nuts3_kreise)[4], crs=NA)
r5 <- rasterize(xy, r5, income, mean)
r6 <- raster(ncol=10, nrow=10, xmn=extent(nuts3_kreise)[1], xmx=extent(nuts3_kreise)[3], ymn=extent(nuts3_kreise)[2], ymx=extent(nuts3_kreise)[4], crs=NA)
r6 <- rasterize(xy, r6, income, mean)
r7 <- raster(ncol=20, nrow=20, xmn=extent(nuts3_kreise)[1], xmx=extent(nuts3_kreise)[3], ymn=extent(nuts3_kreise)[2], ymx=extent(nuts3_kreise)[4], crs=NA)
r7 <- rasterize(xy, r7, income, mean)
r8 <- raster(ncol=50, nrow=50, xmn=extent(nuts3_kreise)[1], xmx=extent(nuts3_kreise)[3], ymn=extent(nuts3_kreise)[2], ymx=extent(nuts3_kreise)[4], crs=NA)
r8 <- rasterize(xy, r8, income, mean)
r9 <- raster(ncol=100, nrow=100, xmn=extent(nuts3_kreise)[1], xmx=extent(nuts3_kreise)[3], ymn=extent(nuts3_kreise)[2], ymx=extent(nuts3_kreise)[4], crs=NA)
r9 <- rasterize(xy, r9, income, mean)
```

Die einzelnen Grafiken verdeutlichen wie die räumliche Anordnung der Zonen verteilt ist. Es handelt sich um 2 Streifenzonierungen und  7 unterschiedlich aufgelöste Gitter von  4x4, 5x5, 10x10, 20x20, 50x50 und 100x100 Regionen.


```r
# Festlegen der Grafik-Ausgabe
par(mfrow=c(3,3), las=1)

# Plotten der 9 Regionen
plot(r1,main="ncol=1, nrow=4"); plot(r2,main="ncol=4, nrow=1");
plot(r3,main="ncol=2, nrow=2"); plot(r4,main="ncol=3, nrow=3");
plot(r5,main="ncol=5, nrow=5"); plot(r6,main="ncol=10,nrow=10");
plot(r7,main="ncol=20, nrow=20");plot(r8,main="ncol=50, nrow=50");
plot(r9,main="ncol=100, nrow=100")
```

<img src="{{ site.baseurl }}/assets/images/unit04/plot_zone_raster-1.png" width="1100px" height="900px" />

*Abbildung 04-02-03: 3x3 Matrix der unterschiedlichen Zonen-Anordnungen*


Wenn man nun die korrespondierenden zonalen Histogramme anschaut, wird deutlich wie sehr eine Zonierung die Verteilung der Ergebnisse im Raum beeinflusst.



```r
# Festlegen der Grafik-Ausgabe
par(mfrow=c(3,3), las=1)

# Plotten der zugehörigen Histogramme
hist(r1, main='', col=rev(terrain.colors(10)), xlim=c(0,125000), breaks=seq(0, 125000, 12500))
hist(r2, main='', col=rev(terrain.colors(10)), xlim=c(0,125000), breaks=seq(0, 125000, 12500))
hist(r3, main='', col=rev(terrain.colors(10)), xlim=c(0,125000), breaks=seq(0, 125000, 12500))
hist(r4, main='', col=rev(terrain.colors(10)), xlim=c(0,125000), breaks=seq(0, 125000, 12500))
hist(r5, main='', col=rev(terrain.colors(10)), xlim=c(0,125000), breaks=seq(0, 125000, 12500))
hist(r6, main='', col=rev(terrain.colors(10)), xlim=c(0,125000), breaks=seq(0, 125000, 12500))
hist(r7, main='', col=rev(terrain.colors(10)), xlim=c(0,125000), breaks=seq(0, 125000, 12500))
hist(r8, main='', col=rev(terrain.colors(10)), xlim=c(0,125000), breaks=seq(0, 125000, 12500))
hist(r9, main='', col=rev(terrain.colors(10)), xlim=c(0,125000), breaks=seq(0, 125000, 12500))
```

![]({{ site.baseurl }}/assets/images/unit04/zone_hist-1.png)<!-- -->

*Abbildung 04-02-04: 3x3 Matrix der unterschiedlichen Zonen-Histogramme*



## Distanz

Als Distanz wird die Entfernung von zwei Positionen bezeichnet. Sie kann zunächst einmal als ein zentrales Konzept der Geographie angenommen werden. Erinnern sie sich an Waldo Toblers ersten Satz zur Geographie, dass "*alles mit allem anderen verwandt ist, aber nahe Dinge mehr verwandt sind als ferne Dinge*".  Die Entfernung ist scheinbar sehr einfach zu bestimmen. Natürlich können wir die Entfernung auf eine isometrischen und isomorhpen Fläche mittels der "*Luftlinie*" (euklidische Distanz) berechnen. Zentrales Problem ist das diese Betrachtung häufig wenn in der Regel nicht relevant ist. Es gibt nicht nur (nationale) Grenzen, Gebirge oder beliebige andere Hindernisse, die Entfernung zwischen A und B kann auch asymmetrisch sein (bergab geht's einfacher und  schneller  als bergauf). Das heißt Distanzen können auch über z.B. *Distanzkosten* gewichtet werden.

Üblicherweise werden Distanzen in einer "Distanzmatrix" dargestellt. Eine solche Matrix enthält als Spaltenüberschriften und als Zeilenbeschriftung die Kennung von jedem berechneten Ort. Im jedem Feld wird die Entfernung eingetragen. Für kartesische Koordinaten erfolgt dies einfach über den Satz des Pythagoras.

Betrachten wir diese Zusammenhänge einmal ganz praktisch:

Wir erstellen für eine Anzahl Punkte eine Distanzmatrix. Wenn die Positionen in Länge/Breite angegeben sind wird es deutlich aufwendiger. In diesem Fall können wir die Funktion `pointDistance` aus dem `raster` Paket verwenden (allerdings nur wenn das Koordinatensystem korrekt angegeben wird). Eleganter ist jedoch die Erzeugung von Punktdaten als `sf` Objekt. Es ist leicht möglich  beliebige reale Punktdaten mit Hilfe des `tidygeocoder` Pakets zu erzeugen.Wir geben lediglich eine Städte oder Adressliste an:


```r
# Erzeugen von beliebigen Punkten mit Hilfe von tidygeocoder
# Städteliste
staedte=c("München","Berlin","Hamburg","Köln","Bonn","Hannover","Nürnberg","Stuttgart","Freiburg","Marburg")

# Abfragen der Geokoordinaten der Städte mit eine lapply Schleife
 coord_city = lapply(staedte, function(x){
 latlon = c(geo_osm(x)[2],geo_osm(x)[1])
 class(latlon) = "numeric"
  p = st_sfc(st_point(latlon), crs = 4326)
 st_sf(name = x,p)
 #st_sf(p)
 })
 
# Umwandeln der Liste in eine Matrix mit den Stadtnamen und Spalten die Lat Lon benannt sind
 geo_coord_city = do.call("rbind", coord_city)

# plotten der Punkte
 mapview(geo_coord_city,  color='red',legend = FALSE)
```

<!--html_preserve--><div id="htmlwidget-6351161e92343e344deb" style="width:800px;height:600px;" class="leaflet html-widget"></div>
<script type="application/json" data-for="htmlwidget-6351161e92343e344deb">{"x":{"options":{"minZoom":1,"maxZoom":52,"crs":{"crsClass":"L.CRS.EPSG3857","code":null,"proj4def":null,"projectedBounds":null,"options":{}},"preferCanvas":false,"bounceAtZoomLimits":false,"maxBounds":[[[-90,-370]],[[90,370]]]},"calls":[{"method":"addProviderTiles","args":["CartoDB.Positron",1,"CartoDB.Positron",{"errorTileUrl":"","noWrap":false,"detectRetina":false}]},{"method":"addProviderTiles","args":["CartoDB.DarkMatter",2,"CartoDB.DarkMatter",{"errorTileUrl":"","noWrap":false,"detectRetina":false}]},{"method":"addProviderTiles","args":["OpenStreetMap",3,"OpenStreetMap",{"errorTileUrl":"","noWrap":false,"detectRetina":false}]},{"method":"addProviderTiles","args":["Esri.WorldImagery",4,"Esri.WorldImagery",{"errorTileUrl":"","noWrap":false,"detectRetina":false}]},{"method":"addProviderTiles","args":["OpenTopoMap",5,"OpenTopoMap",{"errorTileUrl":"","noWrap":false,"detectRetina":false}]},{"method":"createMapPane","args":["point",440]},{"method":"addCircleMarkers","args":[[48.1371079,52.5170365,53.5437641,50.938361,50.735851,52.3744779,49.453872,48.7784485,47.9960901,50.8117327],[11.5753822,13.3888599,10.0099133,6.959974,7.10066,9.7385532,11.077298,9.1800132,7.8494005,8.7757742],6,null,"geo_coord_city",{"crs":{"crsClass":"L.CRS.EPSG3857","code":null,"proj4def":null,"projectedBounds":null,"options":{}},"pane":"point","stroke":true,"color":"#FF0000","weight":2,"opacity":[0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9],"fill":true,"fillColor":["#6DCD59","#440154","#31688E","#1F9E89","#482878","#26828E","#B4DE2C","#FDE725","#3E4A89","#35B779"],"fillOpacity":[0.6,0.6,0.6,0.6,0.6,0.6,0.6,0.6,0.6,0.6]},null,null,["<html><head><link rel=\"stylesheet\" type=\"text/css\" href=\"lib/popup/popup.css\"><\/head><body><div class=\"scrollableContainer\"><table class=\"popup scrollable\" id=\"popup\"><tr class='coord'><td><\/td><td><b>Feature ID<\/b><\/td><td align='right'>1&emsp;<\/td><\/tr><tr class='alt'><td>1<\/td><td><b>name&emsp;<\/b><\/td><td align='right'>München&emsp;<\/td><\/tr><tr><td>2<\/td><td><b>p&emsp;<\/b><\/td><td align='right'>sfc_POINT&emsp;<\/td><\/tr><\/table><\/div><\/body><\/html>","<html><head><link rel=\"stylesheet\" type=\"text/css\" href=\"lib/popup/popup.css\"><\/head><body><div class=\"scrollableContainer\"><table class=\"popup scrollable\" id=\"popup\"><tr class='coord'><td><\/td><td><b>Feature ID<\/b><\/td><td align='right'>2&emsp;<\/td><\/tr><tr class='alt'><td>1<\/td><td><b>name&emsp;<\/b><\/td><td align='right'>Berlin&emsp;<\/td><\/tr><tr><td>2<\/td><td><b>p&emsp;<\/b><\/td><td align='right'>sfc_POINT&emsp;<\/td><\/tr><\/table><\/div><\/body><\/html>","<html><head><link rel=\"stylesheet\" type=\"text/css\" href=\"lib/popup/popup.css\"><\/head><body><div class=\"scrollableContainer\"><table class=\"popup scrollable\" id=\"popup\"><tr class='coord'><td><\/td><td><b>Feature ID<\/b><\/td><td align='right'>3&emsp;<\/td><\/tr><tr class='alt'><td>1<\/td><td><b>name&emsp;<\/b><\/td><td align='right'>Hamburg&emsp;<\/td><\/tr><tr><td>2<\/td><td><b>p&emsp;<\/b><\/td><td align='right'>sfc_POINT&emsp;<\/td><\/tr><\/table><\/div><\/body><\/html>","<html><head><link rel=\"stylesheet\" type=\"text/css\" href=\"lib/popup/popup.css\"><\/head><body><div class=\"scrollableContainer\"><table class=\"popup scrollable\" id=\"popup\"><tr class='coord'><td><\/td><td><b>Feature ID<\/b><\/td><td align='right'>4&emsp;<\/td><\/tr><tr class='alt'><td>1<\/td><td><b>name&emsp;<\/b><\/td><td align='right'>Köln&emsp;<\/td><\/tr><tr><td>2<\/td><td><b>p&emsp;<\/b><\/td><td align='right'>sfc_POINT&emsp;<\/td><\/tr><\/table><\/div><\/body><\/html>","<html><head><link rel=\"stylesheet\" type=\"text/css\" href=\"lib/popup/popup.css\"><\/head><body><div class=\"scrollableContainer\"><table class=\"popup scrollable\" id=\"popup\"><tr class='coord'><td><\/td><td><b>Feature ID<\/b><\/td><td align='right'>5&emsp;<\/td><\/tr><tr class='alt'><td>1<\/td><td><b>name&emsp;<\/b><\/td><td align='right'>Bonn&emsp;<\/td><\/tr><tr><td>2<\/td><td><b>p&emsp;<\/b><\/td><td align='right'>sfc_POINT&emsp;<\/td><\/tr><\/table><\/div><\/body><\/html>","<html><head><link rel=\"stylesheet\" type=\"text/css\" href=\"lib/popup/popup.css\"><\/head><body><div class=\"scrollableContainer\"><table class=\"popup scrollable\" id=\"popup\"><tr class='coord'><td><\/td><td><b>Feature ID<\/b><\/td><td align='right'>6&emsp;<\/td><\/tr><tr class='alt'><td>1<\/td><td><b>name&emsp;<\/b><\/td><td align='right'>Hannover&emsp;<\/td><\/tr><tr><td>2<\/td><td><b>p&emsp;<\/b><\/td><td align='right'>sfc_POINT&emsp;<\/td><\/tr><\/table><\/div><\/body><\/html>","<html><head><link rel=\"stylesheet\" type=\"text/css\" href=\"lib/popup/popup.css\"><\/head><body><div class=\"scrollableContainer\"><table class=\"popup scrollable\" id=\"popup\"><tr class='coord'><td><\/td><td><b>Feature ID<\/b><\/td><td align='right'>7&emsp;<\/td><\/tr><tr class='alt'><td>1<\/td><td><b>name&emsp;<\/b><\/td><td align='right'>Nürnberg&emsp;<\/td><\/tr><tr><td>2<\/td><td><b>p&emsp;<\/b><\/td><td align='right'>sfc_POINT&emsp;<\/td><\/tr><\/table><\/div><\/body><\/html>","<html><head><link rel=\"stylesheet\" type=\"text/css\" href=\"lib/popup/popup.css\"><\/head><body><div class=\"scrollableContainer\"><table class=\"popup scrollable\" id=\"popup\"><tr class='coord'><td><\/td><td><b>Feature ID<\/b><\/td><td align='right'>8&emsp;<\/td><\/tr><tr class='alt'><td>1<\/td><td><b>name&emsp;<\/b><\/td><td align='right'>Stuttgart&emsp;<\/td><\/tr><tr><td>2<\/td><td><b>p&emsp;<\/b><\/td><td align='right'>sfc_POINT&emsp;<\/td><\/tr><\/table><\/div><\/body><\/html>","<html><head><link rel=\"stylesheet\" type=\"text/css\" href=\"lib/popup/popup.css\"><\/head><body><div class=\"scrollableContainer\"><table class=\"popup scrollable\" id=\"popup\"><tr class='coord'><td><\/td><td><b>Feature ID<\/b><\/td><td align='right'>9&emsp;<\/td><\/tr><tr class='alt'><td>1<\/td><td><b>name&emsp;<\/b><\/td><td align='right'>Freiburg&emsp;<\/td><\/tr><tr><td>2<\/td><td><b>p&emsp;<\/b><\/td><td align='right'>sfc_POINT&emsp;<\/td><\/tr><\/table><\/div><\/body><\/html>","<html><head><link rel=\"stylesheet\" type=\"text/css\" href=\"lib/popup/popup.css\"><\/head><body><div class=\"scrollableContainer\"><table class=\"popup scrollable\" id=\"popup\"><tr class='coord'><td><\/td><td><b>Feature ID<\/b><\/td><td align='right'>10&emsp;<\/td><\/tr><tr class='alt'><td>1<\/td><td><b>name&emsp;<\/b><\/td><td align='right'>Marburg&emsp;<\/td><\/tr><tr><td>2<\/td><td><b>p&emsp;<\/b><\/td><td align='right'>sfc_POINT&emsp;<\/td><\/tr><\/table><\/div><\/body><\/html>"],{"maxWidth":800,"minWidth":50,"autoPan":true,"keepInView":false,"closeButton":true,"closeOnClick":true,"className":""},["München","Berlin","Hamburg","Köln","Bonn","Hannover","Nürnberg","Stuttgart","Freiburg","Marburg"],{"interactive":false,"permanent":false,"direction":"auto","opacity":1,"offset":[0,0],"textsize":"10px","textOnly":false,"className":"","sticky":true},null]},{"method":"addScaleBar","args":[{"maxWidth":100,"metric":true,"imperial":true,"updateWhenIdle":true,"position":"bottomleft"}]},{"method":"addHomeButton","args":[6.959974,47.9960901,13.3888599,53.5437641,"geo_coord_city","Zoom to geo_coord_city","<strong> geo_coord_city <\/strong>","bottomright"]},{"method":"addLayersControl","args":[["CartoDB.Positron","CartoDB.DarkMatter","OpenStreetMap","Esri.WorldImagery","OpenTopoMap"],"geo_coord_city",{"collapsed":true,"autoZIndex":true,"position":"topleft"}]}],"limits":{"lat":[47.9960901,53.5437641],"lng":[6.959974,13.3888599]}},"evals":[],"jsHooks":{"render":[{"code":"function(el, x, data) {\n  return (\n      function(el, x, data) {\n      // get the leaflet map\n      var map = this; //HTMLWidgets.find('#' + el.id);\n      // we need a new div element because we have to handle\n      // the mouseover output separately\n      // debugger;\n      function addElement () {\n      // generate new div Element\n      var newDiv = $(document.createElement('div'));\n      // append at end of leaflet htmlwidget container\n      $(el).append(newDiv);\n      //provide ID and style\n      newDiv.addClass('lnlt');\n      newDiv.css({\n      'position': 'relative',\n      'bottomleft':  '0px',\n      'background-color': 'rgba(255, 255, 255, 0.7)',\n      'box-shadow': '0 0 2px #bbb',\n      'background-clip': 'padding-box',\n      'margin': '0',\n      'padding-left': '5px',\n      'color': '#333',\n      'font': '9px/1.5 \"Helvetica Neue\", Arial, Helvetica, sans-serif',\n      'z-index': '700',\n      });\n      return newDiv;\n      }\n\n\n      // check for already existing lnlt class to not duplicate\n      var lnlt = $(el).find('.lnlt');\n\n      if(!lnlt.length) {\n      lnlt = addElement();\n\n      // grab the special div we generated in the beginning\n      // and put the mousmove output there\n\n      map.on('mousemove', function (e) {\n      if (e.originalEvent.ctrlKey) {\n      if (document.querySelector('.lnlt') === null) lnlt = addElement();\n      lnlt.text(\n                           ' lon: ' + (e.latlng.lng).toFixed(5) +\n                           ' | lat: ' + (e.latlng.lat).toFixed(5) +\n                           ' | zoom: ' + map.getZoom() +\n                           ' | x: ' + L.CRS.EPSG3857.project(e.latlng).x.toFixed(0) +\n                           ' | y: ' + L.CRS.EPSG3857.project(e.latlng).y.toFixed(0) +\n                           ' | epsg: 3857 ' +\n                           ' | proj4: +proj=merc +a=6378137 +b=6378137 +lat_ts=0.0 +lon_0=0.0 +x_0=0.0 +y_0=0 +k=1.0 +units=m +nadgrids=@null +no_defs ');\n      } else {\n      if (document.querySelector('.lnlt') === null) lnlt = addElement();\n      lnlt.text(\n                      ' lon: ' + (e.latlng.lng).toFixed(5) +\n                      ' | lat: ' + (e.latlng.lat).toFixed(5) +\n                      ' | zoom: ' + map.getZoom() + ' ');\n      }\n      });\n\n      // remove the lnlt div when mouse leaves map\n      map.on('mouseout', function (e) {\n      var strip = document.querySelector('.lnlt');\n      strip.remove();\n      });\n\n      };\n\n      //$(el).keypress(67, function(e) {\n      map.on('preclick', function(e) {\n      if (e.originalEvent.ctrlKey) {\n      if (document.querySelector('.lnlt') === null) lnlt = addElement();\n      lnlt.text(\n                      ' lon: ' + (e.latlng.lng).toFixed(5) +\n                      ' | lat: ' + (e.latlng.lat).toFixed(5) +\n                      ' | zoom: ' + map.getZoom() + ' ');\n      var txt = document.querySelector('.lnlt').textContent;\n      console.log(txt);\n      //txt.innerText.focus();\n      //txt.select();\n      setClipboardText('\"' + txt + '\"');\n      }\n      });\n\n      }\n      ).call(this.getMap(), el, x, data);\n}","data":null}]}}</script><!--/html_preserve-->

Zuerst die Ausgabe mit dem Paket `mapview`.

{% include media url="/assets/misc/geo_city_city.html" %}
[Full-screen version of the map]({{ site.baseurl }}/assets/misc/geo_city_city.html){:target="_blank"}

*Abbildung 04-02-05: Webkarte mit den erzeugten Punktdaten. In diesem Falle zehn nicht ganz zufällige Städte Deutschlands*

Dann die klassische Variante mit der `plot` Funktion.


```r
# Festlegen der Grafik-Ausgabe
 
# klassisches Plotten eines sf Objects  erfordert den Zugriff auf die Koordinatenpaare
# mit Hilfe der Funktion st_coordinates(geo_coord_city) leicht möglich
# mit Hilfe der Funktion min(st_coordinates(geo_coord_city)[,1]) werden 
# minimum und maximum Ausdehnung bestimmt 
 plot(st_coordinates(geo_coord_city),
     xlim = c(min(st_coordinates(geo_coord_city)[,1]) - 0.5 
     ,max(st_coordinates(geo_coord_city)[,1]) + 1), 
     ylim = c(min(st_coordinates(geo_coord_city)[,2]) - 0.5
     ,max(st_coordinates(geo_coord_city)[,2]) + 0.5),
     pch=20, cex=1.5, col='darkgreen', xlab='Längengrad', ylab='Breitengrad')
     text(st_coordinates(geo_coord_city), labels = staedte, cex=1.2, pos=4, col="purple")
```

<img src="{{ site.baseurl }}/assets/images/unit04/points-2-1.png" width="800px" height="600px" />

*Abbildung 04-02-06: Klassische Plot-Ausgabe mit den erzeugten Punktdaten. Erneut zehn nicht ganz zufällige Städte Deutschlands*

Die Berechnung der Distanzen zwischen den 10 Städten greift einigermaßen tief in Geodaten-Verarbeitung ein. Die Punkte werden über die Funktion `tidygeocoder::geo_osm` mit Hilfe der Stadtnamen erzeugt. Standardisiert werden Kugelkoordinaten (also geographische Koordinaten) erzeugt also Längen und Breitengrade auf Grundlage des WGS84 Datum. Auf einem Ellipsoid ist es deutlich aufwendiger (oder fehlerträchtiger) Entfernungen zu rechnen als in in einem projizierten (also kartesischen) Koordinatensystem da hier sphärische Trigonometrie im Gegensatz zum Satz des Pythagoras zur Anwendung kommt.

Daher transformieren wir zunächst den Datensatz in das amtlich gültige [Referenzsystem](https://de.wikipedia.org/wiki/Europ%C3%A4isches_Terrestrisches_Referenzsystem_1989) für Deutschland nämlich `ETRS89/UTM`. Zur Zuweisung werden viele konkurrierende Systeme verwendet. Im nachstehenden Beispiel nutzen wir die [EPSG](https://de.wikipedia.org/wiki/European_Petroleum_Survey_Group_Geodesy#EPSG-Codes) Konvention. Für das zuvor genannte System ist das der [EPSG-Code 25832](https://epsg.io/25832).


```r
# Zuerst projizieren wir den Datensatz auf ETRS89/UTM
 proj_coord_city = st_transform(geo_coord_city, crs = 25832)

# nun berechnen wir die Distanzen
 city_distanz = dist(st_coordinates(proj_coord_city))
# mit Hilfe von dist_setNames können wir die Namen der distanzmatrix zuweisen
 dist_setNames(city_distanz, staedte)
```

```
##            München   Berlin  Hamburg     Köln     Bonn Hannover Nürnberg
## Berlin    504156.6                                                      
## Hamburg   611337.7 253841.6                                             
## Köln      456511.6 477393.1 356810.5                                    
## Bonn      434329.5 478145.8 370325.9  24607.7                           
## Hannover  489061.8 248686.4 131348.9 249893.3 258162.1                  
## Nürnberg  150928.4 377495.0 460900.6 337014.5 318118.3 338167.6         
## Stuttgart 190926.8 511243.2 533104.9 288316.8 264173.9 401815.7 157508.4
## Freiburg  278029.8 639022.5 635366.9 333440.4 309440.8 505124.9 287407.8
## Marburg   359924.1 371191.0 315365.1 128538.4 118421.0 186154.7 223271.1
##           Stuttgart Freiburg
## Berlin                      
## Hamburg                     
## Köln                        
## Bonn                        
## Hannover                    
## Nürnberg                    
## Stuttgart                   
## Freiburg   131404.0         
## Marburg    227925.4 320161.6
```

```r
city_distanz
```

```
##           1        2        3        4        5        6        7        8
## 2  504156.6                                                               
## 3  611337.7 253841.6                                                      
## 4  456511.6 477393.1 356810.5                                             
## 5  434329.5 478145.8 370325.9  24607.7                                    
## 6  489061.8 248686.4 131348.9 249893.3 258162.1                           
## 7  150928.4 377495.0 460900.6 337014.5 318118.3 338167.6                  
## 8  190926.8 511243.2 533104.9 288316.8 264173.9 401815.7 157508.4         
## 9  278029.8 639022.5 635366.9 333440.4 309440.8 505124.9 287407.8 131404.0
## 10 359924.1 371191.0 315365.1 128538.4 118421.0 186154.7 223271.1 227925.4
##           9
## 2          
## 3          
## 4          
## 5          
## 6          
## 7          
## 8          
## 9          
## 10 320161.6
```



Wir erzeugen aus der Distanz-Matrix `class = dist` eine normale R-Matrix. Leider müssen dann die Namen wieder zugewiesen werden.


```r
# make a full matrix an
city_distanz <- as.matrix(city_distanz)
rownames(city_distanz)=staedte
colnames(city_distanz)=staedte
```


```r
# Ausgabe einer hübscheren Tabelle mit kintr::kable die Notation ist das sogennante pipen aus der tidyverse Welt
knitr::kable(city_distanz) %>%
  kable_styling(bootstrap_options = "striped", full_width = F)
```

<table class="table table-striped" style="width: auto !important; margin-left: auto; margin-right: auto;">
 <thead>
  <tr>
   <th style="text-align:left;">   </th>
   <th style="text-align:right;"> München </th>
   <th style="text-align:right;"> Berlin </th>
   <th style="text-align:right;"> Hamburg </th>
   <th style="text-align:right;"> Köln </th>
   <th style="text-align:right;"> Bonn </th>
   <th style="text-align:right;"> Hannover </th>
   <th style="text-align:right;"> Nürnberg </th>
   <th style="text-align:right;"> Stuttgart </th>
   <th style="text-align:right;"> Freiburg </th>
   <th style="text-align:right;"> Marburg </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;"> München </td>
   <td style="text-align:right;"> 0.0 </td>
   <td style="text-align:right;"> 504156.6 </td>
   <td style="text-align:right;"> 611337.7 </td>
   <td style="text-align:right;"> 456511.6 </td>
   <td style="text-align:right;"> 434329.5 </td>
   <td style="text-align:right;"> 489061.8 </td>
   <td style="text-align:right;"> 150928.4 </td>
   <td style="text-align:right;"> 190926.8 </td>
   <td style="text-align:right;"> 278029.8 </td>
   <td style="text-align:right;"> 359924.1 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Berlin </td>
   <td style="text-align:right;"> 504156.6 </td>
   <td style="text-align:right;"> 0.0 </td>
   <td style="text-align:right;"> 253841.6 </td>
   <td style="text-align:right;"> 477393.1 </td>
   <td style="text-align:right;"> 478145.8 </td>
   <td style="text-align:right;"> 248686.4 </td>
   <td style="text-align:right;"> 377495.0 </td>
   <td style="text-align:right;"> 511243.2 </td>
   <td style="text-align:right;"> 639022.5 </td>
   <td style="text-align:right;"> 371191.0 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Hamburg </td>
   <td style="text-align:right;"> 611337.7 </td>
   <td style="text-align:right;"> 253841.6 </td>
   <td style="text-align:right;"> 0.0 </td>
   <td style="text-align:right;"> 356810.5 </td>
   <td style="text-align:right;"> 370325.9 </td>
   <td style="text-align:right;"> 131348.9 </td>
   <td style="text-align:right;"> 460900.6 </td>
   <td style="text-align:right;"> 533104.9 </td>
   <td style="text-align:right;"> 635366.9 </td>
   <td style="text-align:right;"> 315365.1 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Köln </td>
   <td style="text-align:right;"> 456511.6 </td>
   <td style="text-align:right;"> 477393.1 </td>
   <td style="text-align:right;"> 356810.5 </td>
   <td style="text-align:right;"> 0.0 </td>
   <td style="text-align:right;"> 24607.7 </td>
   <td style="text-align:right;"> 249893.3 </td>
   <td style="text-align:right;"> 337014.5 </td>
   <td style="text-align:right;"> 288316.8 </td>
   <td style="text-align:right;"> 333440.4 </td>
   <td style="text-align:right;"> 128538.4 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Bonn </td>
   <td style="text-align:right;"> 434329.5 </td>
   <td style="text-align:right;"> 478145.8 </td>
   <td style="text-align:right;"> 370325.9 </td>
   <td style="text-align:right;"> 24607.7 </td>
   <td style="text-align:right;"> 0.0 </td>
   <td style="text-align:right;"> 258162.1 </td>
   <td style="text-align:right;"> 318118.3 </td>
   <td style="text-align:right;"> 264173.9 </td>
   <td style="text-align:right;"> 309440.8 </td>
   <td style="text-align:right;"> 118421.0 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Hannover </td>
   <td style="text-align:right;"> 489061.8 </td>
   <td style="text-align:right;"> 248686.4 </td>
   <td style="text-align:right;"> 131348.9 </td>
   <td style="text-align:right;"> 249893.3 </td>
   <td style="text-align:right;"> 258162.1 </td>
   <td style="text-align:right;"> 0.0 </td>
   <td style="text-align:right;"> 338167.6 </td>
   <td style="text-align:right;"> 401815.7 </td>
   <td style="text-align:right;"> 505124.9 </td>
   <td style="text-align:right;"> 186154.7 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Nürnberg </td>
   <td style="text-align:right;"> 150928.4 </td>
   <td style="text-align:right;"> 377495.0 </td>
   <td style="text-align:right;"> 460900.6 </td>
   <td style="text-align:right;"> 337014.5 </td>
   <td style="text-align:right;"> 318118.3 </td>
   <td style="text-align:right;"> 338167.6 </td>
   <td style="text-align:right;"> 0.0 </td>
   <td style="text-align:right;"> 157508.4 </td>
   <td style="text-align:right;"> 287407.8 </td>
   <td style="text-align:right;"> 223271.1 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Stuttgart </td>
   <td style="text-align:right;"> 190926.8 </td>
   <td style="text-align:right;"> 511243.2 </td>
   <td style="text-align:right;"> 533104.9 </td>
   <td style="text-align:right;"> 288316.8 </td>
   <td style="text-align:right;"> 264173.9 </td>
   <td style="text-align:right;"> 401815.7 </td>
   <td style="text-align:right;"> 157508.4 </td>
   <td style="text-align:right;"> 0.0 </td>
   <td style="text-align:right;"> 131404.0 </td>
   <td style="text-align:right;"> 227925.4 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Freiburg </td>
   <td style="text-align:right;"> 278029.8 </td>
   <td style="text-align:right;"> 639022.5 </td>
   <td style="text-align:right;"> 635366.9 </td>
   <td style="text-align:right;"> 333440.4 </td>
   <td style="text-align:right;"> 309440.8 </td>
   <td style="text-align:right;"> 505124.9 </td>
   <td style="text-align:right;"> 287407.8 </td>
   <td style="text-align:right;"> 131404.0 </td>
   <td style="text-align:right;"> 0.0 </td>
   <td style="text-align:right;"> 320161.6 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Marburg </td>
   <td style="text-align:right;"> 359924.1 </td>
   <td style="text-align:right;"> 371191.0 </td>
   <td style="text-align:right;"> 315365.1 </td>
   <td style="text-align:right;"> 128538.4 </td>
   <td style="text-align:right;"> 118421.0 </td>
   <td style="text-align:right;"> 186154.7 </td>
   <td style="text-align:right;"> 223271.1 </td>
   <td style="text-align:right;"> 227925.4 </td>
   <td style="text-align:right;"> 320161.6 </td>
   <td style="text-align:right;"> 0.0 </td>
  </tr>
</tbody>
</table>

## Räumlicher Einfluss

Die beiden Aspekte zuvor haben die räumlichen Verhältnisse in Form von Raumabgrenzung und Distanz beschrieben. In der räumlichen Analyse ist es jedoch von zentraler Bedeutung den räumlichen **Einfluss** zwischen geographischen Objekten zu schätzen bzw. zu messen. Dies kann prozessorientiert funktional erfolgen, der oberliegende Teil eines Baches fließt in den unterliegenden. In der datengetriebenen Betrachtungsweise ist dies in der Regel eine Funktion der *Nachbarschaft* oder der *(inversen) Entfernung*. Um damit in statistischen Modellen arbeiten zu können werden diese Konzepte als *räumliche Gewichtungsmatrix* ausgedrückt. 

Das generelle und schwerwiegende Problem ist, dass der räumliche Einfluss sehr komplex ist und faktisch nie gemessen werden kann. Daher gibt es zahllose Arten ihn zu schätzen. 

Zum Beispiel kann der räumliche Einfluss von Polygonen aufeinander (z.B, NUTS3 Verwaltungsbezirke) so ausgedrückt werden, dass sie eine/keine gemeinsame Grenze, sie kann als euklidische Distanz zwischen ihren Schwerpunkten bestimmt werden oder über die Länge gemeinsamer Grenzen gewichtet werden und so fort.

### Nachbarschaft

Die Nachbarschaft ist das vielleicht wichtigste Konzept. höherdimensionale Geoobjekte können als benachbart betrachtet werden wenn sie sich *berühren*, z.B. benachbarte Länder. Bei null-dimensionalen Objekten (Punkte) ist der gebräuchlichste Ansatz die Entfernung in Kombination mit einer Anzahl von Punkten für die Ermittlung der Nachbarschaft zu nutzen.

Betrachten wir diese Zusammenhänge einmal ganz praktisch:

Wir erstellen eine Nachbarschaftsmatrix für die oben erzeugten Punktdaten. Punkte seien *benachbart*, wenn sie innerhalb eines Abstands von z.B. 25 Kilometer liegen.


```r
# Distanzmatrix für Entfernungen > 250 km
cd = city_distanz  < 250000
```


```r
# Ausgabe einer hübscheren Tabelle mit kintr::kable die Notation ist das sogenannte pipen aus der tidyverse Welt
knitr::kable(cd) %>%
  kable_styling(bootstrap_options = "striped", full_width = F)
```

<table class="table table-striped" style="width: auto !important; margin-left: auto; margin-right: auto;">
 <thead>
  <tr>
   <th style="text-align:left;">   </th>
   <th style="text-align:left;"> München </th>
   <th style="text-align:left;"> Berlin </th>
   <th style="text-align:left;"> Hamburg </th>
   <th style="text-align:left;"> Köln </th>
   <th style="text-align:left;"> Bonn </th>
   <th style="text-align:left;"> Hannover </th>
   <th style="text-align:left;"> Nürnberg </th>
   <th style="text-align:left;"> Stuttgart </th>
   <th style="text-align:left;"> Freiburg </th>
   <th style="text-align:left;"> Marburg </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;"> München </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Berlin </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Hamburg </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Köln </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Bonn </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Hannover </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Nürnberg </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Stuttgart </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Freiburg </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> FALSE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Marburg </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
</tbody>
</table>



### Gewichtungs-Matrix für Punkte

Anstatt den räumlichen Einfluss als binären Wert (also topologisch benachbart ja/nein) auszudrücken, kann er als kontinuierlicher Wert ausgedrückt werden. Der einfachste Ansatz ist die Verwendung des inversen Abstands (je weiter entfernt, desto niedriger der Wert).



```r
# inverse Distanz
gewichtungs_matrix =  (1 / city_distanz)
```

```r
# inverse Distanz zum Quadrat
gewichtungs_matrix_q =  (1 / city_distanz ** 2)
```


```r
# Ausgabe einer hübscheren Tabelle mit kintr::kable die Notation ist das sogenannte pipen aus der tidyverse Welt
knitr::kable(gewichtungs_matrix) %>%
  kable_styling(bootstrap_options = "striped", full_width = F)
```

<table class="table table-striped" style="width: auto !important; margin-left: auto; margin-right: auto;">
 <thead>
  <tr>
   <th style="text-align:left;">   </th>
   <th style="text-align:right;"> München </th>
   <th style="text-align:right;"> Berlin </th>
   <th style="text-align:right;"> Hamburg </th>
   <th style="text-align:right;"> Köln </th>
   <th style="text-align:right;"> Bonn </th>
   <th style="text-align:right;"> Hannover </th>
   <th style="text-align:right;"> Nürnberg </th>
   <th style="text-align:right;"> Stuttgart </th>
   <th style="text-align:right;"> Freiburg </th>
   <th style="text-align:right;"> Marburg </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;"> München </td>
   <td style="text-align:right;"> Inf </td>
   <td style="text-align:right;"> 2.0e-06 </td>
   <td style="text-align:right;"> 1.6e-06 </td>
   <td style="text-align:right;"> 2.20e-06 </td>
   <td style="text-align:right;"> 2.30e-06 </td>
   <td style="text-align:right;"> 2.0e-06 </td>
   <td style="text-align:right;"> 6.6e-06 </td>
   <td style="text-align:right;"> 5.2e-06 </td>
   <td style="text-align:right;"> 3.6e-06 </td>
   <td style="text-align:right;"> 2.8e-06 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Berlin </td>
   <td style="text-align:right;"> 2.0e-06 </td>
   <td style="text-align:right;"> Inf </td>
   <td style="text-align:right;"> 3.9e-06 </td>
   <td style="text-align:right;"> 2.10e-06 </td>
   <td style="text-align:right;"> 2.10e-06 </td>
   <td style="text-align:right;"> 4.0e-06 </td>
   <td style="text-align:right;"> 2.6e-06 </td>
   <td style="text-align:right;"> 2.0e-06 </td>
   <td style="text-align:right;"> 1.6e-06 </td>
   <td style="text-align:right;"> 2.7e-06 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Hamburg </td>
   <td style="text-align:right;"> 1.6e-06 </td>
   <td style="text-align:right;"> 3.9e-06 </td>
   <td style="text-align:right;"> Inf </td>
   <td style="text-align:right;"> 2.80e-06 </td>
   <td style="text-align:right;"> 2.70e-06 </td>
   <td style="text-align:right;"> 7.6e-06 </td>
   <td style="text-align:right;"> 2.2e-06 </td>
   <td style="text-align:right;"> 1.9e-06 </td>
   <td style="text-align:right;"> 1.6e-06 </td>
   <td style="text-align:right;"> 3.2e-06 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Köln </td>
   <td style="text-align:right;"> 2.2e-06 </td>
   <td style="text-align:right;"> 2.1e-06 </td>
   <td style="text-align:right;"> 2.8e-06 </td>
   <td style="text-align:right;"> Inf </td>
   <td style="text-align:right;"> 4.06e-05 </td>
   <td style="text-align:right;"> 4.0e-06 </td>
   <td style="text-align:right;"> 3.0e-06 </td>
   <td style="text-align:right;"> 3.5e-06 </td>
   <td style="text-align:right;"> 3.0e-06 </td>
   <td style="text-align:right;"> 7.8e-06 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Bonn </td>
   <td style="text-align:right;"> 2.3e-06 </td>
   <td style="text-align:right;"> 2.1e-06 </td>
   <td style="text-align:right;"> 2.7e-06 </td>
   <td style="text-align:right;"> 4.06e-05 </td>
   <td style="text-align:right;"> Inf </td>
   <td style="text-align:right;"> 3.9e-06 </td>
   <td style="text-align:right;"> 3.1e-06 </td>
   <td style="text-align:right;"> 3.8e-06 </td>
   <td style="text-align:right;"> 3.2e-06 </td>
   <td style="text-align:right;"> 8.4e-06 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Hannover </td>
   <td style="text-align:right;"> 2.0e-06 </td>
   <td style="text-align:right;"> 4.0e-06 </td>
   <td style="text-align:right;"> 7.6e-06 </td>
   <td style="text-align:right;"> 4.00e-06 </td>
   <td style="text-align:right;"> 3.90e-06 </td>
   <td style="text-align:right;"> Inf </td>
   <td style="text-align:right;"> 3.0e-06 </td>
   <td style="text-align:right;"> 2.5e-06 </td>
   <td style="text-align:right;"> 2.0e-06 </td>
   <td style="text-align:right;"> 5.4e-06 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Nürnberg </td>
   <td style="text-align:right;"> 6.6e-06 </td>
   <td style="text-align:right;"> 2.6e-06 </td>
   <td style="text-align:right;"> 2.2e-06 </td>
   <td style="text-align:right;"> 3.00e-06 </td>
   <td style="text-align:right;"> 3.10e-06 </td>
   <td style="text-align:right;"> 3.0e-06 </td>
   <td style="text-align:right;"> Inf </td>
   <td style="text-align:right;"> 6.3e-06 </td>
   <td style="text-align:right;"> 3.5e-06 </td>
   <td style="text-align:right;"> 4.5e-06 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Stuttgart </td>
   <td style="text-align:right;"> 5.2e-06 </td>
   <td style="text-align:right;"> 2.0e-06 </td>
   <td style="text-align:right;"> 1.9e-06 </td>
   <td style="text-align:right;"> 3.50e-06 </td>
   <td style="text-align:right;"> 3.80e-06 </td>
   <td style="text-align:right;"> 2.5e-06 </td>
   <td style="text-align:right;"> 6.3e-06 </td>
   <td style="text-align:right;"> Inf </td>
   <td style="text-align:right;"> 7.6e-06 </td>
   <td style="text-align:right;"> 4.4e-06 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Freiburg </td>
   <td style="text-align:right;"> 3.6e-06 </td>
   <td style="text-align:right;"> 1.6e-06 </td>
   <td style="text-align:right;"> 1.6e-06 </td>
   <td style="text-align:right;"> 3.00e-06 </td>
   <td style="text-align:right;"> 3.20e-06 </td>
   <td style="text-align:right;"> 2.0e-06 </td>
   <td style="text-align:right;"> 3.5e-06 </td>
   <td style="text-align:right;"> 7.6e-06 </td>
   <td style="text-align:right;"> Inf </td>
   <td style="text-align:right;"> 3.1e-06 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Marburg </td>
   <td style="text-align:right;"> 2.8e-06 </td>
   <td style="text-align:right;"> 2.7e-06 </td>
   <td style="text-align:right;"> 3.2e-06 </td>
   <td style="text-align:right;"> 7.80e-06 </td>
   <td style="text-align:right;"> 8.40e-06 </td>
   <td style="text-align:right;"> 5.4e-06 </td>
   <td style="text-align:right;"> 4.5e-06 </td>
   <td style="text-align:right;"> 4.4e-06 </td>
   <td style="text-align:right;"> 3.1e-06 </td>
   <td style="text-align:right;"> Inf </td>
  </tr>
</tbody>
</table>

Wie auch die *räumliche Gewichtung* wird die Matrix oft *zeilennormiert*, das heißt dass die Summe der Gewichte für jede Zeile (Position oder Wert der Position) in der Matrix gleich ist. 

```r
# löschen der Inf Werte die durch den Selbstbezug der Punkte entestehen
gewichtungs_matrix <- as.matrix(gewichtungs_matrix)
rownames(gewichtungs_matrix)=staedte
colnames(gewichtungs_matrix)=staedte

gewichtungs_matrix[!is.finite(gewichtungs_matrix)] <- NA
zeilen_summe <- rowSums(gewichtungs_matrix,  na.rm=TRUE)
zeilen_summe
```

```
##      München       Berlin      Hamburg         Köln         Bonn     Hannover 
## 2.839529e-05 2.299421e-05 2.748176e-05 6.894169e-05 7.021031e-05 3.435182e-05 
##     Nürnberg    Stuttgart     Freiburg      Marburg 
## 3.481930e-05 3.715831e-05 2.915882e-05 4.222911e-05
```

### Gewichtungs-Matrix für Polygone

#### Binäre 4-er Nachbarschaft

Die Gewichtungsmatrix für Polygone zu bestimmen ist natürlich etwas komplexer dafür gibt es allerdings mit `spdep` ein bewährtes Paket.

Die Funktion `poly2nb` kann unter anderem um eine *Rook* Nachbarschaftsliste erzeugen. Sie dient als Grundlage einer Nachbarschaftsmatrix.


```r
rook = poly2nb(nuts3_kreise, row.names=nuts3_kreise$NUTS_NAME, queen=FALSE)
rook_ngb_matrix = nb2mat(rook, style='B', zero.policy = TRUE)

# Berechne die Anzahl der Nachbarn für jedes Gebiet
anzahl_nachbarn <- rowSums(rook_ngb_matrix)

# als prozentsatz
prozentzahl_nachbarn  = round(100 * table(anzahl_nachbarn) / length(anzahl_nachbarn), 1)

plot(st_geometry(nuts3_kreise), border="grey", reset=FALSE,
     main=paste("Binary neighbours", sep=""))
coords <- coordinates(as(nuts3_kreise,"Spatial"))
plot(rook, coords, col='red', lwd=2, add=TRUE)
```

<img src="{{ site.baseurl }}/assets/images/unit04/binary_ngb-1.png" width="1000px" height="1000px" />

*Abbildung 04-02-05: Binäre 4-er Nachbarschaft für die Landkreise Deutschlands*




#### Nearest-Distance Nachbarn

Nachfolgend werden exemplarisch die 3 bzw. 5 nächsten Nachbarn zu einem Kreis ausgewiesen.


```r
rn <- row.names(nuts3_kreise)

# Berechne die 3 und 5 Nachbarschaften
kreise_dist_k3 <- knn2nb(knearneigh(coords, k=3, RANN=FALSE))
kreise_dist_k5 <- knn2nb(knearneigh(coords, k=5, RANN=FALSE))
##
par(mfrow=c(1,2),las=1)
plot(st_geometry(nuts3_kreise), border="grey", reset=FALSE,
     main=paste("Drei Nachbarkreise", sep=""))

plot(kreise_dist_k3, lwd = 1, coords, add=TRUE,col="blue")

plot(st_geometry(nuts3_kreise), border="grey", reset=FALSE,
     main=paste("Fünf Nachbarkreise", sep=""))

plot(kreise_dist_k5, lwd =1, coords, add=TRUE,col="red")
```

<img src="{{ site.baseurl }}/assets/images/unit04/neareast_ngb-1.png" height="1000px" />

*Abbildung 04-02-06:Nächste 3-er/5-er Nachbarschaft für die Landkreise Deutschlands*




#### Distanz-basierte Nachbarschaft

nachfolgend wird für den Mittelwert, das dritte Quartil und den Maximalwert der Distanzverteilung aller Kreis-Zentroide die Nachbarschaft bestimmt.


```r
# berechne alle Distanzen für die Flächenschwerpunkte der Kreise
knn2nb = knn2nb(knearneigh(coords))

# erzeuge die Kreisdistanzen
kreise_dist <- unlist(nbdists(knn2nb, coords))
summary(kreise_dist)
```

```
##    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
##    1026   11485   20205   19811   26498   56096
```

```r
rn <- row.names(nuts3_kreise)

# berechne die Nachbarschaften für alle Kreise < mean 3. Quartil und Max Wert
nachbarschaft_mean <- dnearneigh(coords, 0, summary(kreise_dist)[4], row.names=rn)
nachbarschaft_max <- dnearneigh(coords, 0, summary(kreise_dist)[6], row.names=rn)
nachbarschaft_thirdQ <- dnearneigh(coords, 0, summary(kreise_dist)[5], row.names=rn)

# Plotten der drei Distanzen
par(mfrow=c(1,3), las=1)
# Maxwert
plot(st_geometry(nuts3_kreise), border="grey", reset=FALSE,
     main=paste("Nachbarkreise näher ",round(summary(kreise_dist)[6],0) ," km", sep=""))
plot(nachbarschaft_max, lwd =1,coords, add=TRUE,col="green")

# 3. Quartil
plot(st_geometry(nuts3_kreise), border="grey", reset=FALSE,
     main=paste("Nachbarkreise näher ",round(summary(kreise_dist)[5],0) ," km", sep=""))
plot(nachbarschaft_thirdQ, lwd = 2, coords, add=TRUE,col="red")

# Mittelwert
plot(st_geometry(nuts3_kreise), border="grey", reset=FALSE,
     main=paste("Nachbarkreise näher ",round(summary(kreise_dist)[4],0) ," km", sep=""))
plot(nachbarschaft_mean, lwd =3, coords, add=TRUE,col="blue")
```

![]({{ site.baseurl }}/assets/images/unit04/distanz-ngb-1.png)<!-- -->


*Abbildung 04-02-07: Distanzbasierte Nachbarschaften für die Landkreise Deutschlands, a) Nachbarschaften innerhalb der Maximaldistanz der Distanzmatrix, b) Nachbarschaften innerhalb der 3. Quartils der Distanzmatrix, c) b) Nachbarschaften innerhalb der Mittelwerts der Distanzmatrix*



## Räumliche Autokorrelation

Das Konzept der räumlichen Autokorrelation komplizierter als das der zeitlichen Autokorrelation räumliche Objekte haben in der Regel zwei Dimensionen und komplexe Formen aufweisen und daher Nähe unterschiedlich dimensional zusammenhängen kann.

Grundsätzlich beschreiben die räumlichen Autokorrelationsmaße die Ähnlichkeit, der  beobachteten Werte zueinander. Räumliche Autokorrelation entstehen durch Beobachtungen und Beobachtungen und Positionen/Objekte im Raum.

Die räumliche Autokorrelation in einer Variable kann exogen (sie wird durch eine andere räumlich autokorrelierte Variable verursacht, z.B. Niederschlag) oder endogen (sie wird durch den Prozess verursacht, der im Spiel ist, z.B. die Ausbreitung einer Krankheit) sein.

Eine häufig verwendete Statistik ist Moran's I und invers dazu  Geary's C. Binäre Daten werden mit dem Join-Count-Index getestet.

### Beispiel - Binäre Nachbarschaft


```r
nuts3_kreise_rook = poly2nb(nuts3_kreise, row.names=nuts3_kreise$NUTS_NAME, queen=FALSE)

coords <- coordinates(as(nuts3_kreise,"Spatial"))

plot(st_geometry(nuts3_kreise), border="grey", reset=FALSE,
     main=paste("Binary neighbours", sep=""))
plot(nuts3_kreise_rook, coords, col='red', lwd=2, add=TRUE)
```

![]({{ site.baseurl }}/assets/images/unit04/moran_setup-1.png)<!-- -->

*Abbildung 04-02-08: Binäre 4-er Nachbarschaft für die Landkreise Deutschlands - Hier für die Berechnung der räumlichen Autokorrelation*

### Moran's-I und Geary-C Test

Wie bereits bekannt ist hängt der Wert von Morans I deutlich von den Annahmen ab, die in die räumliche Gewichtungsmatrix verwendet werden. Die Idee ist, eine Matrix zu konstruieren, die Ihre Annahmen über das jeweilige räumliche Phänomen passend wiedergibt. Der übliche Ansatz besteht darin, eine Gewichtung von 1 zu geben, wenn zwei *Zonen* Nachbarn sind falls nicht wird eine 0 vergeben. Natürlich variiert die Definition von *Nachbarn* (vgl. Reader räumliche Konzepte und oben). Quasi-kontinuierlich ist der Ansatz eine inverse Distanzfunktion zur Bestimmung der Gewichte zu verwenden. Auch wenn in der Praxis fast nie vorzufinden sollte die Auswahl räumlicher Gewichtungsmatritzen das betreffende Phänomen abbilden. So ist die Benachbartheit entlang von Autobahnen für Warentransporte anders zu gewichten als beispielsweise über ein Gebirge oder einen See.

Der Moran-I-Test und der Geary C Test sind übliche Verfahren für die Überprüfung räumlicher Autokorrelation. Das Geary's-C ist invers mit Moran's-I, aber nicht identisch. Moran's-I ist eher ein Maß für die globale räumliche Autokorrelation, während Geary's-C eher auf eine lokale räumliche Autokorrelation reagiert. 

\\[I = \frac{n}{\sum_{i=1}^{n}\sum_{j=1}^{n}w_{ij}}
   \frac{\sum_{i=1}^{n}\sum_{j=1}^{n}w_{ij}(x_i-\bar{x})(x_j-\bar{x})}{\sum_{i=1}^{n}(x_i - \bar{x})^2}\\]

\\[ C = \frac{(n-1)}{2\sum_{i=1}^{n}\sum_{j=1}^{n}w_{ij}}
   \frac{\sum_{i=1}^{n}\sum_{j=1}^{n}w_{ij}(x_i-x_j)^2}{\sum_{i=1}^{n}(x_i - \bar{x})^2}\\]


wobei \\(x_i, i=1, \ldots, n\\)   \\({n}\\) Beobachtungen der interessierenden numerischen Variablen und \\(w_{ij}\\) die räumlichen Gewichte sind.

Im wesentlichen ist dies eine erweiterte Version der Formel zur Berechnung des Korrelationskoeffizienten mit einer Matrix an räumlichen Gewichten.


```r
nuts3_kreise_rook = poly2nb(nuts3_kreise, row.names=nuts3_kreise$NUTS_NAME, queen=FALSE)
w_nuts3_kreise_rook =  nb2listw(nuts3_kreise_rook, style='B',zero.policy = TRUE)
m_nuts3_kreise_rook =   nb2mat(nuts3_kreise_rook, style='B', zero.policy = TRUE)
nuts3_gewicht <- mat2listw(as.matrix(m_nuts3_kreise_rook))


# lineares Modell
lm_uni_bau = lm(nuts3_kreise$Anteil.Hochschulabschluss ~ nuts3_kreise$Anteil.Baugewerbe, data=nuts3_kreise)
summary(lm_uni_bau)
```

```
## 
## Call:
## lm(formula = nuts3_kreise$Anteil.Hochschulabschluss ~ nuts3_kreise$Anteil.Baugewerbe, 
##     data = nuts3_kreise)
## 
## Residuals:
##       Min        1Q    Median        3Q       Max 
## -0.080237 -0.026671 -0.005943  0.017007  0.165641 
## 
## Coefficients:
##                                 Estimate Std. Error t value Pr(>|t|)    
## (Intercept)                     0.175566   0.005368   32.71   <2e-16 ***
## nuts3_kreise$Anteil.Baugewerbe -0.988080   0.073797  -13.39   <2e-16 ***
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 0.0382 on 398 degrees of freedom
## Multiple R-squared:  0.3105,	Adjusted R-squared:  0.3088 
## F-statistic: 179.3 on 1 and 398 DF,  p-value: < 2.2e-16
```


```r
# Extraktion der Residuen
residuen_uni_bau <- lm (lm ( nuts3_kreise$Anteil.Hochschulabschluss ~ nuts3_kreise$Anteil.Baugewerbe, data=nuts3_kreise))$resid

# Moran I test rondomisiert und nicht randomisiert
m_nr_residuen_uni_bau = moran.test(residuen_uni_bau, nuts3_gewicht,randomisation=FALSE)
m_r_residuen_uni_bau = moran.test(residuen_uni_bau, nuts3_gewicht,randomisation=TRUE)
m_r_residuen_uni_bau
```

```
## 
## 	Moran I test under randomisation
## 
## data:  residuen_uni_bau  
## weights: nuts3_gewicht    
## 
## Moran I statistic standard deviate = 11.344, p-value < 2.2e-16
## alternative hypothesis: greater
## sample estimates:
## Moran I statistic       Expectation          Variance 
##      0.3451598142     -0.0025062657      0.0009392513
```
Anstelle des normalen Moran I  sollte eine Monte-Carlo-Simulation verwendet werden. Das ist eigentlich die einzige gute Methode um festzustellen, wie wahrscheinlich es ist, dass die beobachteten Werte als zufällige Ziehung angesehen werden können.



```r
moran.plot (residuen_uni_bau, nuts3_gewicht)
```

<img src="{{ site.baseurl }}/assets/images/unit04/moran_plot-1.png" width="800px" height="800px" />

*Abbildung 04-02-08: Moran-I Plot*


Für mehr Informationen kann unter den folgenden Ressourcen nachgeschaut werden: 

* [Spatial Data Analysis](https://rspatial.org/raster/analysis/2-scale_distance.html) von Robert Hijmans. Sehr umfangreich und empfehlenswert. Viel der Beispiele basieren auf seiner Vorlesung und sind für unsere Verhältnisse angepasst.

* Der [UseR! 2019 Spatial Workshop](https://edzer.github.io/UseR2019/part2.html) von Roger Bivand. Roger ist die absolute Referenz hinischtlich räumlicher Ökonometrie mit R. Er hat unzählige Pakete geschrieben und ebensoviel Schulungs-Material und ist unermüdlich in  der Unterstützung der Community.


## Download Skript
Das Skript kann unter [unit04-02_sitzung.R]({{ site.baseurl }}/assets/scripts/unit04-02_sitzung.R){:target="_blank"} heruntergeladen werden

## Bearbeiten Sie…
Versuchen Sie sich zu verdeutlichen, dass alle Analysenund röumliche Modelle auf den Grundannahmen dieser Übung basieren. Das heisst es kommt maßgeblich auf Ihre konzepteionellen oder theoriegeleiteten Vorstellungen an welche Nachbarschaft, welches Maß an Nähe und somit auch welche räumlichen Korrelationen zustande kommen:

* das Skript schrittweise durchzugehen. 
* verscuhen Sie die Inhalte aus dem Statistikkurs in ihrer  räumlichen Wirkung und Visualisierung nachzuvollziehen.
* spielen Sie mit den Einstellungen und lernen Sie schrittweise die Handhabung von R
* Identifizieren Sie die im "Vorbeigehen" gemachten Erläuterungen wie Daten zu plotten sind oder Was es mit Georefrenzieren auf sich hat. Vieles ist in den Kommentarzeilen untergebracht.

