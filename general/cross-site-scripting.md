---
layout: guide
title: "Cross-Site-Scripting"
group: "Allgemein"
creator: Manko10
author:
    -   name: Manko10
        profile: 1139

    -   name: nikosch
        profile: 2314

    -   name: mermshaus
        profile: 15041

inhalt:
    -   name: "Voraussetzungen"
        anchor: voraussetzungen
        simple: ""

    -   name: "Code Injection allgemein"
        anchor: code-injection-allgemein
        simple: ""

    -   name: "Durchführung einer XSS-Attacke"
        anchor: durchfhrung-einer-xss-attacke
        simple: ""

    -   name: "Konsequenzen einer XSS-Attacke"
        anchor: konsequenzen einer xss-attacke
        simple: ""

    -   name: "Schutz vor Cross-Site-Scripting"
        anchor: schutz-vor-cross-site-scripting
        simple: ""

entry-type: in-discussion
---

Beim **Cross-Site-Scripting** (oft auch XSS) handelt es sich um eine
Angriffstechnik auf die Besucher der Webseite mittels
[JavaScript](http://php-de.github.io/general/javascript.html) oder anderer
client-basierter Skriptsprachen. Dabei wird sich eine unzureichende Prüfung der
eingegebenen Daten zunutze gemacht.

Ein verwandtes Problem ist das Einschleusen anderer clientverarbeiteten
Sprachteile (Code Injection) wie HTML-Code oder CSS-Formatierungsangaben. Das
Spektrum reicht hier von der Einbringung eigener Inhalte bis zur
Neuformatierung, damit schlimmstenfalls bis zur Unbenutzbarkeit der Seite.
Neben konkreten Schäden für den Nutzer kann ein erheblicher Imageschaden für
den Seitenbetreiber entstehen.



### Voraussetzungen

Die Manipulation clientbasierter Sprachen funktioniert dann, wenn Teilinhalte
dynamisch auf einem Server erzeugt und in die HTML Ausgabe (oder jede andere
interpretierte Ressource) geschrieben werden. Da der Seitennutzer erst diese
Ausgabe direkt wahrnimmt, basieren XSS & Co immer auf einer Sicherheitslücke
auf dem Server.

Die zweite Komponente ist der zeitliche Aspekt. Ein Angriff über XSS soll ja
nicht den Manipulierenden selbst, sondern Besucher dieser Seite treffen.
Deshalb ist ein erfolgreicher XSS Angriff nur möglich, wenn der Schadcode in
irgendeiner Form gespeichert und Teil der öffentlichen Ausgabe der Website
wird. Aus dem selben Grund kann eine dynamische Seitenmanipulation
überJavascript/DOM nicht zu Cross Side Scripting Attacken gezählt werden.



### Code Injection allgemein

Dynamische Websiteprogrammierung fußt auf einer Kombination von statischen HTML
Codeelementen und darin eingefügten Inhaltsteilen. Das entstehende Produkt wird
als HTML Quellcode vom Webserver ausgeliefert (vgl.
[Was_ist_PHP](http://php-de.github.io/general/was-ist-php.html)). Daraus
entsteht ein ungewöhnlicher Effekt, wenn man die Möglichkeit der Flexibilität
zu einseitig betrachtet.

~~~ php
<h1><?php echo $myHeader; ?></h1>
~~~

~~~ php
<p title="<?php echo $myTitle; ?>">
~~~

~~~ php
<input type="text" name="test" value="<?php echo $myValue; ?>">
~~~

Direkte Angabe eines dynamischen Wertes und zweier dynamischer
Attributzuweisungen. Denkbar wären folgende Angaben, die aus Nutzereingaben
entstehen könnten:

~~~ php
$myHeader = 'Überschrift 1</h1>
<p>Ein zusätzliches Kapitel</p>
<h1>Überschrift 2';
~~~

~~~ php
$myTitle = '" style="background:url(http://badserver/badimage.jpg);';
~~~

~~~ php
$myValue = '"><input type="text" name="badtest" value="bad value';
~~~

Als Resultat ergibt sich in allen drei Fällen ein völlig neuer Kontext also
angedacht:

~~~ html
<h1>Überschrift 1</h1>
<p>Ein zusätzliches Kapitel</p>
<h1>Überschrift 2</h1>
~~~

Statt einer Überschrift wird eine zweite Headline und sogar ein zusätzlicher
Absatz eingefügt.

~~~ html
<p title="" style="background:url(http://badserver/badimage.jpg);">
~~~

Statt eines relativ harmlosen title Attributs wird eine style Attribut gesetzt
und der Absatz neu formatiert.

~~~ html
<input type="text" name="test" value=""><input type="text" name="badtest" value="bad value">
~~~

Statt ein Formfeld auszufüllen wird das Tag geschlossen und ein weiteres
Eingabefeld in den Quelltext geschrieben.

Code Injection für sich ist zwar ein vom Entwickler nicht vorgesehener Effekt,
schadet aber vornehmlich erst einmal dem Manipulator selbst. Der veränderte
Browserquelltext ist damit zunächst nicht schwerwiegender, als wäre er auf
nachträglich auf dem Client auf einen andere Weise, bspw. über einen Proxy oder
ein Browserplugin wie Greasemonkey, verändert worden.

Wie bereits einleitend beschrieben, ergibt sich die eigentlich Gefahr dann,
wenn so eingeschleuster Code – vornehmlich JavaScript – gespeichert und auch
für andere Nutzer ausgegeben wird.



### Durchführung einer XSS-Attacke

Als Beispiel für einen erfolgreichen XSS-Angriff soll ein Gästebuch dienen.
Werden die Daten unzureichend validiert, ist es dem Angreifer ein Leichtes,
schädlichen Code einzuschleusen.

~~~ html
<script type="text/javascript">alert('XSS!');</script>
~~~

Wird dieser Code nicht validiert (und beispielsweise mit `htmlspecialchars()`
behandelt), so ist er für jeden Besucher des Gästebuchs sichtbar und kommt
demnach zur Ausführung. Dem Besucher wird eine `alert`-Box mit dem Inhalt
„XSS!“ angezeigt.

#### Problem magic_quotes_gpc

Ist die Direktive
[magic_quotes_gpc](http://php-de.github.io/general/php-ini.html#magicquotesgpc)
in der [php.ini](http://php-de.github.io/general/php-ini.html) eingeschaltet,
wird der obige Code u.U. nicht funktionieren, da die Stringbegrenzer `"` und
`'` nicht funktionieren werden. Dies stellt jedoch kein Problem dar, da die
Angabe `type="text/javascript"` für die Ausführung in den meisten Browsern
nicht vonnöten ist. Ebenso können Strings in JavaScript auch auf andere Weise
dargestellt werden, indem man sich die Syntax regulärer Ausdrücke zunutze
macht.

~~~ html
<script>alert(/XSS!/.source);</script>
~~~

Dieser Code enthält keine regulären Stringbegrenzer und ist dennoch
funktionstüchtig.



### Konsequenzen einer XSS-Attacke

Das obige Beispiel zum Einschleusen schädlichen Codes ist nur eine milde
Variante dessen, was möglich ist. Möglich ist u.a. Folgendes:

* Umleiten des Besuchers auf eine andere Seite
* Auslesen der Zugangsdaten, die evtl. durch den Browser in Login-Felder eingefüllt werden
* Verunsicherung internet-unerfahrener Besucher (Beispiel: alert-Box als Betriebssystemmeldung tarnen)
* Manipulation der Internetseite
* Abfangen von Tastatureingaben

Cross Site Scripting stellt also, wie unschwer zu erkennen ist, eine
unmittelbare Gefahr für den Besucher der Seite dar. Eine Gefahr, die oft
unterschätzt wird. Nicht selten wird der Benutzer auf eine andere Seite
umgeleitet, die das gleiche Aussehen hat, aber mit dem Zweck erstellt wurde,
Zugangsdaten oder E-Mail-Adressen der Benutzer herauszufinden.



### Schutz vor Cross Site Scripting

#### Als Webmaster

Als Webmaster bzw. Programmierer besitzt man hier besondere Verantwortung, da
das Wohl der eigenen Besucher von den getroffenen Sicherheitsvorkehrungen
abhängt.

Es ist unumgänglich, alle vom Benutzer eingegebenen Daten als potenziell
gefährlich einzustufen. Um Cross Site Scripting zu verhindern, ist es
notwendig, die `<script>`-Tags unschädlich zu machen. Oftmals wird der Fehler
gemacht, diese einfach durch einen regulären Ausdruck zu entfernen, doch ist es
damit nicht getan, da in Browsern wie dem Internet-Explorer auch solches
möglich ist:

~~~ html
<img src="javascript:alert('XSS!')">
<img src="javascript:alert(/XSS!/.source)">
~~~

#### Entfernen des Markups

Eine Entfernung der `<script>`-Tags mindert das Risiko also nur, verhindert
aber noch lange kein Cross-Site-Scripting. Besser ist es, *alle*
HTML-Auszeichnungen in Parameterinhalten zu zerstören. Um den Benutzern dennoch
Formatierungsmöglichkeiten zu bieten, können sog. BB-Codes oder ähnliche
Textauszeichnungsformate eingeführt werden.

Eine Möglichkeit, XSS zu verhindern, ist die Entfernung aller Tags mit
`strip_tags()`. Dies ist die empfohlene Vorgehensweise, wenn es sich um Daten
handelt, in denen HTML nichts zu suchen hat (bspw. Angabe des Namens, der
Adresse oder der Telefonnummer).

<div class="alert alert-info">

<strong>Hinweis:</strong>

Besser als eine Anpassung von Daten an eine erlaubte Zeichenmenge kann ein
frühzeitiger Abbruch bei der Eingabevalidierung sein. Gegebenenfalls ist dort
eine Unterscheidung sinnvoll, zwischen eindeutig angriffcharakteristischen
Angaben und Fehldaten, die bspw. durch Tippfehler entstehen.

</div>

#### Maskieren des Markups

Die zweite Möglichkeit ist, HTML-eigene Zeichen zu maskieren. Dabei werden alle
HTML-Steuerzeichen (`<`, `>`, `"`, `&` etc.) durch ihre Entitäten ersetzt
(`&lt;`, `&gt;`, `&quot;`, `&amp;` etc.).

Dies ist vor allem dann notwendig, wenn Benutzer HTML-Codes (oder einzelne
Zeichen aus dem HTML Zeichenvorrat) posten können sollen, es dabei aber nicht
zur Ausführung oder ungewünschten Nebeneffekten kommen soll. Beispiele dafür
sind Beiträge in Foren und Mailinglisten oder sogenannte ASCII Art Einträge wie
Signaturen und dergleichen.

PHP bietet zur Maskierung von HTML-eigenen Zeichen neben allgemeinen Befehlen
zur Stringersetzung die beiden speziellen Funktion `htmlspecialchars` und
`htmlentities` an. Während erstgenannte versucht, möglichst viele Sonderzeichen
(so auch deutsche Umlaute) in HTML-Code umzusetzen, beschränkt sich
`htmlentities` auf syntaxrelevante Sprachbestandteile und ist für einen Schutz
gegen XSS ausreichend.

#### Softwaregestütztes Whitelisting

Gefällt einem keine der oben genannten Lösungen, so kann man noch den Einsatz
von HTML-Validierungswerkzeugen wie [HTML Purifier](http://htmlpurifier.org/)
in Betracht ziehen. Diese bieten eine extrem hohe Sicherheit gegen
Cross-Site-Scripting, doch sollte man sich immer des kleinen Restrisikos, das
durch neu entdeckte Sicherheitslücken in Browsern entstehen kann, bewusst sein.

<div class="alert alert-danger">

<strong>Achtung!</strong>

Häufig gemachter Fehler: Es kann nicht oft genug gesagt werden, dass
<strong>alle</strong> vom User kommenden Daten bösartiger Natur sein können,
nicht nur <a
href="http://php-de.github.io/request-handling/gpc.html">GPC-Werte</a>. Auch
das <code>$_SERVER</code>-Array gehört zur Kategorie der gefährlichen
Datenquellen und muss unbedingt validiert werden.

</div>

#### Als Benutzer

Auch als Benutzer gibt es einige Dinge, mit denen man sich vor
Cross-Site-Scripting schützen kann. Zunächst sollte man die automatische
Passwortspeicherung des Browsers abschalten, sofern diese bei Wiedererkennung
der Login-Seite die Daten automatisch in die dafür vorgesehenen Felder füllt.
Andere Passwortmanager wie Operas *Wand* oder das Add-on *Secure Login* für
Firefox bieten hier mehr Sicherheit, da die Daten nicht direkt in die Felder
gefüllt werden.

Dazu sollte man auf Login- sowie anderen Seiten mit sensiblen Daten sämtliche
Skript-Sprachen ausschalten, bevor man mit der Dateneingabe beginnt.

Vor der Eingabe sensibler Daten ist außerdem unbedingt zu prüfen, ob man sich
noch auf der korrekten Seite befindet oder ohne sein Wissen auf eine Fremdseite
geleitet wurde.
