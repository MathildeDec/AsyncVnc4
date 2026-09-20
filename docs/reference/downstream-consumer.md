# Consommateur en aval (pluginvnc2)

Le plugin GTK4 `pluginvnc2` (paquet séparé, `pluginvnc2.zip`) consomme ce
fork. Points de couplage à connaître si tu modifies ce fichier :

- `pluginvnc2` gère déjà le repli automatique si le paramètre `shared` de
  `connect()` n'existe pas encore (détection par `TypeError`) — ne pas
  retirer ce paramètre sans vérifier ce repli côté `vnc_tab.py`.
- `pluginvnc2` ne câble PAS `send_qemu_extended_key_event()` aujourd'hui
  (aucune trace de `qemu` dans `vnc_tab.py`) — normal, `pluginvnc2` n'a
  pas encore connaissance de la vérification faite dans cette session
  (voir `features.md`, section « Client requests »). Depuis
  **2026-09-02** le format est vérifié contre le code source QEMU
  officiel (✅, plus ⚠️) donc un futur câblage côté GTK4 peut s'appuyer
  dessus avec la même confiance que xvp ci-dessous — sous réserve du
  test contre un vrai serveur QEMU/Proxmox, seul point encore non
  couvert (voir section dédiée plus haut).
- `pluginvnc2` ne câble pas non plus xvp (`send_xvp_shutdown()`/`_reboot()`/
  `_reset()`) — normal, le patch vient d'être ajouté dans ce paquet-ci,
  `pluginvnc2` n'en a pas encore connaissance. Le format xvp est vérifié
  contre la spec officielle (✅) donc un futur câblage côté GTK4 (par ex.
  un bouton « redémarrer la VM » dans l'onglet) peut s'appuyer dessus
  avec confiance — sous réserve du test contre un vrai serveur mentionné
  plus haut. Il faudrait aussi exposer `Client.xvp_supported` côté GTK4
  pour savoir quand afficher ce genre de bouton. **Depuis le 2026-09-06**,
  un futur câblage GTK4 devra en plus obtenir sa propre confirmation
  utilisateur (boîte de dialogue) avant de passer `confirm=True` à
  `send_xvp_shutdown()`/`_reboot()`/`_reset()` — sans quoi ces appels
  lèvent désormais une `PermissionError` (voir « Confirmation avant
  commande xvp destructive » dans `features.md`).
- `pluginvnc2` ne câble pas non plus `listen()` aujourd'hui — normal,
  tout juste ajouté ici. Contrairement à xvp, câbler `listen()` côté
  GTK4 n'est pas un simple ajout : `vnc_tab.py` suppose aujourd'hui que
  c'est toujours l'utilisateur qui compose vers un hôte (`connect()`),
  alors que la reconnexion inversée demande un mode "écouter une
  connexion entrante" à part dans l'UI (choix du port d'écoute,
  affichage "en attente de connexion...", etc.) plutôt qu'un simple
  paramètre optionnel sur le formulaire existant.
- VeNCrypt (TLSNone/TLSVnc, X509None/X509Vnc depuis le 2026-09-05, et
  TLSPlain/X509Plain depuis le **2026-09-14**) reste absent des deux
  paquets — `pluginvnc2` n'expose aucun champ `ssl_context`/CA dans
  `vnc_tab.py`. Pour le cas nominal, cela ne demanderait qu'un
  paramètre optionnel de plus à faire passer vers `connect()` (voir
  `features.md` § « Dépendant en aval ») ; le pinning de clé hôte côté
  `pluginvnc2` (`HostKeyStore`) ne couvre lui que l'auth Apple ARD pour
  l'instant, pas un futur TLS/X.509. **Les sous-types Plain
  changeraient un peu la donne côté UI** : contrairement aux autres
  sous-types VeNCrypt (qui ne demandent rien de plus que ce que
  `vnc_tab.py` sait déjà collecter — un mot de passe, pour "Vnc"), Plain
  exige aussi un **nom d'utilisateur**, champ que le formulaire de
  connexion actuel de `pluginvnc2` ne semble pas exposer (à vérifier
  côté `vnc_tab.py` — l'auth Apple, seul autre cas déjà utilisant
  `username` dans ce fork, n'a peut-être pas eu besoin d'un champ dédié
  si `pluginvnc2` la câble avec une valeur fixe ou absente).
- **RSA-AES (RA2/RA2_256, ajouté le 2026-09-15)** reste lui aussi
  absent des deux paquets. Même besoin de champ `username` que pour
  VeNCrypt Plain ci-dessus (le sous-type "mot de passe seul" existe
  aussi côté RSA-AES, donc ce n'est pas garanti nécessaire selon le
  serveur visé, mais un serveur configuré comme celui utilisé pour
  vérifier ce patch — `-RequireUsername=1` — l'exige). Contrairement à
  VeNCrypt, aucun paramètre `ssl_context`/CA ne serait nécessaire ici
  (RSA-AES ne s'appuie pas sur `ssl`/des autorités de certification) ;
  en revanche, `Client.rsa_aes_server_key_fingerprint` (une chaîne
  `%02x-%02x-...`, disponible après une connexion réussie) est
  directement du même type de donnée que ce que `HostKeyStore` sait déjà
  gérer pour l'auth Apple ARD — un épinglage RSA-AES côté `pluginvnc2`
  réutiliserait vraisemblablement la même mécanique de stockage/
  confirmation utilisateur, pas une nouvelle à inventer.
- Le repli de version de protocole ne demande AUCUNE adaptation côté
  `pluginvnc2` : entièrement interne à `Client.create()`, transparent
  pour l'appelant. Le seul ajout visible est le champ informatif
  `Client.protocol_version` (tuple `(3, 3)`/`(3, 7)`/`(3, 8)`), que
  `pluginvnc2` peut ignorer sans risque ou afficher en diagnostic s'il
  veut un jour.
- **Extended Clipboard** (ajouté le 2026-09-13, `pluginvnc2` n'en a pas
  encore connaissance) — côté réception, aucune adaptation requise :
  `Enc.EXTENDED_CLIPBOARD` est inclus dans `Enc.default()`, donc
  `client.clipboard.text` reçoit déjà le texte UTF-8 étendu dès lors que
  `pluginvnc2` utilise `connect()` tel quel. Côté émission, câbler un
  copier-coller vers le serveur distant demanderait d'appeler
  `client.send_clipboard_update(text)` (pas `send_clipboard_provide()`
  seul, qui suppose qu'un `Request` a déjà été reçu — voir
  `features.md`) ; ceci exige au préalable `client.clipboard_ext_supported`,
  qui ne passe à `True` qu'après le premier message du serveur, donc un
  bouton/raccourci "coller" côté GTK4 devrait rester désactivé (ou
  silencieusement retomber sur `Clipboard.write()`, Latin-1 seul) tant
  que ce n'est pas le cas. Aucun nouveau paramètre sur `connect()`/
  `listen()`/`Client.create()`, contrairement à VeNCrypt ci-dessus.

- Depuis **2026-09-06**, `asyncvnc2.py` importe `loguru` (nouvelle
  dépendance tierce, absente jusqu'ici) pour la journalisation d'audit
  de `send_xvp()`/`send_qemu_extended_key_event()`/
  `send_set_desktop_size()` (voir section dédiée ci-dessus).
  `pluginvnc2` doit donc désormais lister `loguru` dans ses propres
  dépendances s'il ne l'avait pas déjà (à vérifier côté `pluginvnc2`) ;
  sans configuration de sink explicite côté `pluginvnc2`, ces logs
  sortent sur `stderr` par défaut (comportement loguru standard), donc
  rien ne casse si `pluginvnc2` ne fait rien — mais il ne les verra pas
  dans ses propres fichiers de logs tant qu'il n'appelle pas
  `logger.add(...)` à son propre démarrage.

Voir `features.md` de ce paquet pour l'état complet
encodage-par-encodage/extension-par-extension (créé le 2026-09-01, mis à
jour la même journée avec xvp, `listen()` puis le repli de version, puis
le 2026-09-02 avec la vérification source de
`send_qemu_extended_key_event()` puis avec le nettoyage de style partiel
ci-dessus) — ce fichier `CLAUDE.md` reste la référence pour le
« pourquoi » et les détails de vérification.

