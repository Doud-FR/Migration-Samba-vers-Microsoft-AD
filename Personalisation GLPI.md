<!DOCTYPE html>
<html>

<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Personalisation GLPI</title>
  <link rel="stylesheet" href="https://stackedit.io/style.css" />
</head>

<body class="stackedit">
  <div class="stackedit__html"><h1 id="personalisation-glpi">Personalisation GLPI</h1>
<p><strong>Conseil</strong></p>
<p><strong>Fais toujours une sauvegarde</strong> de tes logos/icônes modifiés et documente les changements pour pouvoir les remettre après une mise à jour.</p>
<h3 id="sommaire">Sommaire:</h3>
<ol>
<li>
<p><strong>Changer le logo (en haut à gauche)</strong></p>
</li>
<li>
<p><strong>Changer l’icône/favicone</strong></p>
</li>
<li>
<p><strong>Changer le texte d’accueil / page de login</strong></p>
</li>
<li>
<p><strong>Faire d’autres customisations sympas</strong> (bannière, CSS, thème, etc.)</p>
</li>
</ol>
<hr>
<h2 id="changer-le-logo-de-glpi"><strong>1. Changer le logo de GLPI</strong></h2>
<h3 id="a.-par-l’interface-méthode-officielle-depuis-glpi-10"><strong>A. Par l’interface (méthode officielle, depuis GLPI 10)</strong></h3>
<ol>
<li>
<p>Va dans <strong>Configuration &gt; Générale &gt; Apparence</strong></p>
</li>
<li>
<p>Tu trouveras une section <strong>Logo</strong> (et parfois “Logo de connexion”)</p>
</li>
<li>
<p><strong>Téléverse</strong> (upload) l’image/logo de ton client.</p>
<ul>
<li>
<p>Format conseillé : PNG ou SVG (fond transparent)</p>
</li>
<li>
<p>Dimensions typiques : 200 × 40 px (pour le menu), ou adapte selon rendu</p>
</li>
</ul>
</li>
<li>
<p><strong>Enregistre</strong><br>
Le logo apparaît immédiatement (rafraîchis la page).</p>
</li>
</ol>
<hr>
<h3 id="b.-par-fichier-si-interface-absente"><strong>B. Par fichier (si interface absente)</strong></h3>
<ul>
<li>
<p>Remplace manuellement le fichier dans <code>/var/www/glpi/public/img/</code></p>
</li>
<li>
<p>Exemple : remplace <code>logo-glpi.svg</code> par ton propre logo (sauvegarde l’original !)</p>
</li>
<li>
<p>Mais cette méthode sera <strong>écrasée à la prochaine mise à jour</strong> → privilégie l’interface.</p>
</li>
</ul>
<hr>
<h2 id="changer-la-favicon-petite-icône-du-navigateur"><strong>2. Changer la favicon (petite icône du navigateur)</strong></h2>
<ul>
<li>
<p>Toujours dans <strong>Configuration &gt; Générale &gt; Apparence</strong></p>
</li>
<li>
<p>Option “Favicon” ou “Icône de site”</p>
</li>
<li>
<p><strong>Téléverse</strong> une icône carrée (PNG, idéalement 32x32 ou 64x64 px)</p>
</li>
</ul>
<hr>
<h2 id="changer-le-message-d’accueil--de-connexion"><strong>3. Changer le message d’accueil / de connexion</strong></h2>
<ul>
<li>
<p><strong>Configuration &gt; Générale &gt; Système</strong></p>
</li>
<li>
<p>“Message d’accueil” ou “Message de la page de connexion”</p>
</li>
<li>
<p>Personnalise le texte HTML (tu peux mettre liens, consignes, infos de contact, etc.)</p>
</li>
<li>
<p>Affiche sur la page de login ou tableau de bord selon option</p>
</li>
</ul>
<hr>
<h2 id="modifier-couleursthème"><strong>4. Modifier couleurs/thème</strong></h2>
<ul>
<li>
<p><strong>Thèmes</strong> : GLPI propose un mode sombre, et quelques variations de couleurs dans<br>
<strong>Configuration &gt; Préférences &gt; Apparence</strong></p>
</li>
<li>
<p>Pour un thème plus poussé, tu peux :</p>
<ul>
<li>
<p>Utiliser un plugin (ex : <a href="https://github.com/lutitech/glpi-modern">Modern Theme</a>)</p>
</li>
<li>
<p>Ajouter une CSS personnalisée via le menu “Apparence avancée” (si disponible)</p>
</li>
<li>
<p>Modifier le fichier <code>/public/css</code> (attention, non pérenne après mise à jour)</p>
</li>
</ul>
</li>
</ul>
<hr>
<h2 id="astuces-supplémentaires"><strong>5. Astuces supplémentaires</strong></h2>
<ul>
<li>
<p><strong>Ajoute un lien personnalisé vers le support</strong> (menu de gauche ou en haut) via “Configuration &gt; Liens externes”</p>
</li>
<li>
<p><strong>Personnalise le footer</strong> (bas de page) : toujours via “Générale &gt; Apparence”</p>
</li>
<li>
<p><strong>Multilingue</strong> : adapte le nom des entités, messages, menus pour l’utilisateur final</p>
</li>
</ul>
</div>
</body>

</html>
