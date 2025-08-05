---


---

<h1 id="process-migration-samba-vers-ad">Process Migration Samba vers AD</h1>
<p>Sources:<br>
Migration: <a href="https://samba.tranquil.it/doc/fr/samba_advanced_methods/samba_migration_to_ms_domain.html">Trankil.it=&gt;Migrer un domaine Samba vers un domaine Microsoft</a><br>
Jonction: <a href="https://samba.tranquil.it/doc/fr/samba_advanced_methods/samba_add_windows_active_directory.html#samba-add-windows-active-directory">Microsoft=&gt;# Ajouter un AD Windows dans son domaine Samba</a></p>
<h2 id="samba">Samba:</h2>
<h3 id="prérequis">Prérequis:</h3>
<pre class=" language-cmd"><code class="prism  language-cmd">apt install patch python3-markdown
</code></pre>
<p>Vérifier la base de donnée annuaire:</p>
<pre class=" language-cmd"><code class="prism  language-cmd">samba-tool dbcheck --cross-ncs --fix --yes
</code></pre>
<blockquote>
<p>Remarques:<br>
Il est possible que des erreurs remontent lors de la première commande, il suffit de la relancer une seconde fois.<br>
Exécuter le script python pour créer les attributs msDS-SDReferenceDomain manquant dans Samba (Procédure Jonction)</p>
</blockquote>
<p>Il manque un attribut dans Samba qui va générer des messages d’erreur dans la commande <strong>dcdiag</strong>.<br>
Pour résoudre le problème, recréer deux attributs <code>msDS-SDReferenceDomain</code> dans la partition <code>cn=configuration</code> qui pointent vers le <code>rootDN</code> de l’Active Directory.<br>
Pour ce faire, vous pouvez exécuter le script suivant sur le serveur Samba-AD :</p>
<p>Créer le fichier python:</p>
<pre class=" language-cmd"><code class="prism  language-cmd">nano fix_msDS-SDReferenceDomain.py
</code></pre>
<p>Coller le code suivant:</p>
<pre class=" language-python"><code class="prism  language-python"><span class="token comment"># -*- coding: utf-8 -*-</span>
<span class="token keyword">from</span> samba<span class="token punctuation">.</span>auth <span class="token keyword">import</span> system_session
<span class="token keyword">from</span> samba<span class="token punctuation">.</span>credentials <span class="token keyword">import</span> Credentials
<span class="token keyword">from</span> samba<span class="token punctuation">.</span>samdb <span class="token keyword">import</span> SamDB
<span class="token keyword">import</span> optparse
<span class="token keyword">import</span> samba<span class="token punctuation">.</span>getopt <span class="token keyword">as</span> options

parser <span class="token operator">=</span> optparse<span class="token punctuation">.</span>OptionParser<span class="token punctuation">(</span><span class="token string">"/etc/samba/smb.conf"</span><span class="token punctuation">)</span>
sambaopts <span class="token operator">=</span> options<span class="token punctuation">.</span>SambaOptions<span class="token punctuation">(</span>parser<span class="token punctuation">)</span>

lp <span class="token operator">=</span> sambaopts<span class="token punctuation">.</span>get_loadparm<span class="token punctuation">(</span><span class="token punctuation">)</span>
domaine <span class="token operator">=</span> sambaopts<span class="token punctuation">.</span>_lp<span class="token punctuation">.</span>get<span class="token punctuation">(</span><span class="token string">'realm'</span><span class="token punctuation">)</span><span class="token punctuation">.</span>lower<span class="token punctuation">(</span><span class="token punctuation">)</span>

creds <span class="token operator">=</span> Credentials<span class="token punctuation">(</span><span class="token punctuation">)</span>
creds<span class="token punctuation">.</span>guess<span class="token punctuation">(</span>lp<span class="token punctuation">)</span>

samdbloc <span class="token operator">=</span> SamDB<span class="token punctuation">(</span>session_info<span class="token operator">=</span>system_session<span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">,</span>credentials<span class="token operator">=</span>creds<span class="token punctuation">,</span> lp<span class="token operator">=</span>lp<span class="token punctuation">)</span>
listdn <span class="token operator">=</span> <span class="token builtin">list</span><span class="token punctuation">(</span>samdbloc<span class="token punctuation">.</span>search<span class="token punctuation">(</span>base<span class="token operator">=</span><span class="token string">'cn=partitions,'</span> <span class="token operator">+</span> <span class="token builtin">str</span><span class="token punctuation">(</span>samdbloc<span class="token punctuation">.</span>get_config_basedn<span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">,</span> expression<span class="token operator">=</span><span class="token punctuation">(</span><span class="token string">'(|(dnsroot=ForestDnsZones.%s)(dnsroot=DomainDnsZones.%s))'</span> <span class="token operator">%</span> <span class="token punctuation">(</span>domaine<span class="token punctuation">,</span>domaine<span class="token punctuation">)</span> <span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">)</span>

<span class="token keyword">for</span> dn <span class="token keyword">in</span> listdn<span class="token punctuation">:</span>
    <span class="token keyword">if</span> <span class="token operator">not</span> <span class="token string">'msDS-SDReferenceDomain'</span> <span class="token keyword">in</span> dn <span class="token punctuation">:</span>
        ldif_data <span class="token operator">=</span> u<span class="token triple-quoted-string string">"""dn: %s
changetype: modify
replace: msDS-SDReferenceDomain
msDS-SDReferenceDomain: %s"""</span> <span class="token operator">%</span> <span class="token punctuation">(</span>dn<span class="token punctuation">[</span><span class="token string">'dn'</span><span class="token punctuation">]</span><span class="token punctuation">,</span><span class="token builtin">str</span><span class="token punctuation">(</span>samdbloc<span class="token punctuation">.</span>get_root_basedn<span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">)</span>
        <span class="token keyword">print</span><span class="token punctuation">(</span>ldif_data<span class="token punctuation">)</span>
        samdbloc<span class="token punctuation">.</span>modify_ldif<span class="token punctuation">(</span>ldif_data<span class="token punctuation">)</span>
</code></pre>
<p>Exécuter le fichier python:</p>
<pre class=" language-cmd"><code class="prism  language-cmd">./fix_msDS-SDReferenceDomain.py
</code></pre>
<h2 id="windows">Windows:</h2>
<ul>
<li><strong>Prérequis:</strong>
<ul>
<li>IP statique</li>
<li>DNS Principal celui du Samba</li>
<li>Renommer avant jonction au domaine ou avant promote</li>
</ul>
</li>
</ul>
<p>Ici sur 2008R2:</p>
<pre class=" language-powershell"><code class="prism  language-powershell">dcpromo
</code></pre>
<ul>
<li>
<p><strong>Paramétrage avancé:</strong></p>
<ul>
<li>DNS</li>
<li>Catalogue Global</li>
<li>Réplication depuis Samba</li>
</ul>
</li>
<li>
<p><strong>Reboot du serveur puis ouverture d’une session sur le domaine qui soit:</strong></p>
<ul>
<li>Admin du domaine</li>
<li>Admin du Schema</li>
<li>Admin DNS</li>
</ul>
</li>
</ul>
<p>Forcer l’activation du répertoire Sysvol sur le MS-AD :<br>
Dans PowerShell en Admin taper:</p>
<pre class=" language-cmd"><code class="prism  language-cmd">Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Netlogon\Parameters" -Name "SysvolReady" -Value "1"
</code></pre>
<ul>
<li>Recopier le contenu du  <code>sysvol</code>  depuis le serveur Samba. Pour ce faire, dans un explorateur de fichier, tapez  <code>\\srvads\sysvol\</code>, puis aller dans le dossier correspondant au nom de votre domaine (par exemple  <em>ad.mydomain.lan</em>) et copier  <code>Policies</code>  et  <code>Scripts</code>  dans  <code>C:\windows\SYSVOL\domain</code>  (mais pas le nom de domaine). Après la copie on aura donc ces deux répertoires :
<ul>
<li><code>C:\windows\SYSVOL\domain\Policies</code>  ;</li>
<li><code>C:\windows\SYSVOL\domain\Scripts</code>  ;</li>
</ul>
</li>
</ul>
<p><strong>Note</strong><br>
Il y a un lien de  <code>C:\windows\SYSVOL\sysvol\ad.mydomain.lan</code>  vers  <code>C:\windows\SYSVOL\domain</code>.</p>
<ul>
<li>Redémarrer le serveur MS-AD :</li>
</ul>
<pre class=" language-cmd"><code class="prism  language-cmd">shutdown -r -t 0
</code></pre>
<ul>
<li>
<p>Inverser les serveurs DNS de la carte réseau. Le premier serveur DNS doit être lui même (<code>127.0.0.1</code>), et en deuxième on indique le serveur Samba-AD (Microsoft fait l’inverse lors de la jonction).</p>
</li>
<li>
<p>Dans la console DNS, changer le redirecteur DNS vers le récurseur du réseau (par défaut Windows met le premier contrôleur de domaine comme récurseur lors de la jonction).<br>
<strong>Mettre l’AD, puis Samba, puis 1.1.1.1 et 8.8.8.8</strong></p>
</li>
<li>
<p>Modifier ensuite la configuration NTP dans la base de registre du MS-AD :</p>
</li>
</ul>
<pre class=" language-cmd"><code class="prism  language-cmd">Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\W32Time\Parameters" -Name "Type" -Value "NTP"
</code></pre>
<ul>
<li>Puis relancer le service NTP depuis une invite de commande sur le MS-AD :</li>
</ul>
<pre class=" language-cmd"><code class="prism  language-cmd">net stop w32time
net start w32time
</code></pre>
<ul>
<li><strong>Une fois toutes ces étapes passés, si on exécute net share on doit voir les partages:</strong>
<ul>
<li>NETLOGON</li>
<li>SYSVOL</li>
</ul>
</li>
</ul>
<p><strong>NOTE:</strong><br>
La GPO “Synchro_Temps_NTP” va poser problème à la dépromote de Samba:<br>
Modifier pour quelle pointe sur un des AD et non plus sur Samba<br>
<strong>On modifiera à la fin du process cette GPO pour qu’elle pointe sur le dernier AD restant</strong></p>
<h2 id="dépromote-de-samba">Dépromote de Samba:</h2>
<p>Eteindre les services Samba. Toutefois, on maintiendra encore un petit moment <em>oc.addc.epf.oc</em> en fonctionnement pour continuer à bénéficier de la souplesse des commandes <strong>samba-tool</strong> pour certaines opérations ultérieures.</p>
<pre class=" language-cmd"><code class="prism  language-cmd">systemctl stop samba
systemctl disable samba
</code></pre>
<p>Supprimer le dernier contrôleur de domaine Samba en lançant la commande suivante.<br>
On pointe l’exécution de la commande sur le MS-AD <em>EPF-AD-2008.epf.oc</em> :</p>
<pre class=" language-cmd"><code class="prism  language-cmd">samba-tool domain demote --remove-other-dead-server=oc-addc -H ldap://EPF-AD-2008.epf.oc -U Bruno
</code></pre>
<p>On en profite pour supprimer aussi l’AD EPF-2008R2-2025</p>
<pre class=" language-cmd"><code class="prism  language-cmd">samba-tool domain demote --remove-other-dead-server=EPF-2008R2-2025 -H ldap://EPF-AD-2008.epf.oc -U Bruno
</code></pre>
<p>Vérifier que les rôles FSMO ont été transférés lors du dernier <em>demote</em>. Il restera les rôles <em>DomainDnsZones</em> et <em>ForestDNSZones</em> qui n’auront pas été transférés, on force le transfert :</p>
<pre class=" language-cmd"><code class="prism  language-cmd">samba-tool fsmo show -H ldap://EPF-AD-2008.epf.oc -U Bruno
samba-tool fsmo seize --role=all -H ldap://EPF-AD-2008.epf.oc -U Bruno
</code></pre>
<ul>
<li>
<p>Nettoyer les entrées DNS. Dans une console DNS ouverte sur  <em>EPF-AD-2008.epf.oc</em>  vérifier que les entrées DNS pour  <em>EPF-AD-2008.epf.oc</em>  sont bien toutes présentes (champs A, NS, SRV, CNAME) et supprimer les références DNS à  <em>oc-addc.epf.oc</em>. On corrigera aussi les  <em>GLUE records</em>  (champ de type NS) pour le champ  _<em>msdcs</em>  dans la zone  <em>mydomain.lan</em> ainsi que pour la zone <em>msdcs.mydomain.lan</em>.</p>
</li>
<li>
<p>Créer la zone inverse si elle n’existe pas encore puis créer le champ PTR pour  <em>EPF-AD-2008.epf.oc</em>.</p>
</li>
</ul>
<p><strong>Désormais nous avons un domaine full Microsoft fonctionnel avec un seul contrôleur de domaine.</strong></p>
<p><strong>NOTE:</strong></p>
<blockquote>
<p>Vérifier la configuration de la carte réseau pour la partie DNS, elle doit pointer sur 127.0.0.1 ou l’IP du serveur lui même et plus sur Samba.</p>
</blockquote>
<p><strong>NOTE:</strong></p>
<blockquote>
<p>Remarques:<br>
A ce stade il faut normalement faire un DcDiag et regarder l’état de santé de l’AD et s’il y a des échecs.<br>
Attention, il se peu que des erreurs antérieures à la migration soient rapporté dans le diag.<br>
Etant sur un 2008R2 on va trouver des erreurs comme:<br>
Problème : valeur attendue manquante<br>
Nom d’attribut de l’objet valeur : frsComputerReferenceBL<br>
Cette erreur est expliqué ici:<a href="https://learn.microsoft.com/fr-fr/previous-versions/troubleshoot/windows-server/dcdiag-verifyreferences-test-fails">DCDiag VerifyReferences échoue</a></p>
</blockquote>
<p><strong>Dans l’idéal, il faut quand même migrer cet AD vers un AD au minimum en 2012 afin de valider la réplication SYSVOL mais surtout pour réinitialiser la partie DFSR</strong></p>
<h2 id="migration-de-lad-en-2008r2-vers-un-ad-en-2012r2">Migration de l’AD en 2008R2 vers un AD en 2012R2</h2>
<ul>
<li><strong>Prérequis:</strong>
<ul>
<li>IP statique</li>
<li>DNS Principal celui de l’AD en 2008R2</li>
<li>Changer les DNS de 2008Ré pour pointer vers 2012R2</li>
<li>Renommer avant jonction au domaine ou avant promote</li>
</ul>
</li>
</ul>
<p>Sur 2012R2 on ajoute le rôle ADDS et DNS<br>
Une fois le rôle ADDS installé, promouvoir en DC<br>
Sur  <em>EPF-AD-2008</em>, vérifier la réplication :</p>
<pre class=" language-cmd"><code class="prism  language-cmd">repadmin /kcc
repadmin /showrepl
</code></pre>
<p>Si tout est à l’état “Réussi” on démromote 2008R2<br>
Sur la VM Samba:<br>
Modifier le fichier resolv.conf:</p>
<pre class=" language-cmd"><code class="prism  language-cmd">nano /etc/resolv.conf
</code></pre>
<pre class=" language-cmd"><code class="prism  language-cmd">search EPF.OC
nameserver 10.34.1.226
</code></pre>
<p>Puis dépromote de 2008:</p>
<pre class=" language-cmd"><code class="prism  language-cmd">samba-tool domain demote --remove-other-dead-server=EPF-AD-2008 -H ldap://EPF-AD-2012.epf.oc -U Bruno
</code></pre>
<p>Vérifier que Samba-Tool à bien migré les roles FSMO à 2012R2</p>
<pre class=" language-cmd"><code class="prism  language-cmd">netdom query fsmo
</code></pre>
<p><strong>Supprimer l’objet EPF-AD-2008 de l’AD depuis la console Utilisateurs et Ordinateurs Active Directory</strong></p>
<p>Forcer l’activation du répertoire Sysvol sur le MS-AD :<br>
Dans PowerShell en Admin taper:</p>
<pre class=" language-cmd"><code class="prism  language-cmd">Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Netlogon\Parameters" -Name "SysvolReady" -Value "1"
</code></pre>
<ul>
<li>Recopier le contenu du  <code>sysvol</code>  depuis le serveur EPF-AD-2008. Pour ce faire, dans un explorateur de fichier, tapez  <code>\\epf-ad-2008\sysvol\</code>, puis aller dans le dossier correspondant au nom de votre domaine (par exemple  <em>epf.oc</em>) et copier  <code>Policies</code>  et  <code>Scripts</code>  dans  <code>C:\windows\SYSVOL\domain</code>  (mais pas le nom de domaine). Après la copie on aura donc ces deux répertoires :
<ul>
<li><code>C:\windows\SYSVOL\domain\Policies</code>  ;</li>
<li><code>C:\windows\SYSVOL\domain\Scripts</code>  ;</li>
</ul>
</li>
</ul>
<p><strong>Note</strong><br>
Il y a un lien de  <code>C:\windows\SYSVOL\sysvol\ad.mydomain.lan</code>  vers  <code>C:\windows\SYSVOL\domain</code>.</p>
<ul>
<li>Redémarrer le serveur MS-AD :</li>
</ul>
<pre class=" language-cmd"><code class="prism  language-cmd">shutdown -r -t 0
</code></pre>
<ul>
<li>
<p>Changer les serveurs DNS de la carte réseau. Le serveur DNS doit être lui même (<code>127.0.0.1</code>), et en deuxième on laisse vide.</p>
</li>
<li>
<p>Modifier ensuite la configuration NTP dans la base de registre du MS-AD :</p>
</li>
</ul>
<pre class=" language-cmd"><code class="prism  language-cmd">Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\W32Time\Parameters" -Name "Type" -Value "NTP"
</code></pre>
<ul>
<li>Puis relancer le service NTP depuis une invite de commande sur le MS-AD :</li>
</ul>
<pre class=" language-cmd"><code class="prism  language-cmd">net stop w32time
net start w32time
</code></pre>
<ul>
<li><strong>Une fois toutes ces étapes passés, si on exécute net share on doit voir les partages:</strong>
<ul>
<li>NETLOGON</li>
<li>SYSVOL</li>
</ul>
</li>
</ul>
<p><strong>Si tout est bon, on peut éteindre le serveur 2008</strong></p>
<ul>
<li><strong>Nettoyer dans cet ordre:</strong>
<ul>
<li>ADSI</li>
<li>Sites et Services Active Directory</li>
<li>DNS</li>
</ul>
</li>
</ul>
<p><strong>NOTE:</strong><br>
La GPO “Synchro_Temps_NTP”:<br>
Modifier “NtpServer” “10.34.1.227” pour pointer l’ip de l’AD 2012R2<br>
Actualiser les GPO sur le serveur et un client puis vérifier:</p>
<pre class=" language-cmd"><code class="prism  language-cmd">w32tm /query /status
w32tm /query /configuration
</code></pre>
<h2 id="migration-fsr-vers-dfsr">Migration FSR vers DFSR</h2>
<h3 id="vérifier-létat-de-santé-de-lad-avant-de-lancer-la-migration">Vérifier l’état de santé de l’AD avant de lancer la migration</h3>
<pre class=" language-cmd"><code class="prism  language-cmd">dcdiag /e /test:sysvolcheck /test:advertising
</code></pre>
<pre class=" language-cmd"><code class="prism  language-cmd">Diagnostic du serveur d'annuaire
Exécution de l'installation initiale :
   Tentative de recherche de serveur associé...
   Serveur associé : EPF-AD-2012
   * Forêt AD identifiée.
   Collecte des informations initiales terminée.
Exécution des tests initiaux nécessaires
   Test du serveur : Default-First-Site-Name\EPF-AD-2012
      Démarrage du test : Connectivity
         ......................... Le test Connectivity
          de EPF-AD-2012 a réussi
Exécution des tests principaux
   Test du serveur : Default-First-Site-Name\EPF-AD-2012
      Démarrage du test : Advertising
         ......................... Le test Advertising
          de EPF-AD-2012 a réussi
      Démarrage du test : SysVolCheck
         ......................... Le test SysVolCheck
          de EPF-AD-2012 a réussi
   Exécution de tests de partitions sur ForestDnsZones
   Exécution de tests de partitions sur DomainDnsZones
   Exécution de tests de partitions sur Schema
   Exécution de tests de partitions sur Configuration
   Exécution de tests de partitions sur epf
   Exécution de tests d'entreprise sur epf.oc
</code></pre>
<p><strong>NOTE:</strong><br>
Il faut que <strong>tout</strong> soit en <strong>“réussi”</strong> pour lancer la migration</p>
<ul>
<li><strong>Si tout est bon, on lance la migration qui se fera par étape:</strong>
<ul>
<li>Lancement d’une étape</li>
<li>Vérification de l’avancé de l’étape</li>
</ul>
</li>
<li><strong>On ne passe pas à l’étape d’après tant que ce n’est pas validé</strong></li>
</ul>
<ol>
<li>Lancement du mode “Préparation”:</li>
</ol>
<pre class=" language-cmd"><code class="prism  language-cmd">dfsrmig /setglobalstate 1
</code></pre>
<p>Vérification:</p>
<pre class=" language-cmd"><code class="prism  language-cmd">dfsrmig /getmigrationstate
</code></pre>
<p><strong>Relancer la commande de temps en temps… Jusqu’à obtenir ceci :</strong></p>
<pre class=" language-cmd"><code class="prism  language-cmd">C:\Windows\system32&gt;Dfsrmig /getmigrationstate
Tous les controleurs de domaine ont migre vers l'etat Global (&lt; Prepare &gt;).
La migration a atteint un etat coherent sur tous les controleurs de domaine.
Réussi.
</code></pre>
<ol start="2">
<li>Lancement de l’étape “Redirigé”:</li>
</ol>
<pre class=" language-cmd"><code class="prism  language-cmd">dfsrmig /setglobalstate 2
</code></pre>
<p>Vérification:</p>
<pre class=" language-cmd"><code class="prism  language-cmd">dfsrmig /getmigrationstate
</code></pre>
<p><strong>Relancer la commande de temps en temps… Jusqu’à obtenir ceci :</strong></p>
<pre class=" language-cmd"><code class="prism  language-cmd">C:\Windows\system32&gt;Dfsrmig /getmigrationstate
Tous les controleurs de domaine ont migre vers l'etat Global (&lt; Redirigé &gt;).
La migration a atteint un etat coherent sur tous les controleurs de domaine.
Réussi.
</code></pre>
<ol start="3">
<li>Dernière étape, Retirer FSR et son SYSVOL pour passer sur DFSR:</li>
</ol>
<pre class=" language-cmd"><code class="prism  language-cmd">dfsrmig /setglobalstate 3
</code></pre>
<p>Vérification:</p>
<pre class=" language-cmd"><code class="prism  language-cmd">dfsrmig /getmigrationstate
</code></pre>
<p><strong>Relancer la commande de temps en temps… Jusqu’à obtenir ceci :</strong></p>
<pre class=" language-cmd"><code class="prism  language-cmd">C:\Windows\system32&gt;Dfsrmig /getmigrationstate
Tous les controleurs de domaine ont migre vers l'etat Global (&lt; Eliminé &gt;).
La migration a atteint un etat coherent sur tous les controleurs de domaine.
Réussi.
</code></pre>
<h3 id="vérifier-que-le-dossier-sysvol-et-renommé-en-sysvol_dfsr">Vérifier que le dossier SYSVOL et renommé en SYSVOL_DFSR</h3>

