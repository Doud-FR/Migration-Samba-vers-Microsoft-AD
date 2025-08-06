<!DOCTYPE html>
<html>

<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Process</title>
  <link rel="stylesheet" href="https://stackedit.io/style.css" />
</head>

<body class="stackedit">
  <div class="stackedit__html"><h2 id="pour-partir-“propre”-avec-2022-et-pouvoir-ajouter-plus-tard-un-dc-secondaire-sain">Pour partir “propre” avec 2022 (et pouvoir ajouter plus tard un DC secondaire sain)</h2>
<p>Voici <strong>le plan qui marche dans les environnements tordus</strong> :</p>
<h3 id="dépromote-le-2012-proprement">1. <strong>Dépromote le 2012 proprement</strong></h3>
<ul>
<li>
<p><strong>Assure-toi qu’il n’a plus aucun rôle FSMO</strong> (<code>netdom query fsmo</code> sur 2022).</p>
</li>
<li>
<p>Sur 2012, fais :</p>
<p><code>dcpromo</code></p>
<p>ou via “supprimer contrôleur de domaine” dans “Outils &gt; Utilisateurs et ordinateurs AD”.</p>
</li>
<li>
<p><strong>Supprime manuellement</strong> tout ce qui reste dans l’AD (Sites &amp; Services, ADSI, DNS) concernant ce DC après dépromotion.</p>
</li>
</ul>
<hr>
<h3 id="rends-le-2022-“autoritaire”-sur-sysvol">2. <strong>Rends le 2022 “autoritaire” sur SYSVOL</strong></h3>
<p>Puisque tu as copié SYSVOL à la main, c’est ton <strong>unique source officielle</strong>.</p>
<ul>
<li>
<p><strong>Positionne-le comme source autoritaire pour DFSR</strong> :<br>
Lancer PowerShell en admin sur le DC 2022<br>
Arrêter le service DFSR<br>
Stop-Service DFSR<br>
Dans le registre, ajoute la clé:<br>
New-ItemProperty -Path “HKLM:\SYSTEM\CurrentControlSet\Services\DFSR\Parameters” -Name “StopReplicationOnAutoRecovery” -PropertyType DWORD -Value 1</p>
<h1 id="elle-existe-peut-être-déjà-si-t’as-galéré">(Elle existe peut-être déjà si t’as galéré)</h1>
<h1 id="démarre-dfsr">Démarre DFSR</h1>
<p>Start-Service DFSR`</p>
<ul>
<li>
<p>Puis, place la clé <code>SysvolReady</code> à 1 si besoin et redémarre Netlogon :</p>
<p>powershell</p>
<p>CopierModifier</p>
<p><code>Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Netlogon\Parameters" -Name "SysvolReady" -Value 1 Restart-Service Netlogon</code></p>
</li>
</ul>
</li>
</ul>
<hr>
<h3 id="purge-tous-les-restes-d’anciens-dc">3. <strong>Purge tous les restes d’anciens DC</strong></h3>
<ul>
<li>
<p><strong>Vérifie dans ADSI Edit</strong> :</p>
<ul>
<li>
<p>CN=Domain System Volume</p>
</li>
<li>
<p>CN=Topology</p>
</li>
<li>
<p>Supprime tout ce qui n’est plus vivant.</p>
</li>
</ul>
</li>
<li>
<p><strong>Vérifie la configuration DFSR</strong> :<br>
Parfois, il reste des GUID orphelins ou des références à d’anciens DC dans la config du groupe de réplication.</p>
</li>
</ul>
<hr>
<h3 id="ajoute-le-nouveau-dc-secondaire">4. <strong>Ajoute le nouveau DC secondaire</strong></h3>
<ul>
<li>
<p>Promote-le normalement.</p>
</li>
<li>
<p>Il doit créer un SYSVOL vide à l’étape d’intégration, et démarrer une initial sync DFSR avec le 2022.</p>
</li>
<li>
<p><strong>Vérifie dans l’observateur d’événements DFSR</strong> qu’il se met bien en “normal sync” ou “completed”.</p>
</li>
</ul>
<hr>
<h3 id="si-la-réplication-ne-part-pas">5. <strong>Si la réplication ne part pas</strong></h3>
<ul>
<li>
<p><strong>Forcer la réplication initiale</strong> sur le nouveau DC :</p>
<p>powershell</p>
<p>CopierModifier</p>
<p><code># Sur le DC secondaire tout neuf, place le en mode non-auth initial sync dfsrdiag pollad</code></p>
</li>
<li>
<p><strong>Si toujours bloqué</strong>, tu peux le forcer à démarrer comme un “non-authoritative” :</p>
<ol>
<li>
<p>Arrête DFSR</p>
</li>
<li>
<p>Supprime le contenu de <code>C:\System Volume Information\DFSR</code> (sur le DC SECONDARY SEULEMENT)</p>
</li>
<li>
<p>Place la clé “SysvolReady” à 0, puis à 1 après restart, puis start DFSR</p>
</li>
<li>
<p>DFSR doit alors resynchroniser (vérifie les logs !)</p>
</li>
</ol>
</li>
</ul>
<hr>
<h3 id="vérifie-le-partage-sysvol">6. <strong>Vérifie le partage SYSVOL</strong></h3>
<ul>
<li>
<p>Sur les deux DC :</p>
<p>nginx</p>
<p>CopierModifier</p>
<p><code>net share</code></p>
<p>et</p>
<p>sql</p>
<p>CopierModifier</p>
<p><code>\\DC2022\sysvol \\DC2022\netlogon \\DC-NEW\sysvol</code></p>
</li>
</ul>
<hr>
<h3 id="bonus--script-d’audit">7. <strong>Bonus : Script d’audit</strong></h3>
<p>Si tu veux, je peux te faire un <strong>PowerShell</strong> pour auditer toutes les entrées DFSR/SYSVOL et t’afficher l’état de santé.</p>
<hr>
<h2 id="en-résumé"><strong>En résumé</strong></h2>
<ul>
<li>
<p><strong>Dépromote 2012</strong> proprement (attention aux objets orphelins)</p>
</li>
<li>
<p><strong>Vérifie que 2022 est autoritaire</strong></p>
</li>
<li>
<p><strong>Ajoute un DC secondaire tout neuf</strong></p>
</li>
<li>
<p><strong>Vérifie que la réplication démarre</strong></p>
</li>
<li>
<p><strong>Si coincé, purge DFSR sur le secondaire puis restart</strong></p>
</li>
</ul>
<hr>
<h3 id="n’hésite-pas-à-me-redonner-l’état-exact-je-peux-te-faire-la-check-list-détaillée-“à-la-main”-pour-chaque-étape-y-compris-tout-ce-qu’il-faut-vérifier-dans-adsi-et-la-config-dfsr.">N’hésite pas à me redonner l’état exact, je peux te faire la check-list détaillée “à la main” pour chaque étape (y compris tout ce qu’il faut vérifier dans ADSI et la config DFSR).</h3>
<p><strong>Je connais bien ces galères de migrations multi-sauts, t’es pas tout seul !</strong><br>
Tu veux la check-list complète ?</p>
<h5 id="vous-avez-dit-">Vous avez dit :</h5>
<p>2012 à tous les rôles FSMO, faut les migrer avant de le dépromote? donne moi une procédure pour dépromote un AD principal lors d’une migration vers un AD secondaire par exemple dans le cadre d’une migration d’un AD 2012r2 vers un AD 2022</p>
<h6 id="chatgpt-a-dit-">ChatGPT a dit :</h6>
<p>Excellente question, <strong>oui, il faut absolument transférer tous les rôles FSMO avant de dépromouvoir l’AD principal</strong> (le “Primary Domain Controller”, même si le terme technique actuel est “PDC Emulator” ou “FSMO holder”).</p>
<h2 id="🚩-procédure-pour-migrer-tous-les-rôles-fsmo-et-dépromouvoir-un-dc-principal-migration-2012r2-→-2022">🚩 <strong>Procédure pour migrer tous les rôles FSMO et dépromouvoir un DC principal (migration 2012R2 → 2022)</strong></h2>
<h3 id="vérifier-les-rôles-fsmo-actuels"><strong>1. Vérifier les rôles FSMO actuels</strong></h3>
<p>Sur <strong>n’importe quel DC</strong> (mais fais-le de préférence sur le 2012r2 qui les détient) :</p>
<p>powershell</p>
<p>CopierModifier</p>
<p><code>netdom query fsmo</code></p>
<p>Tu dois voir <strong>tous les rôles</strong> détenus par le 2012R2.</p>
<hr>
<h3 id="préparer-ton-dc-2022"><strong>2. Préparer ton DC 2022</strong></h3>
<ul>
<li>
<p><strong>Le DC 2022 doit déjà être intégré au domaine et opérationnel</strong> (ping, DNS, AD répliqués, <code>dcdiag</code> OK, etc.).</p>
</li>
<li>
<p>Vérifie que la réplication est OK :</p>
<p>powershell</p>
<p>CopierModifier</p>
<p><code>repadmin /replsummary dcdiag /v</code></p>
</li>
</ul>
<hr>
<h3 id="transférer-les-rôles-fsmo"><strong>3. Transférer les rôles FSMO</strong></h3>
<p>Sur le <strong>DC 2022</strong>, exécute :</p>
<p>powershell</p>
<p>CopierModifier</p>
<p>`# Importer le module si besoin<br>
Import-Module ActiveDirectory</p>
<h1 id="transférer-tous-les-rôles-fsmo">Transférer tous les rôles FSMO</h1>
<p>Move-ADDirectoryServerOperationMasterRole -Identity “NOM_DU_DC_2022” -OperationMasterRole 0,1,2,3,4 -Force`</p>
<p><strong>Remplace</strong> <code>NOM_DU_DC_2022</code> par le nom NetBIOS de ton serveur 2022.</p>
<p><strong>Rôles FSMO</strong> :</p>
<ul>
<li>
<p>0 : Schema Master</p>
</li>
<li>
<p>1 : Domain Naming Master</p>
</li>
<li>
<p>2 : RID Master</p>
</li>
<li>
<p>3 : PDC Emulator</p>
</li>
<li>
<p>4 : Infrastructure Master</p>
</li>
</ul>
<blockquote>
<p>💡 <strong>Conseil</strong> : Tu peux aussi faire les transferts via les consoles GUI (utilisateurs et ordinateurs AD, domaines et approbations, schéma AD, etc.), mais la commande PowerShell ci-dessus fait tout d’un coup.</p>
</blockquote>
<hr>
<h3 id="vérifier-la-prise-en-compte"><strong>4. Vérifier la prise en compte</strong></h3>
<p>Vérifie que le DC 2022 détient bien <strong>tous les rôles</strong> :</p>
<p>powershell</p>
<p>CopierModifier</p>
<p><code>netdom query fsmo</code></p>
<p>Tous les rôles doivent pointer sur le 2022.</p>
<hr>
<h3 id="dépromouvoir-le-dc-2012r2"><strong>5. Dépromouvoir le DC 2012R2</strong></h3>
<p>Sur le DC 2012R2 :</p>
<ul>
<li>
<p><strong>GUI</strong> : “Outils” → “Services de domaine Active Directory” → “Supprimer les services de domaine…”</p>
</li>
<li>
<p><strong>PowerShell</strong> :</p>
<p>powershell</p>
<p>CopierModifier</p>
<p><code>Uninstall-ADDSDomainController -DemoteOperationMasterRole:$true -RemoveApplicationPartition:$true -Force</code></p>
<p>(tu devras saisir le mot de passe du compte Administrateur local qui sera utilisé après dépromotion)</p>
</li>
</ul>
<p><strong>Laisse la case “Ce contrôleur de domaine est le dernier contrôleur du domaine” décochée</strong> (puisque tu as ton DC 2022 actif).</p>
<hr>
<h3 id="vérifier-le-nettoyage"><strong>6. Vérifier le nettoyage</strong></h3>
<ul>
<li>
<p><strong>DNS</strong> : supprime toute entrée résiduelle du 2012R2 si elles restent.</p>
</li>
<li>
<p><strong>Sites et Services AD</strong> : vérifie que le DC 2012R2 a disparu.</p>
</li>
<li>
<p><strong>ADSI Edit</strong> : purger les objets orphelins si besoin.</p>
</li>
<li>
<p><strong>Ordinateurs du domaine</strong> : vérifie qu’il n’y a pas de compte d’ordinateur “fantôme” du DC 2012R2.</p>
</li>
</ul>
<hr>
<h3 id="contrôles-post-dépromotion"><strong>7. Contrôles post-dépromotion</strong></h3>
<p>Sur le 2022, vérifie que :</p>
<ul>
<li>
<p><strong>SYSVOL/NETLOGON</strong> sont partagés et OK (<code>net share</code>)</p>
</li>
<li>
<p><strong>La réplication DFSR tourne</strong> (évènements, <code>dfsrdiag</code>, etc.)</p>
</li>
<li>
<p><strong>La santé AD est OK</strong> :</p>
<p>powershell</p>
<p>CopierModifier</p>
<p><code>dcdiag /v repadmin /replsummary</code></p>
</li>
</ul>
<hr>
<h2 id="checklist-rapide-résumé"><strong>Checklist rapide (résumé)</strong></h2>
<ol>
<li>
<p>Vérifier les rôles FSMO sur 2012R2</p>
</li>
<li>
<p>Vérifier la santé de 2022</p>
</li>
<li>
<p>Transférer les rôles FSMO vers 2022</p>
</li>
<li>
<p>Vérifier le transfert (<code>netdom query fsmo</code>)</p>
</li>
<li>
<p>Dépromouvoir 2012R2 (via GUI ou PowerShell)</p>
</li>
<li>
<p>Nettoyer les objets/dns/sites</p>
</li>
<li>
<p>Vérifier la santé du domaine sur 2022</p>
</li>
</ol>
<hr>
<h2 id="conseils"><strong>Conseils</strong></h2>
<ul>
<li>
<p><strong>Fais toujours un snapshot/backup avant dépromotion.</strong></p>
</li>
<li>
<p><strong>Attends la fin de réplication AD avant toute opération majeure.</strong></p>
</li>
<li>
<p><strong>Surveille bien les journaux d’évènements sur les deux serveurs pendant et après la transition.</strong></p>
</li>
</ul>
</div>
</body>

</html>
