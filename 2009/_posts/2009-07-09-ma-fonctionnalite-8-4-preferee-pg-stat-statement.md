---
layout: post
title: "ma fonctionnalité 8.4 préférée : pg_stat_statement"
author: "mcousin"
categories: [Articles]
redirect_from: "index.php?post/2009/07/09/ma-fonctionnalité-8.4-préférée-%3A-pg_stat_statement"
---




<!--more-->


Dans ce billet,  je vais essayer de faire la publicité de ma fonctionnalité préférée de la version 8.4.<br /><br />Comme toujours, il y a une foule de nouveautés, toutes très intéressantes, et il est difficile d'en déclarer une comme étant la meilleure. Toutefois, j'ai un faible pour pg_stat_statement, et je vais essayer de vous expliquer pourquoi<br /><br />Pour superviser l'activité sur un serveur PostgreSQL, et trouver les requêtes SQL qui dégradent les performances, jusqu'à aujourd'hui, il n'y avait à ma connaissance qu'une seule solution : on active les traces des ordres SQL et de leur durée (log_statement à all, log_duration à on). Ensuite, on récupère la log de postgresql et on la fait avaler à <a href="http://pgfouine.projects.postgresql.org/">pgfouine</a> ou <a href="http://pgfoundry.org/projects/pqa/">pqa</a>, on obtient un rapport et on va voir les développeurs (ou on rajoute un index dans son coin ...) (les paramètres de log sont documentés ici : <a href="http://docs.postgresql.fr/8.4/runtime-config-logging.html">http://docs.postgresql.fr/8.4/runtime-config-logging.html)</a><br /><br />Cette méthode fonctionne, mais a des gros défauts :<br /><ul><li>On trace TOUS les ordres SQL dans la log, ce qui fait qu'on a une log gigantesque assez rapidement (ça peut monter rapidement à plusieurs gigas sur une base très active)</li>

<li>C'est gourmand en ressources, parce qu'il faut formater les traces, les écrire dans la log, découper le message en plusieurs morceaux si il est plus grand qu'une trame syslog et qu'on a décidé de tracer en syslog. Et le surcoût est le même pour une requête de 20 µs et une requête de 2h.</li>

<li>On n'a que la durée des requêtes</li>

</ul>

<br /><br />On peut mitiger l'impact de la fonction de log de plusieurs façons :<br /><ul><li>On ne trace de que les ordres SQL un peu longs</li>

<li>On ne trace que sur de courtes périodes d'activités</li>

</ul>

<br />Dans le premier cas, le problème est qu'on risque de laisser échapper des requêtes unitaires très courtes exécutées des millions de fois. Je l'ai constaté assez souvent dans des développements objets avec de (trop?) nombreux niveaux d'abstraction. On peut facilement se retrouver avec des requêtes insignifiantes exécutées plusieurs milliers de fois à la minute (l'infâme SELECT * FROM DUAL sous Oracle par exemple d'un développeur qui veut vérifier que sa session marche bien avant de lancer un autre ordre, et qui l'a mis à chaque fois qu'il récupère une session d'un pool, au cas où ... ou bien les gens qui réintérrogent un référentiel en permanence). Bref, vaut-il mieux chasser les 10000 appels inutiles à la minute à une requête qui dure 10ms, ou améliorer la requête lancée une fois par minute qui dure 1 seconde? Ça dépend des cas, mais il est&nbsp; préférable que l'audit de performance révèle les 2 (ce qui est très difficile si on ne trace pas tout...)<br /><br />Dans le second cas, c'est garanti, on va rater une période intéressante. Et on ne pourra pas faire d'analyse à posteriori.<br /><br />C'est ici qu'arrive ma fonctionnalité préférée : "et si, au lieu de tout tracer dans une log pour ensuite devoir reparser et réanalyser tout, on avait une zone de mémoire partagée dans laquelle les processus pouvaient mettre à jour des stats cumulées sur chaque requête ?".<br /><br />Les avantages de cette méthode sont :<br /><ul><li>C'est très performant. Je n'ai pas réussi à en mesurer l'impact, et les benchs fait par le développeur montrent un impact négligeable</li>

<li>On a un peu plus d'informations qu'avec la log (le nombre d'enregistrements cumulés ramenés par la requête)</li>

</ul>

C'est donc un mécanisme qu'on peut avoir activé en permanence, consultable par une simple requête.<br /><br />Je n'ai pas de base de production sous la main en ce moment, mais voici un exemple :<br />"Donne moi les 2 requêtes les plus gourmandes en temps d'exécution cumulé depuis le dernier reset des stats" :<br /><pre>test=# SELECT * from pg_stat_statements order by total_time desc limit 2;</pre><pre>-[ RECORD 1 ]------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------</pre><pre>userid&nbsp;&nbsp;&nbsp;&nbsp; | 10</pre><pre>dbid&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | 16384</pre><pre>query&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | INSERT INTO test SELECT * from test;</pre><pre>calls&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | 12</pre><pre>total_time | 0.036099</pre><pre>rows&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | 20475</pre><pre>-[ RECORD 2 ]------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------</pre><pre>userid&nbsp;&nbsp;&nbsp;&nbsp; | 10</pre><pre>dbid&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | 16384</pre><pre>query&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | SELECT pg_catalog.quote_ident(c.relname) FROM pg_catalog.pg_class c WHERE c.relkind IN ('r', 'S', 'v') AND substring(pg_catalog.quote_ident(c.relname),1,6)='pg_sta' AND pg_catalog.pg_table_is_visible(c.oid)</pre><pre>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; : UNION</pre><pre>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; : SELECT pg_catalog.quote_ident(n.nspname) || '.' FROM pg_catalog.pg_namespace n WHERE substring(pg_catalog.quote_ident(n.nspname) || '.',1,6)='pg_sta' AND (SELECT pg_catalog.count(*) FROM pg_catalog.pg_namespace WHERE substring(pg_catalog.quote_ident(nspname) || '.',1,6) = substring('pg_sta',1,pg_catalog.length(pg_catalog.quote_ident(nspname))+1)) &gt; 1</pre><pre>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; : UNION</pre><pre>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; : SELECT pg_catalog.quote_ident(n.nspname) || '.' || pg_catalog.quote_ident(c.relname) FROM pg_catalog.pg_class c, pg_catalog.pg_namespace n WHERE c.relnamespace = n.oid AND c.relkind IN ('r', 'S', 'v') AND substring(pg_catalog.quote_ident(n.nspname) || '.' || pg_catalog.quote_ident(c.relname),1,6)='pg_sta' AND substring(pg_catalog.quote_ident(n.nspname) || '.',1,6) = substring('pg_sta',1,pg_catalog.length(pg_catalog.quote_ident(n.nspname))+1) AND (SELECT pg_catalog.count(*) FROM pg_catalog.pg_namespace WHERE substring(pg_catalog.quote_ident(nspname) || '.',1,6) = substring('pg_sta',1,pg_catalog.length(pg_catalog.quote_ident(nspname))+1)) = 1</pre><pre>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; : LIMIT 1000</pre><pre>calls&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | 4</pre><pre>total_time | 0.006295</pre><pre>rows&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | 87</pre><br />Évidemment, sur une base de test, avec une table test, ce n'est pas très intéressant. Avec une vraie base de production c'est autre chose...<br /><br />On peut aussi utiliser cette vue pour faire des snapshots de l'activité toutes les quelques minutes par exemple :<br /><br />Pour le premier snapshot :<br /><pre><code>CREATE TABLE snapshot as select now(),* from pg_stat_statements;</code></pre>Pour les suivants :<br /><pre><code>INSERT INTO snapshot select now(),* from pg_stat_statements;</code></pre><br /><br />On peut aussi imaginer remettre à zéro les compteurs (suivant ce qu'on veut faire des snapshots, cumulés ou indépendants) avec pg_stat_statements_reset.Avec cela il est possible de 'retrouver' le ou les ordres SQL qui ont fait 'ramer' l'application à une heure donnée (ou dédouaner la base de données...)<br /><br />Les seuls points qui me chagrinent encore sur pg_stat_statement sont qu'il s'agit d'une contrib (c'est normal, mais ça veut dire que cette fonctionnalité sera moins exposée qu'elle le mérite), et que quelques statistiques importantes supplémentaires pourraient servir : la quantité de données lues du cache, la quantité lue du disque, et la quantité écrite dans le cache. Pourquoi pas aussi être capable de séparer le temps de parsing du temps d'exécution (pour repérer les requêtes qui pourraient gagner à être préparées).<br /><br />Bref, une fonctionnalité à tester d'urgence si ce n'est pas déjà fait...<br /><br />La doc officielle : <a href="http://docs.postgresql.fr/8.4/pgstatstatements.html">http://docs.postgresql.fr/8.4/pgstatstatements.html</a><br /><br />Un dernier point : l'autre raison pour laquelle c'est ma fonctionnalité préférée, c'est que c'est une des fonctionnalités qui manquait à PostgreSQL pour faciliter un audit ou un suivi des performances par rapport à Oracle (la vue V$SQLAREA). C'est une fonctionnalité à laquelle on s'habitue assez facilement, c'était assez frustrant de ne pas l'avoir sur son SGBD favori.<br />