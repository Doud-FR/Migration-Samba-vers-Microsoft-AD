<!DOCTYPE html>
<html>

<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Migration de Samba vers Windows</title>
  <link rel="stylesheet" href="https://stackedit.io/style.css" />
</head>

<body class="stackedit">
  <div class="stackedit__html"><h1 id="migrer-un-domaine-samba-vers-un-domaine-microsoft">Migrer un domaine Samba vers un domaine Microsoft</h1>
<h2 id="présentation-de-la-procédure">Présentation de la procédure</h2>
<p>Dans la documentation suivante, on suppose :</p>
<ul>
<li>
<p>Que le dernier serveur Samba-AD que l’on conservera dans le domaine jusqu’à la bascule vers MS-AD s’appelle  <em>oc-addc.epf.oc</em>.</p>
</li>
<li>
<p>Que le serveur MS-AD temporaire nécessaire pour initier le processus de migration s’appelle  <em>ms-ad-temp.epf.oc</em>.</p>
</li>
<li>
<p>Que le premier MS-AD définitif qui sera conservé au terme de la migration s’appelle  <em>ms-ad-final1.epf.oc</em>.</p>
</li>
<li>
<p>Que le deuxième MS-AD définitif qui sera conservé au terme de la migration s’appelle  <em>ms-ad-final2.epf.oc</em>.</p>
</li>
</ul>
<p>Dans les instructions décrites ci-dessous, vous remplacerez  <em>.epf.oc</em>  avec votre propre nom de domaine et les noms des machines par ceux de votre choix ;</p>
<p>La première machine Windows  <em>ms-ad-temp…epf.oc</em>  sera une machine de transition car il existe actuellement un soucis avec l’attribut  <code>ntSecurityDescriptor</code>  quand on joindra  <em>ms-ad-temp.epf.oc</em>  avec  <em>oc-addc.epf.oc</em>. On utilisera donc  <em>ms-ad-temp.epf.oc</em>  comme pivot. Ensuite on joindra  <em>ms-ad-final1.epf.oc</em>  à  <em>ms-ad-temp.epf.oc</em>, ce qui assurera le bon fonctionnement de la réplication et la bonne application des  ACLs  sur la LDAP et le SYSVOL. Ensuite, on supprimera le contrôleur  <em>ms-ad-temp.epf.oc</em>. Enfin, on joindra un deuxième contrôleur de domaine Windows  <em>ms-ad-final2.epf.oc</em>  au domaine Windows 2012R2, ce qui permettra de valider le bon fonctionnement global.</p>
<h3 id="joindre-un-premier-contrôleur-de-domaine-ms-ad-au-domaine-samba-ad">Joindre un premier contrôleur de domaine MS-AD au domaine Samba-AD</h3>
<ul>
<li>
<p><strong>Syspréper</strong>  une première machine Windows 2012R2  <em>ms-ad-temp.epf.oc</em>  en suivant  <a href="https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/sysprep--system-preparation--overview">la documentation officielle de Microsoft</a>  .</p>
</li>
<li>
<p>Intégrer  <em>ms-ad-temp.epf.oc</em>  dans le domaine Samba-AD en suivant la documentation pour  <a href="https://samba.tranquil.it/doc/fr/samba_advanced_methods/samba_add_windows_active_directory.html#samba-add-windows-active-directory">joindre un AD Windows dans un domaine Samba-AD</a>.</p>
</li>
</ul>
<h3 id="démoter-les-contrôleurs-de-domaine-samba-ad">Démoter les contrôleurs de domaine Samba-AD<a href="https://samba.tranquil.it/doc/fr/samba_advanced_methods/samba_migration_to_ms_domain.html#demoting-the-samba-ad-domain-controllers" title="Lien permanent vers cette rubrique"></a></h3>
<p>Une fois que le MS-AD est joint correctement il faut démoter les serveurs Samba-AD. Pour cela il est préférable de supprimer toutes les références au domaine Samba-AD directement sur  <em>ms-ad-temp.epf.oc</em>.</p>
<p>Note</p>
<p>Conceptuellement il est préférable de supprimer les références sur le serveur qui reste actif plutôt que de le faire sur le serveur que l’on veut supprimer.</p>
<ul>
<li>
<p>Supprimer tous les contrôleurs de domaine sauf  <em>samba-ad1.epf.oc</em>. Pour cela, et pour chaque contrôleur du domaine Samba-AD, on execute la ligne de commande suivante sur  <em>samba-ad1.epf.oc</em>  :</p>
<p>samba-tool  domain  demote  --remove-other-dead-server=</p>
</li>
<li>
<p>Eteindre les services Samba sur le dernier Samba-AD  <em>samba-ad1.epf.oc</em>. Toutefois, on maintiendra encore un petit moment  <em>samba-ad1.epf.oc</em>  en fonctionnement pour continuer à bénéficier de la souplesse des commandes  <strong>samba-tool</strong>  pour certaines opérations ultérieures, et aussi pour rendre moins douloureux votre  <a href="http://www.cdeville.fr/article-32408659.html">deuil de Samba</a>  .</p>
<p>systemctl  stop  samba<br>
systemctl  disable  samba</p>
</li>
<li>
<p>Supprimer le dernier contrôleur de domaine Samba-AD en lançant la commande suivante  <em>samba-ad1.epf.oc</em>. On pointe l’exécution de la commande sur le MS-AD  <em>ms-ad-temp.epf.oc</em>  :</p>
<p>samba-tool  domain  demote  --remove-other-dead-server=samba-ad1  -H  ldap://ms-ad-temp.epf.oc  -U  administrator</p>
</li>
<li>
<p>Vérifier que les rôles FSMO ont été transférés lors du dernier  <em>demote</em>. Il restera les rôles  <em>DomainDnsZones</em>  et  <em>ForestDNSZones</em>  qui n’auront pas été transférés, on force le transfert :</p>
<p>samba-tool  fsmo  show  -H  ldap://ms-ad-temp.epf.oc  -U  administrator<br>
samba-tool  fsmo  seize  --role=all  -H  ldap://ms-ad-temp.epf.oc  -U  administrator</p>
</li>
<li>
<p>Nettoyer les entrées DNS. Dans une console DNS ouverte sur  <em>ms-ad-temp.epf.oc</em>  vérifier que les entrées DNS pour  <em>ms-ad-temp.epf.oc</em>  sont bien toutes présentes (champs A, NS, SRV, CNAME) et supprimer les références DNS à  <em>samba-ad1.epf.oc</em>. On corrigera aussi les  <em>GLUE records</em>  (champ de type NS) pour le champ  _<em>msdcs</em>  dans la zone  <em>.epf.oc</em>  (pas dans la zone  _<em>msdcs.epf.oc</em>).</p>
</li>
<li>
<p>Créer la zone inverse si elle n’existe pas encore puis créer le champ PTR pour  <em>ms-ad-temp.epf.oc</em>  ;</p>
</li>
</ul>
<p><strong>Désormais nous avons un domaine full Microsoft avec un seul contrôleur de domaine.</strong></p>
<ul>
<li>
<p>Mettre à jour la forêt en niveau 2012R2 avec un  <strong>Powershell</strong>  :</p>
<p>Set-ADDomainMode -identity .epf.oc -DomainMode Windows2012R2Domain<br>
Set-ADForestMode -identity .epf.oc -ForestMode Windows2012R2Forest</p>
</li>
</ul>
<h3 id="joindre-le-premier-contrôleur-de-domaine-windows-définitif">Joindre le premier contrôleur de domaine Windows définitif<a href="https://samba.tranquil.it/doc/fr/samba_advanced_methods/samba_migration_to_ms_domain.html#joining-the-first-definitive-windows-domain-controller" title="Lien permanent vers cette rubrique"></a></h3>
<p>Pour finir la migration il est nécessaire de mettre un deuxième MS-AD en place et de réinitialiser la partie DFS-R pour la réplication du  <code>SYSVOL</code>  :</p>
<ul>
<li>
<p><strong>Syspréper</strong>  une deuxième machine Windows 2012R2  <em>ms-ad-final1.epf.oc</em>  en suivant  <a href="https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/sysprep--system-preparation--overview">la documentation officielle de Microsoft</a>.</p>
</li>
<li>
<p>Joindre  <em>ms-ad-final1.epf.oc</em>  au contrôleur de domaine  <em>ms-ad-temp.epf.oc</em>.</p>
</li>
<li>
<p>Avec une console DNS ouverte sur  <em>ms-ad-final1.epf.oc</em>, vérifier que les champs DNS sont bien tous présents.</p>
</li>
<li>
<p>Sur  <em>ms-ad-final1.epf.oc</em>, vérifier la réplication :</p>
<p>repadmin /kcc<br>
repadmin /showrepl</p>
</li>
<li>
<p>Démoter  <em>ms-ad-temp.epf.oc</em>  en exécutant la commande suivante sur  <em>samba-ad1.epf.oc</em>  (avec bien sûr les services Samba toujours arrêtés et désactivés) ;</p>
<p>samba-tool  domain  demote  --remove-other-dead-server=ms-ad-temp  -H  ldap://ms-ad-final1.epf.oc  -U  administrator</p>
</li>
<li>
<p>Nettoyer les DNS ;</p>
</li>
<li>
<p>Regénérer le DFS-R ;</p>
<p>dfsrmig  /createglobalobjects<br>
net  stop  dfsr<br>
net  start  dfsr</p>
</li>
<li>
<p>Vérifier que le  <strong>dcdiag</strong>  est propre (Attention : le  <strong>dcdiag</strong>  peut afficher des erreurs  <em>eventlog</em>  qui peuvent être obsolètes et qui ne concernent pas la migration) ;</p>
<p>dcdiag</p>
</li>
</ul>
<h3 id="joindre-le-deuxième-contrôleur-de-domaine-windows-définitif">Joindre le deuxième contrôleur de domaine Windows définitif<a href="https://samba.tranquil.it/doc/fr/samba_advanced_methods/samba_migration_to_ms_domain.html#joining-the-second-final-windows-domain-controller" title="Lien permanent vers cette rubrique"></a></h3>
<p>Cette étape permet de valider le bon fonctionnement du domaine en environement MS-AD.</p>
<ul>
<li>
<p><strong>Syspréper</strong>  une troisième machine Windows 2012R2  <em>ms-ad-final2.epf.oc</em>  en suivant  <a href="https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/sysprep--system-preparation--overview">la documentation officielle de Microsoft</a>.</p>
</li>
<li>
<p>Intégrer  <em>ms-ad-final2.epf.oc</em>  dans le domaine Windows en suivant la documentation pour  <a href="https://samba.tranquil.it/doc/fr/samba_advanced_methods/samba_add_windows_active_directory.html#samba-add-windows-active-directory">joindre un AD Windows dans un domaine</a>  en s’arrêtant après la jonction. Après redémarrage le répertoire  <code>SYSVOL</code>  doit être correctement répliqué et les partages  <code>SYSVOL</code>  et  <code>NetLogon</code>  créés sans avoir à modifier la clef  <code>SysvolReady</code>.</p>
</li>
<li>
<p>Nettoyer les DNS (<strong>attention au champ CNAME _msdcs</strong>).</p>
</li>
<li>
<p>Vérifier que la réplication est bonne en créant un fichier dans le dossier  <code>SYSVOL</code>  et en vérifiant qu’il se réplique bien.</p>
</li>
</ul>
<h3 id="eteindre-définitivement-votre-samba">Eteindre définitivement votre Samba<a href="https://samba.tranquil.it/doc/fr/samba_advanced_methods/samba_migration_to_ms_domain.html#turning-off-your-samba-permanently" title="Lien permanent vers cette rubrique"></a></h3>
<ul>
<li>
<p>Sur votre  <em>samba-ad1.epf.oc</em>, lancer la commande :</p>
<p>shutdown  -h  now</p>
</li>
<li>
<p>Optionnellement : mettre à jour votre CV.</p>
</li>
</ul>
<p>Note</p>
<p>Désormais, vous avez un domaine Microsoft qui fonctionne de la même manière que votre domaine Samba-AD. Si votre domaine Samba-AD ne fonctionnait pas bien, alors votre domaine MS-AD ne fonctionnera pas mieux.</p>
</div>
</body>

</html>
