---
author: Umberto D'Ovidio
date: "2016-09-16T00:00:00Z"
description: Jekyll utilizza di default le date inglesi. Scopriamo come cambiarle
  in modo da mostrare date italiane.
image: /images/calendario.jpg
title: Italianizzare le date in Jekyll
---

Una modifiche obbligate che noi italiani dobbiamo fare per il nostro blog jekyll è quella di cambiare il formato delle date dei post. Infatti Jekyll utilizza di default i mesi in lingua inglese.
<!--more-->
Ad esempio, la pagina che mostra gli ultimi post utilizza abbreviazioni dei mesi inglesi, tipo **Jan** o **Sep**, numero del giorno e anno. Noi italiani siamo invece abituati a numero del giorno, mese ed anno.
![immagine esempio del sito](/images/date-inglesi-brevi.png)
Diamo un'occhiata al codice che genera le date visualizzate nell'immagine sopra.

Se volessimo una data italiana, vorremmo inanzitutto il giorno, selezionabile con l'opzione **%-d**, poi il nome del mese, che selezioneremo con uno **switch statement**, ed infine l'anno, **%Y**.
Ecco il codice completo.
{{< gist dovidio 15f4e4c4fff1a7abca0d3234ef35f794 >}}
In breve, utilizziamo la variabile **m** per ottenere il numero del mese, e a seconda di quest'ultimo selezioniamo l'abbreviazione italiana del mese.
Ecco come cambiano le date
![immagine esempio del sito](/images/date-italiane-brevi.png)
La stessa tecnica si può utilizzare se si vogliono i nomi dei mesi interi, o i nomi dei giorni.
