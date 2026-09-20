# Historique des patches — vue d'ensemble

Table de référence de tous les patches appliqués à `asyncvnc2.py` depuis
le fork d'origine, avec leur niveau de vérification. Pour le détail du
raisonnement de chaque session, voir `docs/sessions/`.

## Ce que ce zip contient

`asyncvnc2.py` = ton `asyncvnc2-1.py` original (Tight/Hextile/ZlibHex,
curseur, resize, Fence/ContinuousUpdates — ton propre fork, déjà testé
21/21 par tes soins contre TigerVNC/x11vnc réels) **+ 5 patches d'une
session précédente + xvp + la reconnexion inversée (`listen()`) + le
repli de version de protocole + VeNCrypt (sous-types TLSNone/TLSVnc,
puis X509None/X509Vnc)**, ces quatre derniers ajoutés dans des sessions
ultérieures le même jour (VeNCrypt complété par les sous-types
X509None/X509Vnc le 2026-09-05), chacun vérifié par exécution réelle
dans ce sandbox (pas juste relu) :

| # | Patch | Vérification effectuée |
|---|-------|------------------------|
| 1 | `shared` flag (ClientInit) | `inspect.signature()` sur `connect()`/`Client.create()`/`Video.create()` |
| 2 | Mot de passe VNC Auth non-ASCII (repli Latin-1) | Encodage testé avec un mot de passe accentué réel |
| 2 | Troncature ARD silencieuse → `ValueError` explicite | Testé avec un identifiant de 64 octets (doit lever) et 63 (doit passer) |
| 3 | Plafonds anti-DoS (Tight ×2, ZlibHex ×2, curseurs ×3) | `_check_alloc_size()` testé, callers vérifiés un par un |
| 4 | Encodages RRE (3) et CoRRE (4) | **Décodage réel testé** contre un flux RRE/CoRRE fabriqué à la main (fond + sous-rectangle, pixel-parfait) ; garde-fou 255×255 de CoRRE vérifié |
| 5 | `send_set_desktop_size()` (RFC officielle) | **Format binaire vérifié octet par octet** (1 écran implicite : 24 octets ; 2 écrans explicites : 40 octets) |
| 5 | `send_qemu_extended_key_event()` (extension QEMU) | **2026-09-02** — format vérifié contre le code source QEMU officiel (`github.com/qemu/qemu`, `ui/vnc.c`, commit `a925240509d1b4b656cc480f1cc79ba4d7c8bc08`), plus depuis la mémoire ; implémentation déjà conforme, aucune correction nécessaire — voir section dédiée ci-dessous |
| 6 | xvp (`Enc.XVP`, `UpdateType.XVP`, `Client.process_xvp()`, `send_xvp_shutdown()`/`_reboot()`/`_reset()`) | **Format vérifié contre la spec RFB officielle** (`rfbproto.rst`, sections « xvp Client Message » / « xvp Server Message » / « xvp Pseudo-encoding » — message type 250 bidirectionnel, pseudo-encoding -309) ; décodage `XVP_INIT`/`XVP_FAIL` et encodage des 3 requêtes (`XVP_SHUTDOWN`/`XVP_REBOOT`/`XVP_RESET`) testés octet par octet contre un flux fabriqué à la main dans ce sandbox — **pas testé contre un vrai serveur supportant xvp** (aucun disponible ici) |
| 7 | Reconnexion inversée (`listen()`) | **Testé par exécution réelle** : vrai socket TCP loopback dans ce sandbox, un script jouant le serveur VNC distant (ProtocolVersion/Security-None/SecurityResult/ServerInit fabriqués à la main) se connecte au listener, `Client` obtenu vérifié utilisable de bout en bout (dimensions + nom lus correctement) — a débusqué et corrigé un vrai deadlock en cours de route, voir section dédiée ci-dessous |
| 8 | Repli de version de protocole (`Client.protocol_version`, 3.3/3.7/3.8) | **Format vérifié contre RFC 6143 officielle** (§7.1.1/§7.1.2/§7.1.3 + Appendix A) ; **7 scénarios testés de bout en bout sur de vrais sockets TCP loopback** couvrant 3.8+None, 3.8+VNC Auth (régression), 3.7+None (SecurityResult sauté), 3.3+None et 3.3+VNC Auth (négociation U32 unilatérale), minor 3.x inconnu (repli 3.3), et major > 3 façon RealVNC Enterprise (traité comme 3.8, confirmé par un rapport de bug tiers réel) — voir section dédiée ci-dessous |
| 9 | VeNCrypt (`_vencrypt_negotiate()`, `_start_tls_client()`, security type 19, sous-types `TLSNone`/`TLSVnc`), nouveau paramètre `ssl_context` sur `Client.create()`/`connect()`/`listen()` | **2026-09-02** — format vérifié contre la spec communautaire officielle `rfbproto.rst` **et** contre l'implémentation de référence TigerVNC (`CSecurityVeNCrypt.cxx`/`CSecurityTLS.cxx`) ; une divergence réelle avec un tout premier brouillon de spec (mailing-list, 2010) a été trouvée et corrigée en cours de session (voir section dédiée ci-dessous) ; **négociation de version 0.2 + sélection de sous-type + bascule TLS testées de bout en bout sur un vrai socket TCP loopback**, pour TLSNone et pour TLSVnc (VNC Authentication standard réutilisée sur le flux chiffré), avec un faux serveur généré à la main et un certificat auto-signé `openssl` ; non-régression vérifiée (un serveur offrant `[None, VeNCrypt]` continue de choisir None comme avant ce patch) — **pas testé contre un vrai serveur exigeant VeNCrypt**, voir section dédiée ci-dessous |
| 10 | RFB-sur-WebSockets, rôle client (`_ws_handshake()`, `_ws_pump()`, `_WebSocketWriter`, `_ws_wrap_connection()`, nouveaux paramètres `websocket`/`ws_path` sur `connect()`) | **2026-09-03, élevé au niveau vrai serveur le même jour** — format vérifié contre RFC 6455 officielle (§4.1/§4.2 poignée de main, §5.2 format de trame, §5.3 masquage client obligatoire, §5.5.2 ping/pong) ; d'abord testé sur un faux serveur RFB-sur-WS fabriqué à la main, **puis contre un vrai processus `websockify`** en `ws://` et en `wss://` (TLS) — un vrai bug de fermeture propre à la combinaison WebSocket+TLS a été trouvé et corrigé au passage ; non-régression vérifiée — voir section dédiée ci-dessous |
| 11 | RFB-sur-WebSockets, rôle serveur (`_ws_handshake_server()`, `_ws_wrap_connection_server()`, nouveau paramètre `websocket` sur `listen()`) | **2026-09-04** — même format RFC 6455, appliqué en sens inverse (accepte une poignée de main WebSocket au lieu d'en initier une, trames sortantes non masquées conformément à §5.3) ; **testé de bout en bout contre un vrai client WebSocket tiers** (bibliothèque Python `websockets`, indépendante de ce fork), qui joue le rôle d'un serveur VNC distant se reconnectant à travers un proxy WebSocket ; non-régression vérifiée (`listen()` sans `websocket=True` inchangé). **Complété le 2026-09-10** : la topologie « reverse » complète (un proxy tiers relayant une connexion TCP entrante vers une connexion WebSocket sortante) restait non couverte — **testée pour de vrai contre un vrai `websocat`** (binaire officiel indépendant, `tcp-listen:` → `ws://`) ; au passage, `websockify` (pressenti à tort comme le proxy à utiliser pour ce rôle précis, cf. commentaires du code) s'est révélé **structurellement incapable** de ce sens de relais (confirmé par son propre `--help` et par exécution réelle de l'invocation erronée qui figurait en commentaire) — docstrings/commentaires corrigés en conséquence, zéro ligne de code exécutable modifiée (diff vérifié) — voir `docs/sessions/session-12-2026-09-10.md` |
| 12 | VeNCrypt, sous-types `X509None`/`X509Vnc` (authentifiés par un vrai certificat CA), nouveau paramètre `server_hostname` sur `Client.create()` (renseigné automatiquement par `connect()`) | **2026-09-05** — vraie chaîne de certification (CA racine + certificat serveur, SAN `IP:127.0.0.1`) générée avec `openssl` dans ce sandbox, contrairement au certificat auto-signé anonyme du patch 9 ; **testé de bout en bout sur un vrai socket TCP loopback** pour `X509None` et `X509Vnc` (VNC Auth réelle par-dessus), **rejet réel confirmé avec une CA non liée** (`ssl.SSLCertVerificationError` authentique), ordre de préférence à 4 sous-types vérifié, comportement par défaut (`ssl.create_default_context()`) testé en échec sûr et en réussite simulée, non-régression du patch 9 revérifiée ; une hypothèse fausse sur le comportement de `server_hostname=None` (cas `listen()`) a été débusquée et corrigée en cours de session — voir section dédiée ci-dessous |
| 13 | Journalisation d'audit (`from loguru import logger`) sur `send_xvp()`, `send_qemu_extended_key_event()`, `send_set_desktop_size()` | **2026-09-06** — comble l'écart n°2 identifié par l'audit `PATTERNS.md` du même jour (`switch-capture::log_outbound_command`) ; `send_xvp()` en `logger.warning()` (commande potentiellement destructive), les deux autres en `logger.info()` ; **testé par exécution réelle** dans ce sandbox (faux `writer` + sink `loguru` captant les logs), aucun format binaire modifié, cas d'erreur `send_xvp(code invalide)` confirmé sans log ni écriture — voir section dédiée ci-dessous |
| 14 | `confirm: bool = False` sur `send_xvp()`/`send_xvp_shutdown()`/`_reboot()`/`_reset()` | **2026-09-06** — comble l'écart n°1 identifié par le même audit `PATTERNS.md` : `PermissionError` levée avant tout envoi si `confirm=True` n'est pas passé explicitement, avec son propre log `logger.warning()` de refus ; **testé par exécution réelle** (sans `confirm` → `PermissionError` + zéro octet écrit ; avec `confirm=True` → écriture socket inchangée) ; code invalide toujours prioritaire (`ValueError`) sur la vérification de `confirm` — voir section dédiée ci-dessous |
| 15 | Nettoyage style `ruff check` sur le code WebSockets (`_WebSocketWriter.wait_closed()`, `_ws_pump()`) : suppression d'un `pass` redondant (`PIE790`) et d'une variable `exc` inutilisée (`F841`) | **2026-09-07** — étend aux patches 10/11 (WebSockets) le même traitement « sous-ensemble sûr » que la session du 2026-09-02 sur le reste du fichier ; zéro changement de comportement, voir section dédiée ci-dessous |
| 16 | `Enc.ZYWRLE = 17`, décodage partagé avec `Enc.ZRLE` (`process_xrle`), **exclu** de `Enc.default()` | **2026-09-09** — **testé pour de vrai contre un vrai QEMU 8.2.2** (TigerVNC/Xvnc 1.13.1 n'implémente pas ZYWRLE, confirmé par `strings` sur le binaire) : l'hypothèse « aucun changement client nécessaire » est infirmée — QEMU envoie de vraies tuiles avec un octet de subencoding dans la plage 17-126, réservée « unused » par la RFC, une fois son chemin lossy réellement engagé (`lossy=on` + `jpeg_quality<9`). Décodage standard (subencodings 0/1/2-16/127/128/130-255) confirmé identique à `Enc.ZRLE` par test réel puis par 3 tests committés ; la plage 17-126 est explicitement refusée (`ValueError`, message enrichi) plutôt que devinée — deux hypothèses de format essayées contre le vrai serveur, aucune concluante. Confirmé sans risque pour les appelants existants : `Enc.ZRLE` seul (sans ZYWRLE négocié) et `Enc.ZYWRLE` négocié avec `jpeg_quality=9` (qualité max) ne déclenchent jamais ce chemin, testés 5 fois de suite chacun contre le même serveur — voir `docs/sessions/session-11-2026-09-09.md` pour le détail complet |
| 17 | `Enc.EXTENDED_CLIPBOARD` (`0xC0A1E5CE`), constantes `CLIPBOARD_FORMAT_*`/`CLIPBOARD_ACTION_*`, `Client.process_extended_clipboard()`, `send_clipboard_caps/notify/request/provide/update()` | **2026-09-13** — format vérifié contre `rfbproto.rst` (« Extended Clipboard Pseudo-Encoding ») **et** contre deux implémentations de référence indépendantes lues directement (code source QEMU `ui/vnc.h`+`ui/vnc-clipboard.c`+`ui/vnc.c`, et le binaire `Xtigervnc` 1.13.1 lui-même, origine de cette extension) — identiques, aucun écart. **Testé pour de vrai contre les deux** le même jour : les deux serveurs ignorent silencieusement un `Provide` non précédé d'un `Notify` du même émetteur (raisons mécaniques différentes), d'où l'ajout de `send_clipboard_update()` (Notify puis Provide enchaînés) ; round-trip complet confirmé dans les deux sens contre TigerVNC (presse-papiers X réel via un vrai `xclip`, texte accentué + CJK) et en relais à deux connexions contre QEMU. Seul le format *text* (UTF-8) est décodé/encodé — rtf/html/dib/files sont correctement sautés dans le flux (jamais de désynchronisation) mais jamais décodés. 21 tests committés — voir `docs/sessions/session-14-2026-09-13.md` pour le détail complet. **2026-09-13 (2) : `send_clipboard_caps()` revérifiée contre le même TigerVNC/Xvnc réel** — confirme qu'elle a un effet réel et mesurable (pas seulement un format sur le fil correct) : avec `max_text_size=0`, le serveur passe de "fournit directement" à "notifie puis attend notre `Request`", conformément à `handleClipboardCaps()` lu dans `common/rfb/SConnection.cxx` — voir `docs/sessions/session-15-2026-09-13.md` |
| 18 | `_VENCRYPT_TLS_PLAIN = 259`, `_VENCRYPT_X509_PLAIN = 262` ; nouvelle branche dans `Client.create()` (envoi identifiant+mot de passe après la poignée de main TLS/X.509) ; ordre de préférence VeNCrypt étendu à 6 sous-types | **2026-09-14** — découverts en inspectant `Xtigervnc -help` (qui les propose réellement), jamais mentionnés dans ce fichier avant cette date. Format vérifié contre `rfbproto.rst` (section VeNCrypt, « Plain subtype ») **et** contre le client de référence TigerVNC (`CSecurityPlain.cxx`) — identiques (`U32` longueur-identifiant, `U32` longueur-mot-de-passe, puis les deux chaînes, UTF-8). Réutilise les paramètres `username`/`password` déjà acceptés par `Client.create()` pour l'authentification Apple (33) — aucun nouveau paramètre. **Testé pour de vrai contre TigerVNC/Xvnc 1.13.1 dès le départ** (contrairement aux 4 autres sous-types VeNCrypt, d'abord testés contre un faux serveur fabriqué à la main) : `SSecurityPlain` de TigerVNC délègue la vérification du mot de passe à PAM, donc un vrai utilisateur Linux + un service PAM (`PAMService tigervnc`, config `common-auth`/`common-account` du paquet Debian/Ubuntu) ont été mis en place dans ce sandbox pour l'occasion. Quatre scénarios vérifiés : identifiants corrects → connexion réussie ; mauvais mot de passe → `PermissionError` réelle (`SecurityResult` existant, code inchangé) ; utilisateur inexistant → même rejet ; identifiants manquants → `ValueError` avant tout envoi réseau. Aucun test unitaire ajouté — VeNCrypt dans son ensemble en est délibérément exclu de `test_asyncvnc2.py` depuis le 2026-09-07 (nécessiterait un faux serveur TLS complet). RSA-AES (`RA2`/`RA2ne`/`RA2_256`/`RA2ne_256`), également découvert dans ce même `-help`, volontairement **non implémenté** cette session (échange de clés RSA propre, hors périmètre d'une seule tâche) — voir `docs/features-backlog.md` § Authentification et `docs/sessions/session-16-2026-09-14.md` pour le détail complet |
| 19 | Constantes `RSA_AES_MIN_KEY_LENGTH`/`RSA_AES_MAX_KEY_LENGTH`/`RSA_AES_SUBTYPE_USERPASS`/`RSA_AES_SUBTYPE_PASS` ; primitives `_omac`/`_eax_seal`/`_eax_open`/`_eax_increment_counter` ; classes `_EAXReader`/`_EAXWriter` ; fonction `_rsa_aes_negotiate()` ; nouveau champ `Client.rsa_aes_server_key_fingerprint` ; sécurité 5/129 ajoutée à la boucle de préférence de `Client.create()` | **2026-09-15** — RSA-AES (5=RA2, AES-128+SHA-1 ; 129=RA2_256, AES-256+SHA-256), découvert la veille en même temps que VeNCrypt Plain mais volontairement différé (patch 18). Contrairement à VeNCrypt, pas un TLS enveloppé : le client génère sa propre paire RSA (même taille que celle du serveur, lue sur le fil), puis toute la session — SecurityResult inclus — passe par un chiffrement AES-EAX construit dans ce fichier (`cryptography` n'a pas de classe EAX native, contrairement à AESGCM/AESCCM/AESSIV/AESOCB3/ChaCha20Poly1305 — vérifié le 2026-09-15). Format croisé entre `rfbproto.rst` (« RSA-AES Security Type »/« RSA-AES-256 Security Type ») **et** le code source TigerVNC (`CSecurityRSAAES.cxx/.h` + `common/rdr/AESInStream/AESOutStream.cxx`) — plusieurs détails (taille réelle de la clé de session, ordre exact des préimages des deux hachages, cadrage EAX exact) ne sont pas fixés sans ambiguïté par la seule prose de la spec et viennent de cette seconde source. **Un vrai bug trouvé et corrigé par l'exécution réelle** : la clé de session était tronquée à 16 octets sans condition dans une première version — correct pour AES-128 (RA2), faux pour AES-256 (RA2_256, qui doit utiliser les 32 octets complets du condensé SHA-256, sans troncature) ; RA2 fonctionnait dès le premier essai contre le vrai serveur, RA2_256 échouait immédiatement avec un jeton EAX invalide — signal net qui a permis de localiser l'erreur tout de suite. **Testé pour de vrai contre TigerVNC/Xvnc 1.13.1** pour les deux variantes (un vrai utilisateur Linux + PAM, réutilisés du patch 18, plus une clé RSA serveur générée pour l'occasion via `-RSAKey` ; `-RequireUsername=1` nécessaire, sans quoi ce serveur bascule sur une vérification façon VNC Auth nécessitant `-Password`/`-PasswordFile` plutôt que PAM — piège distinct trouvé en cours de route) : poignée de main complète, **lecture réelle du framebuffer au travers du chiffrement** (pas seulement la poignée de main), écriture clavier chiffrée, rejets corrects (mauvais mot de passe, utilisateur inexistant, identifiants manquants → `ValueError` avant tout envoi réseau). Empreinte de la clé serveur façon RealVNC (`%02x-%02x-...`, même format que `CSecurityRSAAES.cxx`) exposée via `Client.rsa_aes_server_key_fingerprint` pour un épinglage éventuel côté application appelante — cette bibliothèque ne stocke ni ne compare elle-même aucune empreinte de référence (choix délibéré, voir `docs/reference/downstream-consumer.md`). 6 tests unitaires ajoutés pour les primitives EAX elles-mêmes (`_eax_seal`/`_eax_open`/`_eax_increment_counter` : fonctions pures, testables sans serveur, contrairement à la poignée de main RSA-AES complète volontairement non testée unitairement, même choix que pour le reste de VeNCrypt). **Non implémenté, délibérément** : RA2ne/RA2ne_256 (ne chiffrent que la poignée de main, pas la session — annule l'intérêt même de RA2) et RA2r/RA2r_256 (seconde ronde de dérivation de clé, propriété marginale pour ce fork) — voir `docs/sessions/session-17-2026-09-15.md` pour le détail complet |

| 20 | Nettoyage style `ruff check` sur `_WebSocketWriter.close()` : `except Exception: pass` → `except (OSError, RuntimeError, ssl.SSLError) as exc:` + `logger.debug()` | **2026-09-16** — comble les 2 derniers signalements préexistants (`S110`/`BLE001`), volontairement laissés de côté depuis le patch 15 (2026-09-07). Comportement fonctionnel inchangé (aucune exception ne remonte plus à l'appelant), mais **pas revérifié contre un vrai transport cassé** dans les trois cas ciblés — seule la non-régression des 43 tests existants a été confirmée. Accompagne la migration de l'outillage projet vers `uv` (`pyproject.toml`/`uv.lock` ajoutés, `requires-python = ">=3.10"` désormais formellement acté) — voir `docs/sessions/session-18-2026-09-16.md` pour le détail complet |

| 21 | `ruff format` complet sur `asyncvnc2.py` + `test_asyncvnc2.py`, avec `quote-style = "single"` configuré pour préserver les guillemets simples | **2026-09-16** — sur choix explicite de l'utilisateur entre plusieurs options proposées. Diff purement formel (retours à la ligne, littéraux multi-lignes, alignement de commentaires), relu en entier — 1143+107 lignes, contre 1504+409 sans le réglage `quote-style`. 43 tests toujours au vert, `ruff check` toujours à 0 signalement — voir `docs/sessions/session-19-2026-09-16b.md` |

| 22 | MSLogonII (security type 113) : échange Diffie-Hellman 64 bits + chiffrement DES-CBC des champs nom d'utilisateur/mot de passe (`_pack_mslogonii_field()`, ajouté à côté de `pack_ard()`) | **2026-09-16** — sur confirmation explicite de l'utilisateur d'investir dans le backlog « valeur incertaine ». Format vérifié contre `rfbproto.rst` seul (aucun serveur de référence installable dans ce sandbox Linux — extension UltraVNC/Windows). Vérifié par un round-trip auto-cohérent fabriqué à la main (un « serveur » recalcule le secret Diffie-Hellman et déchiffre les deux champs) — ne prouve pas l'interopérabilité avec un vrai serveur MSLogonII, seulement la cohérence interne de l'implémentation avec sa propre lecture de la spec. Deux ambiguïtés de spec tranchées et documentées en commentaire (voir `docs/sessions/session-20-2026-09-16.md`). Classé en dernier de l'ordre de préférence de négociation. Pas de test unitaire committé (même choix que VeNCrypt/RSA-AES/Apple ARD) |

| 23 | SASL (security type 20), mécanismes `PLAIN` (RFC 4616) et `ANONYMOUS` (RFC 4505) seulement (`_sasl_negotiate()`, ajoutée à côté de `_pack_mslogonii_field()`) | **2026-09-16** — ré-examen de SASL après son rejet en bloc au patch 22/session 20 : le tramage RFB (`rfbproto.rst`, section « SASL » — mechlist/client-start/server-start) est en réalité documenté indépendamment de toute bibliothèque SASL, et `PLAIN`/`ANONYMOUS` sont chacun définis par leur propre RFC autonome sans échange défi-réponse — seuls ces deux mécanismes sont implémentés, tout autre (`DIGEST-MD5`/`GSSAPI`/etc., qui nécessiteraient réellement `cyrus-sasl`) est refusé explicitement via `ValueError` au moment de la négociation plutôt que deviné. `PLAIN` choisi en priorité si offert avec des identifiants ; `ANONYMOUS` sinon ; aucun mécanisme pris en charge → échec avant tout envoi réseau. Ni l'un ni l'autre ne négocient de couche de confidentialité (SSF nul par construction, RFC 4422), donc pas de `sasl_encode`/`sasl_decode` à implémenter pour ces deux mécanismes précisément. Classé en toute dernière position de l'ordre de préférence (après MSLogonII) : `PLAIN` envoie le mot de passe en clair sur le fil, sans aucune couche de chiffrement propre à ce type de sécurité. **Aucun serveur SASL de référence installable dans ce sandbox** — vérifié par un round-trip auto-cohérent fabriqué à la main (7 tests unitaires ajoutés, `SaslNegotiateTests`), même limite que MSLogonII. Voir `docs/sessions/session-21-2026-09-16c.md` pour le détail complet |

## Comment réappliquer si tu pars d'une version plus récente de ton fork


Les 5 patches de la session précédente sont documentés en détail (format
"avant/après" avec numéros de ligne) dans les fichiers séparés déjà
livrés à l'époque : `shared_flag_patch.py`, `auth_edge_cases_patch.py`,
`robustness_caps_patch.py`, `rre_corre_patch.py`,
`client_requests_patch.py`. Utilise-les comme référence si `asyncvnc2.py`
dans ce zip a divergé de ton fork actuel.

Les patches xvp, `listen()` et repli de version de cette session n'ont
pas (encore) de fichier avant/après séparé — se référer directement à
`asyncvnc2.py` :

- xvp : chercher `Enc.XVP`, `UpdateType.XVP`, `process_xvp`, `send_xvp`
  pour retrouver l'ensemble des points touchés (Enc, UpdateType, deux
  champs sur `Client`, une méthode `process_xvp` + quatre méthodes
  `send_xvp*`, et les constantes
  `XVP_FAIL`/`XVP_INIT`/`XVP_SHUTDOWN`/`XVP_REBOOT`/`XVP_RESET` en haut
  du fichier).
- `listen()` : une seule fonction, juste après `connect()` en fin de
  fichier, plus l'ajout de `start_server`/`Queue`/`QueueFull` à
  l'import `asyncio` en tête de fichier. Attention si tu la réappliques
  à la main : ne PAS ajouter de `await server.wait_closed()` après
  `server.close()` (voir la section deadlock ci-dessus).
- Repli de version : dans `Client.create()`, chercher `server_major`/
  `server_minor`/`server_version` — remplace le bloc qui lisait
  jusqu'ici `intro`/écrivait `b'RFB 003.008\n'` en dur puis lisait la
  liste de types de sécurité sans condition. Touche aussi le champ
  `protocol_version` ajouté sur `Client` et le test `auth_type == 1 and
  server_version in ((3, 3), (3, 7))` juste avant la lecture de
  `auth_result`. Attention si tu réappliques à la main : ne PAS traiter
  major > 3 comme 3.3 (voir la section dédiée ci-dessus — c'est
  contre-intuitif mais délibéré).
- Extended Clipboard (patch 17) : chercher `Enc.EXTENDED_CLIPBOARD`,
  `CLIPBOARD_FORMAT_`, `CLIPBOARD_ACTION_`, `clipboard_ext_supported`,
  `process_extended_clipboard`, `send_clipboard_` pour retrouver
  l'ensemble des points touchés (un membre `Enc`, dix constantes de
  module, deux champs sur `Client`, une méthode de réception + cinq
  méthodes d'émission, et la branche `UpdateType.CLIPBOARD` de
  `Client.read()` modifiée pour lire une longueur S32 signée au lieu
  d'un U32). Attention si tu réappliques à la main : ne pas oublier le
  garde-fou `clipboard_ext_supported` sur les cinq méthodes d'émission
  (voir la section dédiée dans `features-backlog.md` pour pourquoi ce
  choix diffère de `send_xvp()`), et toujours préférer
  `send_clipboard_update()` à `send_clipboard_provide()` seul en dehors
  d'une réponse à un `Request` déjà reçu.
- VeNCrypt Plain (patch 18) : chercher `_VENCRYPT_TLS_PLAIN`,
  `_VENCRYPT_X509_PLAIN` pour retrouver les deux constantes ajoutées, le
  tuple de préférence étendu dans `_vencrypt_negotiate()` (6 sous-types
  désormais), et la nouvelle branche `elif vencrypt_subtype in
  (_VENCRYPT_TLS_PLAIN, _VENCRYPT_X509_PLAIN):` dans `Client.create()`,
  juste après la branche "Vnc" existante. Attention si tu réappliques à
  la main : ne PAS faire pointer cette branche vers le bloc VNC
  Authentication existant (`auth_type = 2`) comme le fait la branche
  "Vnc" juste au-dessus — le format Plain n'a aucun défi envoyé par le
  serveur, l'envoi du couple identifiant/mot de passe doit se faire
  directement dans cette branche, `auth_type` restant à 19.
- RSA-AES (patch 19) : chercher `_rsa_aes_negotiate`, `_EAXReader`,
  `_EAXWriter`, `_eax_seal`, `_eax_open`, `RSA_AES_` pour retrouver
  l'ensemble des points touchés (quatre constantes de module, quatre
  fonctions primitives EAX, deux classes d'enveloppe flux, une fonction
  de négociation, un nouveau champ sur `Client`, et `(5, 129)` ajouté au
  tuple de préférence dans `Client.create()`). Attention si tu
  réappliques à la main : la troncature de la clé de session dérivée du
  hachage doit être `[:key_bits // 8]`, PAS `[:16]` en dur — c'est
  exactement le bug trouvé et corrigé cette session (correct par
  accident pour AES-128/RA2, faux pour AES-256/RA2_256 qui doit
  utiliser les 32 octets complets). Vérifier aussi que `_EAXReader`/
  `_EAXWriter` remplacent bien `reader`/`writer` pour TOUTE la suite de
  `Client.create()` (SecurityResult inclus), contrairement à VeNCrypt où
  seule la poignée de main change de transport.
- SASL (patch 23) : chercher `_sasl_negotiate`, `auth_type == 20` pour
  retrouver l'ensemble des points touchés (une fonction de négociation,
  `20` ajouté au tuple de préférence de `Client.create()`, un appel sous
  `if auth_type == 20:`). Attention si tu réappliques à la main : ne PAS
  tenter d'ajouter `DIGEST-MD5`/`GSSAPI`/`CRAM-MD5` sans une vraie
  bibliothèque SASL tierce (`cyrus-sasl`) — le tramage RFB seul ne
  suffit pas à calculer leur `clientout-data`, contrairement à
  `PLAIN`/`ANONYMOUS` ; ne PAS oublier le bourrage NUL d'un octet sur
  `clientout-data` (la spec le rend obligatoire même pour un message
  vide, pour préserver la distinction NULL/chaîne-vide côté serveur).


## Ce qui n'est PAS dans ce patch (cf. ROADMAP_completeness.py, non
## inclus dans ce zip — livré séparément dans la session)

- Ancien type de sécurité « TLS » isolé (18, distinct de VeNCrypt/19,
  obsolète) — non implémenté, supplanté par VeNCrypt en pratique,
  valeur ajoutée incertaine par rapport au type 19 (voir `features.md`).
- SASL (`PLAIN`/`ANONYMOUS`) et MSLogonII (113) implémentés depuis les
  patches 22/23 ci-dessus. MSLogon (l'original, distinct de MSLogonII)
  et Ultra (LZO) restent non commencés — extensions spécifiques à
  UltraVNC (Windows), sans section dédiée dans `rfbproto.rst` (MSLogon)
  ou sans dépendance de décompression LZO dans la bibliothèque standard
  (Ultra), et sans serveur de référence installable dans ce sandbox
  Linux dans les deux cas. Les mécanismes SASL au-delà de `PLAIN`/
  `ANONYMOUS` (`DIGEST-MD5`, `GSSAPI`, etc.) restent également non
  implémentés — nécessiteraient une vraie bibliothèque SASL tierce. SASL
  est en outre confirmé absent des types de sécurité qu'offre réellement
  `Xtigervnc` 1.13.1, donc non testable contre ce serveur non plus.
- RSA-AES, variantes `RA2ne`/`RA2ne_256`/`RA2r`/`RA2r_256` — `RA2`/
  `RA2_256` implémentés depuis le patch 19 ci-dessus ; les variantes
  "ne" (ne chiffrent que la poignée de main, pas la session — annule
  l'intérêt même de RA2) et "r" (seconde ronde de dérivation de clé,
  propriété marginale pour ce fork) restent délibérément non
  implémentées.
- Extended Clipboard, formats rtf/html/dib/files — le format *text*
  (UTF-8) est implémenté depuis le patch 17 ci-dessus ; les autres
  formats sont correctement sautés dans le flux mais jamais décodés
  (valeur réelle incertaine, non demandé).

xvp, la reconnexion inversée (`listen()`), le repli de version de
protocole, VeNCrypt (sous-types TLSNone/TLSVnc, X509None/X509Vnc puis
TLSPlain/X509Plain — type de sécurité 19, pas le type 18 isolé
ci-dessus) et RSA-AES (RA2/RA2_256, type de sécurité 5/129) ont été
ajoutés dans des sessions ultérieures (voir tableau des patches
ci-dessus et `features.md` §§ « Client requests » / « Transport » /
« Authentification »)
— ils ne font donc plus partie de cette liste.
