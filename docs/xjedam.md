# Powierzchnia lasów i użytków rolnych - Ryszard Madejski

Dane pobrane z [GUS](http://www.stat.gov.pl/bdl/app/strona.html?p_name=indeks), użyto 2 zestawów danych:
* [lasy.csv](http://sigma.ug.edu.pl/~rmadejski/lasy.csv)
* [uzrolne.csv](http://sigma.ug.edu.pl/~rmadejski/uzrolne.csv)

## Opis
Obydwa w/w zestawy danych zostały zaimportowane do narzędzia Google-Refine, następnie mocno oczyszczone/zmienione:
* Poprawiłem błędy w importowaniu,
* Usunąłem nadmiarowe kolumny,
* Utworzyłem nową kolumnę łączącą region i rok pomiaru,
* Na podstawie w/w kolumny połączyłem dane dot. powierzchni lasów i użytków rolnych,
* Usunąłem nadmiarowe wiersze "śmieciowe",
* Zmieniłem typy wartości powierzchni na numeryczne
* Wyexportowałem dane w postaci json do pliku tekstowego
* Zaimportowałem dane używając mongoimport: `mongoimport --db lasy_uzytki --collection pow_pol --type json --file lasy-rolne.json`

## Przykładowy JSON po imporcie.
```
{
	"_id" : ObjectId("514066f8788baac425452484"),
	"kod" : 2000000000,
	"jednostka_terytorialna" : "Region poludniowy",
	"lata" : 2003,
	"jednostka_miary" : "ha",
	"wartosc_lasy" : 841204,
	"wartosc_rolne" : 1422745
}
```

[Exportowane dane](/data/json/dane_xjedam.json)

## Agregacje

Postanowiłem wykonać agregacje uzywając driverów casbah do scali. Najpierw trzeba pobrać sterownik, z czym niestety było trochę szarpania, gdyż jedne z repozytoriów przestały funkcjonować. Na szczęście uzupełniając odpowiednio plik `build.sbt` sterowniki pobierają się bez problemów:
```Scala
libraryDependencies ++= Seq(
    "com.mongodb.casbah" % "casbah_2.9.0" % "2.2.0-SNAPSHOT"
)

resolvers += "Sonatype OSS Snapshots" at "https://oss.sonatype.org/content/repositories/snapshots"
```
Następnie łączymy się z odpowiednią bazą danych (u mnie baza lokalna, na domyslnym porcie, kolekcja: `lasy_uzytki`):
```Scala
val mongoDb =  MongoConnection()("lasy_uzytki")
```
Jeszcze aby ułatwić sobie życie zdefiniujemy funkcję wykonującą nasze agregacje:
```Scala
def aggregationResult(collectionName:String, pipeline: MongoDBList) = {
    mongoDb.command(MongoDBObject("aggregate" -> collectionName, "pipeline" -> pipeline)).get("result")
}
```
A teraz pozostaje już napisać agregacje np:

### Calkowite powierzchnie na przestrzeni 10 lat
```Scala
val aggr1 = new MongoDBList()
    aggr1 += MongoDBObject("$match" -> MongoDBObject.empty)
    aggr1 += MongoDBObject("$group" -> MongoDBObject(
        "_id" -> "$lata",
        "calk_pow_lasy" -> MongoDBObject("$sum" -> "$wartosc_lasy"),
        "calk_pow_rolne" -> MongoDBObject("$sum" -> "$wartosc_rolne")
        ))
    aggr1 += MongoDBObject("$sort" -> MongoDBObject("_id" -> 1))
```
Część wyniku:
```
{ "_id" : 1995 , "calk_pow_lasy" : 8821818 , "calk_pow_rolne" : 18622227}
{ "_id" : 1996 , "calk_pow_lasy" : 8860687 , "calk_pow_rolne" : 18608157}
{ "_id" : 1997 , "calk_pow_lasy" : 8880908 , "calk_pow_rolne" : 18457049}
```
Wykres(ciemnoniebieskie lasy, jasnoniebieskie użytki rolne):

![Powierzchnie w latach](https://raw.github.com/nosql/data-refine/master/images/xjedam_aggr1.png)

Otrzymany z Google chart api:
```
http://chart.googleapis.com/chart
  ?chxl=0:|1995|1996|1997|1998|1999|2000|2001|2002|2003|2004|2005
  &chxr=a
  &chxt=y,x
  &chbh=a
  &chs=640x320
  &cht=bhg
  &chco=4D89D9,C6D9FD
  &chds=a
  &chd=t:8821818,8860687,8880908,8918094,8970212,9003917,9027800,9058062,9100468,9126723,9172558|
  18622227,18608157,18457049,18442702,18434683,18413249,18392428,18374093,18280092,18204355,18208403
  &chma=100
  &chtt=pow+lasy+i+uzytki+rolne
```

### Powierzchnie lasów i użytków rolnych w poszczególnych regionach
```Scala
val aggr2 = new MongoDBList()
    aggr2 += MongoDBObject("$match" -> MongoDBObject.empty)
    aggr2 += MongoDBObject("$group" -> MongoDBObject(
        "_id" -> "$jednostka_terytorialna",
        "sred_pow_lasy" -> MongoDBObject("$avg" -> "$wartosc_lasy"),
        "sred_pow_rolne" -> MongoDBObject("$avg" -> "$wartosc_rolne")
        ))
    aggr2 += MongoDBObject("$sort" -> MongoDBObject("sred_pow_lasy" -> 1))
```
Część wyniku:
```
{ "_id" : "Region poludniowo-zachodni" , "sred_pow_lasy" : 826767.5454545454 , "sred_pow_rolne" : 1746894.5454545454}
{ "_id" : "Region poludniowy" , "sred_pow_lasy" : 836045.8181818182 , "sred_pow_rolne" : 1483441.2727272727}
```

Wykres(ciemnoniebieskie lasy, jasnoniebieskie użytki rolne):

![Uśrednione powierzchnie w regionach](https://raw.github.com/nosql/data-refine/master/images/xjedam_aggr2.png)

Otrzymany z Google chart api:
```
http://chart.googleapis.com/chart
  ?chxl=0:|pol-zach|poludniowy|centralny|polnocny|wschodni|polno-zach
  &chxr=a
  &chxt=y,x
  &chbh=a
  &chs=640x320
  &cht=bhg
  &chco=4D89D9,C6D9FD
  &chds=a
  &chd=t:826768,836046,1159931,1783246,2133336,2255332|
  1746895,1483441,3649826,3375562,4587297,3560382
  &chma=100
  &chtt=pow+sred+lasy+i+uzytki+rolne
```

###Kiedy było najwięcej lasów w danych regionach
```Scala
val aggr3 = new MongoDBList()
    aggr3 += MongoDBObject("$match" -> MongoDBObject.empty)
    aggr3 += MongoDBObject("$sort" -> MongoDBObject("jednostka_terytorialna" -> 1, "wartosc_lasy" -> -1))
    aggr3 += MongoDBObject("$group" -> MongoDBObject(
        "_id" -> "$jednostka_terytorialna",
        "rok" -> MongoDBObject("$first" -> "$lata"),
        "max_lasy" ->  MongoDBObject("$first" -> "$wartosc_lasy")
        ))
```
Część wyniku:
```
{ "_id" : "Region poludniowy" , "rok" : 2005 , "max_lasy" : 842815}
{ "_id" : "Region polnocny" , "rok" : 2005 , "max_lasy" : 1836406}
```

Wykres:

![W którym roku pow lasów największa](https://raw.github.com/nosql/data-refine/master/images/xjedam_aggr3.png)

Otrzymany z Google chart api:
```
http://chart.googleapis.com/chart
  ?chxl=0:|pol-zach - 2005|poludniowy - 2005|centralny - 2005|polnocny - 2005|wschodni - 2005|polno-zach - 2005
  &chxr=a
  &chxt=y,x
  &chbh=a
  &chs=640x320
  &cht=bhg
  &chco=4D89D9,C6D9FD
  &chds=a
  &chd=t:845239,842815,1177330,1836406,2173353,2297415
  &chma=100
  &chtt=max+pow+lasy
```

Oczywiście aby wyświetlić nasze dane wywołujemy naszą funkcję:
```Scala
aggregationResult("pow_pol", aggr3) match {
    case list: BasicDBList => list.foreach(println(_))
    case _ => println("Error D:")
}
```
## Odczucia/Wnioski
Chciałem sprawdzić, czy wzrost/spadek powierzchni użytków rolnych jest powiązany z powierzchnią lasów. Faktycznie wydaje się,że taka zależność zachodzi, chociaż zapewne jest to bardziej skomplikowany temat ;)
Niektóre operacje transformacji danych nie były łatwe, na szczęście dość duża ilość materiałów nt. Google-Refine umożliwiła w miarę szybkie zapoznanie się z narzędziem i poznanie różnych sposobów osiągnięcia porządanego efektu. Przyznaję że google-refine to dość ciekawe narzędzie, gdyż przy ręcznej obróbce danych(na bazie danych) łatwo zniszczyć posiadane dane, natomiast tutaj zawsze jest możliwość cofnięcia błędnych kroków, co robiłem nieraz.
