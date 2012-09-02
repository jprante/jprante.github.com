
---
layout: post
category : lessons
title: "Warum reicht XML nicht aus?"
tags : [intro, beginner, tutorial, semanticweb]
comments: false
---
{% include JB/setup %}

XML-Tags sind aus Sicht des Semantic Web nicht besser als natürliche Sprache. Denn auch sie sind Wörter, die mehrdeutig sein können, und deren Beziehungen zueinander nicht eindeutig definiert sind. Im folgenden ersten Beispiel haben für den des Deutschen mächtigen Lesenden die Tags sehr wohl eine Bedeutung, für eine Maschine hingegen nicht. Eine Maschine versteht genau so viel wie im zweiten Beispiel:

	<Buch reihe="Historische Klassiker">
	    <Titel>Faust</Titel>
	    <Autor>Goethe</Autor>
	</Buch>

	<iorvmior kekd="Historische Klassiker">
	    <wrtd>Faust</wrtd>
	    <ksckr>Goethe</ksckr>
	</iorvmior>

Tags können also nur dann für den kundigen Betrachter eine Bedeutung haben, wenn sie geschickt gewählt werden. Für Maschinen sind jedoch auch diese vollkommen ohne Bedeutung (ohne Semantik). XML leistet es nicht, eine Maschine in die Lage zu versetzen, neues Wissen aus bekannten Fakten herzuleiten. Mit dem Attribut-Text "Historische Klassiker" kann eine Maschine nichts anfangen - dass es ein Verweis auf eine Reihe sein soll, ist nicht definiert.

Ein weiteres Beispiel. Für den des Deutschen kundigen Betrachter sind folgende Tags mit einer Bedeutung verknüpft:

	<Apfel>lecker</Apfel>
	<Birne>schmeckt auch</Birne>

wogegen eine Maschine mit den Tags "Apfel" und "Birne" nichts anfangen kann, da in ihnen selbst nicht die Beziehung zueinander eingespeichert ist. Ganz davon abgesehen, lösen in der Maschine die Tags keine weiteren Assoziationen aus, es können also maschinell keine Schlussfolgerungen aus den beiden Tags gezogen werden.

Die Folge ist, dass für den Austausch von XML-Dokumenten umfangreiche Dokumentationen erstellt werden müssen, welche die Bedeutung und die Beziehung der verwendeten Tags definieren. Diese Dokumentationen werden lediglich von Anwendungsentwicklern zur Programmierung verwendet, für die Anwender ist die Dokumentation nicht interessant. Daher ist XML für den Austausch von Informationen nur ein "Containerformat" - ohne die Bedeutung der Tags zu kennen, kann auch ein Programmierer mit XML nichts anfangen.

Ein Ziel des Semantic Web ist, diese Dokumentationen überflüssig zu machen, indem die Bedeutungen und die Beziehungen der Tags aus den Tags selbst hervorgehen.

Wie weist das Semantic Web Tags Bedeutungen und Beziehungen zu?
---

![Buch](/assets/images/semanticweb-buch.png)

Aus Sicht der Informatik wird RDF, das dem Semantic Web zugrundeliegende Datenmodell, als Graph kodiert. Ausgangspunkt ist das Konzept der Aussagenlogik. Eine Aussage, die als Subjekt/Prädkat/Objekt-Tripel formuliert ist, lässt sich z.B. in primitiver Form mit Hilfe von zwei Knoten und einer Kante in einem Graphen darstellen. Das ist ein abstraktes Modell für Faktensammlungen, wie sie beispielsweise in einem Bibliothekskatalog vorkommen. 

Einer *Ressource* (gleichbedeutend mit einer verzeichneten Medieneinheit in einem Katalog) werden also bestimmte Eigenschaften zugesichert. Hinzu kommt die Möglichkeit, in einer Aussage auf eine andere Aussage in einem anderen Graphen Bezug nehmen und somit eine Beziehung zu anderen Ressourcen aufstellen zu können, die ausserhalb des aktuellen Kontexts existieren. Deren Existenz braucht erst später überprüft werden - man weiss nicht, ob die Ressource existiert oder nicht (man nennt dies auch *open world assumption*, diese erlaubt es, dass RDF schemafrei ist, also ohne Dokumentation auskommt).

Beispiel für drei Aussagen (umgangssprachlich):

	"Das Buch hat Goethe geschrieben."
	"Das Buch hat den Titel 'Faust'".
	"Das Buch ist in der Reihe 'Historische Klassiker' erschienen."

In RDF (informelle Notation, keine URIs):

	Subjekt:BuchID -> Prädikat:hatAutor -> Objekt:"Goethe"
	Subjekt:BuchID -> Prädikat:hatTitel -> Objekt:"Faust"
	Subjekt:BuchID -> Prädikat:gehörtzurReihe -> Objekt:ReiheHistorischeKlassikerID

wobei das Symbol ``->`` für eine Kante im Graphen steht und Subjekt, Prädikat, und Objekt Knoten im Graphen sind. Manche Knoten sind mit Bezeichner oder IDs versehen, also ausgezeichnet, andere wiederum enthalten nur Werte (oder Literale, im Beispiel in Anführungsstrichen). In diesem Beispiel wird in der dritten Aussage die Beziehung vom Objekt ``BuchID`` zum Objekt ``ReiheHistorischeKlassikerID`` hergestellt, erkennbar am Prädikat ``gehörtzurReihe``.

Für die Anwendung der Aussagenlogik auf Ressourcen im World Wide Web hat man sich auf *Uniform Resource Identifier* (URIs) als IDs geeinigt. In den obigen Besipielen wurden keine URIs verwendet. URIs werden zur Identitätsgebung für Objekte im Semantic Web eingesetzt,  vergleichbar etwa zum Vorgang, wie Signaturen für Bücher und andere Medien im Regal vergeben werden. Nur wirken URIs global, haben eine durch die Internet Engineering Task Force [festgelegte Syntax](http://www.ietf.org/rfc/rfc2396.txt), und weisen eingebaute Mechanismen auf, die helfen, Eindeutigkeit zu organisieren (Hierarchien, Schemata, Komponenten).

RDF-Beispiel mit URI-Syntax in spitzen Klammern:

	<subject:BuchID> <predicate:hasAuthor> "Goethe"
	<subject:BuchID> <predicate:hasTitle> "Faust"
	<subject:BuchID> <predicate:isPartOfSeries> <subject:ReiheHistorischeKlassikerID>

Bei Verwendung von URIs wird deren zentrale Rolle im Semantic Web deutlich: sie dienen im Graphen sowohl als eindeutige Bezeichner einer Ressource, als auch der Aufzählung von Prädikaten und der Verweisung auf andere Ressourcen. Es ist üblich, englischsprachige URIs zu formulieren, man kann jedoch auch mit Hilfe von *Internationalized Resource Identifiers* ([IRIs](http://www.ietf.org/rfc/rfc3987.txt)) beliebige Sprachen kodieren.

Veranschaulichung von RDF-Graphen
---

Ein RDF-Graph kann auf viele Arten veranschaulicht werden - dieser Prozess wird Serialisierung des Graphen genannt, es werden dabei alle Knoten und Kanten in einer algorithmischen Reihenfolge durchlaufen und ausgegeben. Im obigen Beispiel mit URI-Syntax ist dies bereits geschehen - es wurde zeilenweise serialisiert.

Die erste bekannte frühe Form einer RDF-Repräsentation war [RDF/XML](http://www.w3.org/TR/REC-rdf-syntax/) und wurde für die Spezifikation von RDF verwendet. Sie ist jedoch kaum leserlich, wenig einprägsam und speicherverbrauchend, denn der Graph wird mit Hilfe der XML-Markuptechnologie dargestellt. Inzwischen beliebt gewordene Formate, weil kompakter und leserlicher darstellbar, sind Serialisierungen wie 

- [N3](http://www.w3.org/DesignIssues/Notation3)
- [N-Triples](http://www.w3.org/2001/sw/RDFCore/ntriples/)
- [Turtle](http://www.w3.org/TeamSubmission/turtle/).


