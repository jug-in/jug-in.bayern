---
title: Java Treff
subtitle: jeden 2. Dienstag im Monat
comments: false
---

Der Java-Treff ist unsere monatliche Veranstaltung mit Vorträgen, Austausch und Diskussion. Auch das leibliche Wohl kommt an unserem bewirtetem Veranstaltungsort - der [Sportgaststätte des SV Zuchering](https://goo.gl/maps/WdFPbCwjdqWQr5eUA) - nicht zu kurz.

#### Java-Treff im März
### 10.03.2020 - 18:30 Uhr

Vertreter der Firma und des Projektes MicroStream besuchen uns und haben eine sehr heiße Technologie im Gepäck:

# Ultraschnelle Java In-Memory Datenbankanwendungen mit MicroStream

Dieser Vortrag zeigt auf, warum heutige Datenbanksysteme und In-Memory Technologien nicht zu Java passen, welche Folgen deren Einsatz zwangsläufig hat und stellt danach mit MicroStream den ersten, speziell für Java entwickelten Java-nativen Ansatz zur Datenspeicherung und dessen enormen Vorteilen für Java-Entwickler vor. 

Java ist die beste Technologie zur Entwicklung von unschlagbar schnellen Datenbankanwendungen. Mit Objektgraphen bietet Java die perfekte Datenstruktur. Selbst komplexeste Datenstrukturen lassen sich damit abbilden. Dazu bietet Java mit Streams eine mächtige, typsichere Abfragesprache. Das Durchsuchen selbst komplexester Objektgraphen mit extrem großen Datenmengen dauert dank Parallel Streams und JIT-Compiler meist nur wenige Millisekunden, oft sogar nur Mikrosekunden – im Schnitt bis zu 1.000 Mal schneller als heutige SQL/NoSQL Datenbanksysteme und sogar bis zu 100 Mal schneller als direkte Cache-Zugriffe. Das einzige was Java nicht selbst kann, ist Daten speichern.

Um Daten persistent zu speichern, setzen wir bekanntlich externe Datenbanksysteme ein, die jedoch völlig anders als Java funktionieren und uns andere Datenstrukturen, andere Abfragesprachen sowie spezielle Datenbankfunktionen (Transaktionen, Trigger, Stored Procedures, User-Verwaltung, Session-Management, Import/Export Schnittstellen) vorgeben, die wir dann verwenden sollen. Die Folge: aufwändige Mappings, enorm hohe Komplexität, langsame Abfragen, langsame Cache-Zugriffe, permanente Performance Issues.

MicroStream ist der erste Java-native Ansatz zur Datenspeicherung in Java. Mit MicroStream lassen sich erstmals beliebige Java Objektgraphen nativ speichern, d.h. genauso wie diese von der JVM im RAM verwaltet werden. Kein externes Datenbanksystem mehr. Kein aufwändiges Mapping mehr. Kein ORM-Framework mehr. Kein zusätzlicher Data-Cache mehr. Das Ergebnis ist eine ultraschnelle Pure-Java In-Memory Datenbank-Applikation, ultraschnelle Abfragen bis zu 1.000 Mal schneller als heutige Datenbanksysteme und bis zu 100 Mal schneller als jeder Data-Cache. Alles Pure Java. Und die Implementierung ist simpel.

Im Rahmen einer spannenden Live-Demo demonstrieren wir, wie schnell Java im Vergleich zu heutigen Datenbanksystemen und In-Memory-Technologien ist, wie sich unterschiedliche JVMs und Garbage-Collectors auf die Performance auswirken und zeigen anhand von Code, wie die Entwicklung ultraschneller In-Memory Datenbank-Applikationen mit Pure Java und MicroStream funktioniert.

Im Anschluss an den Vortrag sind ausführliche Q&A und Diskussionen möglich und erwünscht.

### Speaker:

* Markus Kett, CEO MicroStream
* Florian Habermann, CTO MicroStream
* Christian Kümmel, Developer Advocate MicroStream

### Infos & Download:

www.microstream.one

### Getting started:

https://manual.docs.microstream.one/data-store/getting-started

---

#### Java-Treff im April
### 14.04.2020 - 18:30 Uhr

Alexander Reelsen von https://www.elastic.co/ besucht uns und bringt die folgenden Vorträge mit:

# Testing Elasticsearch - Von Unit- über Integrationstests bis hin zum Release

Testen muss jeder, egal ob kleiner Webshop oder eine beliebte verteilte Suchmaschine wie Elasticsearch. In diesem Vortrag geht es um die verschiedenen Teststrategien innerhalb von Elasticsearch, angefangen bei Unit Tests und Integration Tests, der Verwendung von randomized Testing, automatisiertes Testen der produzierten Artefakte. Des Weiteren wird die CI Infrastruktur beleuchtet, und was schlußendlich passiert, wenn eine neue Elasticsearch Version veröffentlicht wird.
Der Fokus des Vortrags liegt auf größtmöglicher Automatisierung jedweder Prozesse im Lebenszyklus von Elasticsearch und zeigt auch, welche Teile besonders hart zu automatisieren sind.

# Elasticsearch - Securing A Search Engine While Maintaining Usability

Elasticsearch - das Herz des Elastic Stacks - ist als verteilte, skaliarbare Volltextsuchmaschine und für Analytics bekannt.  Zehntausende Elasticsearch Instanzen & Cluster laufen weltweit, so dass trotz aller neuen Features in jedem Release auch über das Thema Sicherheit nachgedacht werden muss. Dieser Vortrag geht auf unterschiedliche Aspekte von Elasticsearch im Bereich Sicherheit ein, zeigt eine Einführung in den Java Security Manager sowie dessen Verwendung in Elasticsearch. Des Weiteren werden auch native Betriebssystem Sicherheitskomponenten wie seccomp erläutert und wie diese in Java Anwendungen verwendet werden können. Ziel des Vortrags ist es nicht nur, auf einige Details in Elasticsearch einzugehen, sondern vielmehr die Zuhörer dazu zu bekommen, diese Themen auch in ihren eigenen Apps zu berücksichtigen.