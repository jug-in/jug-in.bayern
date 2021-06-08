---
title: Ultraschnelle Java In-Memory Datenbankanwendungen mit MicroStream
date: 2020-03-10
---

### Markus Kett, CEO MicroStream

Dieser Vortrag zeigt auf, warum heutige Datenbanksysteme und In-Memory Technologien nicht zu Java passen, welche Folgen deren Einsatz zwangsläufig hat und stellt danach mit MicroStream den ersten, speziell für Java entwickelten Java-nativen Ansatz zur Datenspeicherung und dessen enormen Vorteilen für Java-Entwickler vor. 

Java ist die beste Technologie zur Entwicklung von unschlagbar schnellen Datenbankanwendungen. Mit Objektgraphen bietet Java die perfekte Datenstruktur. Selbst komplexeste Datenstrukturen lassen sich damit abbilden. Dazu bietet Java mit Streams eine mächtige, typsichere Abfragesprache. Das Durchsuchen selbst komplexester Objektgraphen mit extrem großen Datenmengen dauert dank Parallel Streams und JIT-Compiler meist nur wenige Millisekunden, oft sogar nur Mikrosekunden – im Schnitt bis zu 1.000 Mal schneller als heutige SQL/NoSQL Datenbanksysteme und sogar bis zu 100 Mal schneller als direkte Cache-Zugriffe. Das einzige was Java nicht selbst kann, ist Daten speichern.

Um Daten persistent zu speichern, setzen wir bekanntlich externe Datenbanksysteme ein, die jedoch völlig anders als Java funktionieren und uns andere Datenstrukturen, andere Abfragesprachen sowie spezielle Datenbankfunktionen (Transaktionen, Trigger, Stored Procedures, User-Verwaltung, Session-Management, Import/Export Schnittstellen) vorgeben, die wir dann verwenden sollen. Die Folge: aufwändige Mappings, enorm hohe Komplexität, langsame Abfragen, langsame Cache-Zugriffe, permanente Performance Issues.

MicroStream ist der erste Java-native Ansatz zur Datenspeicherung in Java. Mit MicroStream lassen sich erstmals beliebige Java Objektgraphen nativ speichern, d.h. genauso wie diese von der JVM im RAM verwaltet werden. Kein externes Datenbanksystem mehr. Kein aufwändiges Mapping mehr. Kein ORM-Framework mehr. Kein zusätzlicher Data-Cache mehr. Das Ergebnis ist eine ultraschnelle Pure-Java In-Memory Datenbank-Applikation, ultraschnelle Abfragen bis zu 1.000 Mal schneller als heutige Datenbanksysteme und bis zu 100 Mal schneller als jeder Data-Cache. Alles Pure Java. Und die Implementierung ist simpel.

Im Rahmen einer spannenden Live-Demo demonstrieren wir, wie schnell Java im Vergleich zu heutigen Datenbanksystemen und In-Memory-Technologien ist, wie sich unterschiedliche JVMs und Garbage-Collectors auf die Performance auswirken und zeigen anhand von Code, wie die Entwicklung ultraschneller In-Memory Datenbank-Applikationen mit Pure Java und MicroStream funktioniert.

Im Anschluss an den Vortrag sind ausführliche Q&A und Diskussionen möglich und erwünscht.

### Infos & Download:

www.microstream.one

### Getting started:

https://manual.docs.microstream.one/data-store/getting-started
