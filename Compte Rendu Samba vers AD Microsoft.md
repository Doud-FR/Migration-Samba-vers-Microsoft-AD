<!DOCTYPE html>
<html>

<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Compte Rendu Samba vers AD Microsoft</title>
  <link rel="stylesheet" href="https://stackedit.io/style.css" />
</head>

<body class="stackedit">
  <div class="stackedit__html"><h1 id="compte-rendu-migration-samba-vers-ad-microsoft">Compte Rendu Migration Samba vers AD Microsoft</h1>
<h2 id="résumé-des-process-testés">Résumé des Process Testés</h2>
<ol>
<li>Procedure officielle Microsoft</li>
<li>Procédure Microsoft modifiée et adaptée</li>
<li>Procédure simplifiée</li>
<li>Procédure de migration sur un DC en 2022</li>
</ol>
<h3 id="résultat-des-procédures-testées">Résultat des procédures testées</h3>
<p>1.<strong>Procédure officielle Microsoft</strong>: A la dépromote de Samba, l’AD n’était plus fonctionnel, Domaine Introuvable, DNS sur l’AD non fonctionnel<br>
5. <strong>Procédure Microsoft modifié et adapté</strong>: Cette procédure a donnée un résultat entièrement fonctionnel coté AD avec un serveur DC sous 2008R2, GPO fonctionnelles également, seul quelques traces subsistes dans le DNS et reviennent systématiquement. Donc cette procédure est applicable mais trop complexe car elle nécessite un AD temporaire, un autre AD de Pivot et enfin un AD final.<br>
6. <strong>Procédure simplifié</strong>: Nous avons donc travaillé pour simplifier cette procédure et la fiabilisé. Il suffira de deux AD, un temporaire en 2008R2 puis un en 2012R2 qui sera le DC final.<br>
7. <strong>Procédure de migration sur un DC en 2022</strong>: Nous sommes allés un peu plus loin dans les tests en montant un DC sous 2022 puis en le passant DC principal seul après dépromote du DC en 2012R2. Ce test est concluant, avec un DC en 2022 le domaine est fonctionnel, les GPO également, mais en plus le DC ne remonte que des soucis liés à SYSVOL.</p>
<h3 id="problèmes-identifiés">Problèmes identifiés</h3>
<p>Cette migration de Samba vers un AD Microsoft, bien que fonctionnelle, laisse quelques traces qu’il n’est pas possible à ce jour de corriger, soit par ce qu’elles émanes directement de ce gros changement, soit par la mécanique imposé par Microsoft, comme par exemple le souci avec la réplication de SYSVOL.</p>
<ol>
<li>Il reste des traces d’anciens serveur dans le DNS, elles reviennent malgré les suppressions mais n’impactes en rien le bon fonctionnement. Le DNS ne peut juste pas être considéré comme propre.</li>
<li>La réplication de SYSVOL, dossier important dans la mécanique de fonctionnement des GPO, ne peut se répliquer automatiquement. Il faudra le copier à la main en cas de migration vers un autre DC et modifier une clé de registre pour valider cette nouvelle copie.</li>
</ol>
<h2 id="environment-et-infrastructure-qui-a-permis-de-réaliser-ce-projet">Environment et Infrastructure qui a permis de réaliser ce projet</h2>
<p>Les machines de départ Samba et Windows 2008R2 sont issues de sauvegarde Veeam en Complète avec un Job créé uniquement pour extraire des copies parfaites de deux machines virtuelles principales.<br>
L’infrastructure de test est composé de:<br>
<strong>Serveur Dell R730</strong>:<br>
2x CPU Intel® Xeon® CPU E5-2643 v4 @ 3.40GHz 12 Coeurs/24 Threads<br>
256 Go de RAM en DDR4 2100Mhz<br>
8.2To d’espace disque<br>
<strong>Routeur</strong>:<br>
Stormshield SN310 pour conserver l’adressage IP de l’infrastructure du client<br>
<strong>Switch</strong>:<br>
HP 2920 48 Ports en 1Gbps</p>
<h3 id="machines-virtuelles">Machines Virtuelles</h3>
<p><strong>Hyperviseur ESXI 8.0 Update 3</strong> Identique à celui chez le client<br>
1 VM Linux Debian 12: Samba <strong>issue de la sauvegarde Veeam chez le client</strong><br>
1 VM Windows 2008R2: DC secondaire <strong>issue de la sauvegarde Veeam chez le client</strong><br>
1 VM Windows Serveur 2022: Veeam afin de restaurer les VM<br>
1 VM Virtualisation NAS Synology: Repository pour les VM exportées du client<br>
1 VM Windows 10: Afin de tester les GPO<br>
1 VM Windows 11: Afin de tester les GPO<br>
1 VM Windows Serveur 2012R2: Qui deviendra le DC unique et principal du domaine<br>
1 VM Windows Serveur 2022: Qui a servi à valider la viabilité du process mis en place pour se rassurer sur le devenir de l’infrastrucute avec une telle migration</p>
<h2 id="conclusions">Conclusions</h2>
<p>Après plusieurs jours de travail consécutifs pour élaborer un process viable dans le temps et fiable sur la mise en place ainsi que sur le résultat final, nous sommes désormais en mesure de valider cette possibilité de migration. Celle-ci s’avère beaucoup plus simple et moins impactante à mettre en œuvre que de repartir de zéro.</p>
<p>Malgré le fait que le DNS ne puisse pas être nettoyé complètement, le résultat du DcDiag, qui donne un état de santé précis et complet de la structure AD, est plus que satisfaisant : seules les mentions relatives au problème de réplication du SYSVOL sont en échec, tout le reste est au vert.</p>
<p>Il faudra cependant garder en mémoire que, si ce process est retenu, lors de l’ajout d’un futur contrôleur de domaine il sera nécessaire de copier manuellement le contenu de SYSVOL sur le nouveau DC, puis de créer la clé de registre correspondante pour passer le statut SYSVOL à “Ready”.</p>
</div>
</body>

</html>
