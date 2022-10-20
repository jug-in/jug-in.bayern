---
title: Analytical data API architecture using PostgreSQL magic
date: 2022-10-11
---

Analytisch aufbereitete Daten als transaktionsfähige API zur Verfügung zu stellen kann eine Herausforderung darstellen.
Besonders große, sich regelmäßig ändernde Datenmengen für eine echtzeitfähige Abfrage aktuell zu halten, bedarf einiger Tricks.

**Thomas Wolf** _(MediaMarktSaturn Technology)_ erläuterte uns, wie sein Team dies auf der Google Cloud und mit ganz viel PostgreSQL-Magie bewerkstelligt, die Konzepte sind jedoch auf andere Plattformen übertragbar.
