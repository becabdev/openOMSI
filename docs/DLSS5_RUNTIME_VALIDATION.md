# NR : capture réelle, hôte à deux backends et validation Windows

**Mise à jour du 28 octobre : [prochaines actions et paquet Windows](DLSS5_NEXT_ACTIONS.md).**
Le diagnostic intègre maintenant les imports PE/delay imports, `--verify-runtime`
sans chargement, les codes API et l'identité GPU par replay. Le workflow MSVC est
préparé mais non exécuté ; aucun niveau 3/4 n'est acquis.

Développement du 27 octobre 2026, suite de [LEVEL 2](DLSS5_COMMUNITY_BACKENDS.md).

**LEVEL 2.5 — RUNTIME-READY INTEGRATION au sens du chemin logiciel implémenté.**
La qualification native Windows/MSVC, le linkage NGX et l'inférence restent à effectuer.
**LEVEL 3 non atteint : aucun runtime NR NVIDIA chargé ni évalué dans cette session.**
Aucune image neuronale affichée dans openOMSI ; LEVEL 4 et LEVEL 5 non atteints.
La capture de référence est une petite scène rendue par le vrai renderer openOMSI,
pas une capture d'une partie complète avec un bus et une carte OMSI.

Le premier obstacle matériel est l'absence de `nvngx_dlssnr.dll` autorisé localement.
`sl.dlss_nr.dll` manque également. NGX supprime cette seconde dépendance, pas la première.
Le SDK local apporte les en-têtes et la bibliothèque de dispatcher NGX ; il n'apporte
pas le modèle NR. Aucun binaire communautaire ou runtime propriétaire n'a été téléchargé.

## Préservation et environnement

Base `f3a1f299709f2fcc49c57ac1209010840a8376e8`, branche `experimental/dlss5-community`.
L'état LEVEL 2 a été archivé avant modification :
`target/dlss5-runtime/level2-before-runtime.tar.gz`, 48 fichiers, manifeste de hashes,
patch Git et métadonnées ; contenu relu et vérifié.
SHA256 : `5e4955876e146404d548724d058f91e481cf46ec5360738ee357ceb67abde519`.
Les modifications utilisateur sont conservées ; aucun reset, changement de pilote ou commit imposé.

Linux, RTX 4050 Laptop, Vulkan, pilote `595.99.02`. Wine est disponible, mais son
DXGI expose ici une GTX 470 émulée et annonce D3D12 niveau 12.0 indisponible.
Cette description Wine n'est pas l'identité matérielle de la RTX 4050 ; aucun hook
d'identité GPU n'a été ajouté. Aucun environnement Windows natif n'est accessible.

## Voies comparées

| Voie | Binaires nécessaires / présents localement | GPU et OS | Implémentation et obstacle restant |
|---|---|---|---|
| A : Streamline NR 1004 | `sl.interposer.dll`, `sl.common.dll` présents dans le SDK ; `sl.dlss_nr.dll` et `nvngx_dlssnr.dll` absents | Windows x64 D3D12 ; support à interroger avec le runtime | Conservée dans le même hôte ; signatures, `slInit`, support, tags, allocation, Evaluate, fence et readback. Plugin et modèle absents ; configuration NR/LDR non qualifiée. |
| B : NGX Reserved18 | SDK NGX et `nvsdk_ngx_d.lib` présents ; `nvngx_dlssnr.dll` absent ; aucun plugin Streamline requis | Windows x64 D3D12 ; requête NGX obligatoire, RTX 4050 non qualifiée | Adaptateur optionnel implémenté. Headers publics typés, paramètres NR observés et isolés. Syntaxe vérifiée ; linkage MSVC et modèle non testés. |
| C : hôte UniversalDLSS5 | Sources publiques disponibles ; aucun exécutable tiers lancé | Route D3D12 Windows étudiée ; autres annonces non validées ici | Son ABI 8 de partage GPU diffère d'openOMSI v1. Aucun adaptateur ABI 8 ajouté : notre staging fournit déjà les entrées. Les fallbacks privés et hooks matériels amont sont exclus. |
| D : RTX Encore | Documentation publique disponible ; runtime NR à fournir ; aucun module lancé | RTX 20/30/40 annoncées expérimentalement, Vulkan Windows également annoncé | Pas d'implémentation NR dans l'arbre public inspecté. Jeu DLSS SR/DLAA et profondeur compatible requis ; openOMSI ne satisfait pas ce prérequis automatiquement. |
| E : Streamline natif dans wgpu | Même plugin et runtime manquants que A | Windows DX12 ; accès natifs, états et présentation nécessaires | Différé : changer l'interop GPU ne rendrait pas les composants NR disponibles. |
| F : replay indépendant | Même hôte et mêmes composants que A ou B | Windows D3D12, GPU accepté par le runtime | Mode complet ajouté, lit de vraies captures, appelle le même évaluateur que le jeu, exporte sortie et rapports. Exécution NR bloquée par les prérequis ci-dessus. |
| G : Vulkan/Wine/Proton | Capture Vulkan Linux fonctionnelle ; aucun runtime NR | Client Windows Vulkan possible par staging, hôte D3D12 ; Linux NR non validé | CLI Windows testée sous Wine ; D3D12 indisponible dans ce préfixe. Aucun résultat NR Linux ou Proton déduit de ces tests. |

`Auto` sélectionne Streamline si ses quatre DLL existent ; sinon NGX s'il est compilé
et si son runtime existe. Un build sans NGX ne l'annonce pas comme backend compilé.
`--backend ngx` est indépendant de l'absence du plugin Streamline. Après échec
d'initialisation, relancer explicitement l'autre backend dans un **nouveau processus** :
pas de réinitialisation hasardeuse d'un dispatcher partiellement initialisé.

## Recherche ciblée et contrats

Les révisions, hashes des textes lus, inventaire du SDK et métadonnées des archives
sont dans [DLSS5_RUNTIME_SOURCES.json](DLSS5_RUNTIME_SOURCES.json).

### RTX Encore

Révision `c74655f71b930c3562267fddd9aca9d530cc3d08`, arbre public de 44 entrées :
documentation, aucun code d'initialisation/évaluation NR inspectable, aucune licence
racine donnant le droit de reprendre une implémentation absente. Les notices de
composants tiers ne changent pas cette observation.

La [documentation NR](https://github.com/SilyNoMeta/rtx-encore/blob/c74655f71b930c3562267fddd9aca9d530cc3d08/docs/NEURAL-RENDERING.md)
décrit un traitement après DLSS SR/DLAA, ou après le résultat débruité de RR,
avec une profondeur compatible. Elle cite 310.8.0 et 310.9.16. L'[installation](https://github.com/SilyNoMeta/rtx-encore/blob/c74655f71b930c3562267fddd9aca9d530cc3d08/docs/INSTALLATION.md)
ne constitue donc pas une méthode directement applicable à openOMSI.

| Points examinés | Ce qui est accessible et limite de la preuve |
|---|---|
| Init, détection GPU, chargement, structures et synchronisation | Absents du code public inspecté. Impossible de confirmer appels, ABI, barrières ou propriété des ressources à partir de la seule documentation. |
| Couleur, guides, ordre des passes | Documentation : NR après SR/DLAA/RR, HDR conservant son format numérique, profondeur linéaire après RR limitée à une bêta D3D12. Aucun contrat RGBA8/sRGB utilisable garanti pour notre hôte. |
| Taille, modèles, récupération, succès, artefacts et coût | Réduction de résolution, plusieurs passes, choix de modèle, captures et mesures décrits. Ce sont des fonctionnalités déclarées du mod ; pas une preuve de leur algorithme, ni des mesures RTX 4050 openOMSI. |

Conclusion : aucune logique NR Encore copiée ; pas de SR ajouté uniquement pour
débloquer un mod dont l'implémentation et le runtime restent absents. La route
explicite NGX d'UniversalDLSS5 est plus directement testable avec nos entrées.
Aucun minimum VRAM ni temps GPU RTX 4050 vérifié n'est disponible pour cette intégration.

### UniversalDLSS5 et NGX

Révision inchangée `369872ffb80a2bd8a4f8e3469f5b40e515fcc0f6`.
Fichiers confirmés : `src/neural/ngx_nr.cpp`, `src/neural/streamline_nr.cpp`,
`src/host/`, `include/udlss/`, notamment
`include/udlss/external_host_protocol.hpp`. Licence Apache-2.0 du projet,
licence séparée du runtime. Le [backend amont](https://github.com/MotionflowOffical/UniversalDLSS5/blob/369872ffb80a2bd8a4f8e3469f5b40e515fcc0f6/src/neural/ngx_nr.cpp)
possède une route dispatcher public et des fallbacks privés supplémentaires.
Seule la première a servi à définir notre adaptateur.

Notre [adaptateur](../tools/neural-host/ngx.cpp) emploie les déclarations exactes
du SDK local : ProjectID/custom engine propre à l'opérateur, FeatureDiscoveryInfo,
GetFeatureRequirements, Init_with_ProjectID, Allocate/DestroyParameters,
CreateFeature, EvaluateFeature, ReleaseFeature et Shutdown1.
Les setters C du SDK évitent des appels directs à une vtable C++ reconstruite.
La bibliothèque officielle est liée **uniquement avec MSVC x64** ; MinGW ne
constitue pas une validation de son ABI binaire. Aucun SDK n'est vendu dans le dépôt.

`Reserved18` est un enum réservé public, **le contrat `DLSSNR.*` est privé/observé**.
Les types UI/I/F/resource/ULL, callback de ratio et ressources sont isolés dans
un seul fichier. Profil limité aux FileVersion 310.8.0.* et 310.9.16.* signées NVIDIA,
avec acquiescement explicite. Ce filtre ne garantit pas la compatibilité.
Le preset 0 conserve le choix du DLL ; aucune correspondance A/B officielle inventée.
La requête de support doit réussir avec masque zéro ; `CheckNotPresent`,
`NotImplemented`, refus de GPU/pilote et résultats inconnus arrêtent la route.

Exclus : init snippet privée, app ID emprunté, `_Init_Ext` devinée, proxy
`nvngx.dll`, NVAPI hooks, substitution Ada/Blackwell et modification du runtime.
Le dispatcher peut encore refuser Feature18 : c'est un blocage explicite, pas une
raison d'activer ces fallbacks. Les droits d'utilisation viennent du SDK/runtime
autorisé de l'opérateur ; le code communautaire ne concède aucun droit sur NVIDIA.
Le SDK impose ses propres conditions, dont les conditions de distribution ;
la coexistence avec les licences du projet doit être résolue avant distribution d'un
exécutable lié au SDK propriétaire. Ce travail ne redistribue aucune bibliothèque NGX.

### RHI et AIO

RHI : `c1076d38520dce8d69e006194aad86fece7c26d0`, licence GPL-3.0 du projet.
Le [manifeste](https://github.com/RankFTW/RHI/blob/c1076d38520dce8d69e006194aad86fece7c26d0/dlss_manifest.json)
annonce les associations suivantes ; architecture, modèles et intégrité des DLL
ne sont pas certifiés par ces noms.

| Distribution annoncée | Association annoncée / ce qui n'est pas vérifié |
|---|---|
| NR 310.9.16 | RTX 20/30/40/50 selon manifeste ; modèle/pilote réel non vérifié |
| NR 310.8.0 | RTX 50 selon manifeste ; compatibilité RTX 4050 non établie |
| NR 310.8.Lecram, libellé 310.8.3 | RTX 50 annoncé ; FileVersion/signature inconnues |
| NR 310.8.SF-v2, libellé 310.8.2 | RTX 20/30/40/50 annoncé ; variante communautaire non qualifiée |
| NR 310.8.SF et 310.8.0-RTX40 | Archives présentes dans les métadonnées de releases ; nom insuffisant pour établir origine/architecture |
| Streamline 2.14.16.0-v2 | Archive de 4 116 590 octets annoncée ; contenu exact/plugin NR non extrait |
| RenoDX DLSS5 10.8.0 | Archive de 2 635 198 octets annoncée ; addon/contrats réels non inspectés dans cette archive |

Les [releases](https://github.com/RankFTW/rhi-repo/releases) consultées fournissent
des digests SHA256 d'**archives**, conservés dans le JSON. Aucun n'a été présenté
comme un hash de DLL téléchargée et vérifiée. Le service public CostScaler de RHI
renomme le vrai runtime et installe un proxy/addon : il ne fournit pas une nouvelle
ABI NR autonome garantie. Cette substitution n'est pas reprise.
Les formats, modèles exacts et possibilités hors jeu DLSS ne peuvent être déduits
de ces seules distributions. L'hôte refuse une signature non NVIDIA et un profil
de version NGX inconnu, indépendamment des libellés de téléchargement.

AIO : révision inchangée `c2803c9c8e92df11cc749f4971312ae38eff6179`, wrapper MIT.
Les distinctions ReShade/RenoDX/Streamline/NGX et les limites de redistribution
de la [phase précédente](DLSS5_COMMUNITY_BACKENDS.md) restent applicables.
Aucune nouvelle archive AIO exécutée. Aucun de ces distributeurs ne prouve une
exécution NR Linux native ni l'acceptation de la RTX 4050 par notre dispatcher.

## Capture réellement produite

[neural-capture.rs](../crates/omsi-render/examples/neural-capture.rs) utilise
`Renderer::render` et le même staging que le jeu. `render_to_image` reste une API
de clichés indépendants qui coupe l'historique ; elle n'est pas utilisée pour la séquence.

```bash
cargo run -p omsi-render --example neural-capture --features neural-external --locked --offline -- target/new-reference-sequence
```

Le dossier doit être neuf, avec parent existant. Le résultat produit ici est
`target/dlss5-runtime/reference-sequence/`, également exporté dans
`target/dlss5-runtime/reference-sequence.zip`.
SHA256 ZIP : `62fe0f2fda1c43232b4ad00edd54086323224620880118670d7abc45ee1aeb7a`.
Chaque paquet 256×256 mesure 1 114 496 octets : header 384 + plans 17N.
Cette résolution légère n'est pas une taille minimale NR validée par NVIDIA.

| Frame | Scénario réel GPU | Flags | Pixels valides | Pixels avec mouvement > 0,001 px |
|---|---|---:|---:|---:|
| 1 | Premier historique/reset | 7 | 0 | 0 |
| 2 | Statique | 5 | 31 684 | 0 |
| 3 | Caméra déplacée et tournée | 5 | 31 616 | 31 616 |
| 4 | Objet déplacé | 5 | 31 436 | 31 436 |

Profondeur min 0, max environ 0,01997, cohérente avec reverse Z, plan à 5 m et near 0,1 m.
`verified-inputs.json` contient les hashes des paquets ; les PNG `*.input.png`
visualisent uniquement la **couleur d'entrée**, sans simuler un résultat NR.
Une séquence interrompue reste inspectable via les paquets complets ; `capture.json`
n'est écrit qu'à la fin d'une capture complète.

Pour capturer une vraie partie, build avec `omsi-app/neural-external`, lancer avec
`OPENOMSI_NEURAL_CAPTURE_DIR=<dossier-neuf>` et `OPENOMSI_NEURAL_CAPTURE_FRAMES=4`.
Capture et session d'inférence sont exclusives ; la capture a priorité dans l'environnement.
L'API `Renderer::request_neural_capture(path, count)` accepte 1..16 frames, sous Linux
et Windows. Aucun compteur d'évaluation/composition n'est incrémenté par une capture.
Desktop unique, rendu à 100 %, debug temporel Off ; miroirs, triple et XR restent exclus.

## Formats et exécution

| Donnée openOMSI v1 | Traitement et limites |
|---|---|
| Couleur RGBA8 sRGB encodée LDR après tone mapping, avant HUD | Permutation BGRA→RGBA si nécessaire ; textures D3D12 UNORM contenant les mêmes bytes. Aucun gamma/exposure supplémentaire. Pas de validation propriétaire que ce profil est accepté par NR. |
| Motion RG32F | Précédent−courant en pixels, X droite/Y bas, mouvement caméra inclus. Streamline : échelles 1/w et 1/h ; NGX observé : 1 et 1. Pas de jitter. |
| Depth R32F | Reverse Z [0,1], bit depthInverted. Pas de substitution arbitraire par profondeur linéaire. |
| Validity R8 | Masque moteur distinct ; jamais envoyé comme ControlMask NVIDIA. Le jeu restitue l'original sur les pixels invalides, conserve l'alpha et dessine le HUD après. |
| Matrices et métadonnées | Quatre matrices row-major pour vecteurs ligne, base caméra, near/far/FOV/aspect, génération/session/frame/reset. Le backend NGX observé consomme guides et reset ; aucune matrice privée inventée. |

Le replay contrôle bornes, taille exacte, fin de fichier, flags, guides finis,
profondeur, paramètres caméra et inverse de projection. Une séquence mélangée ou
avec trou est refusée avant chargement. Changement de taille/génération, reset
explicite ou interruption d'identité réinitialisent l'histoire du modèle.
Le moteur conserve aussi son cut après une pause de rendu de plus de 0,5 s ; une
inférence très lente peut donc laisser tous les pixels invalides et conserver l'original.

Chaque backend partage upload, textures, DIRECT queue, fence et readback de l'hôte
existant. NGX utilise SRV NON_PIXEL_SHADER_RESOURCE et UAV output, puis barrières
vers COMMON ; Streamline conserve son chemin de tags et housekeeping par Present.
Une sentinelle constante initialise la sortie à chaque frame : un appel qui n'écrit
rien ne peut certifier une transformation en lisant une allocation indéfinie.
Le timeout GPU de 10 s termine l'hôte isolé plutôt que décharger un DLL utilisé
par des commandes encore en vol. Le jeu garde son fallback et son délai borné.

Le [rapport JSON](../tools/neural-host/evidence.hpp) distingue exactement :
RUNTIME FOUND, VERIFIED, LOADED, GPU SUPPORTED, FEATURE INITIALIZED,
RESOURCES ALLOCATED, EVALUATION SUBMITTED, GPU COMPLETED, OUTPUT READ, OUTPUT VALIDATED.
Les champs sont initialement faux. Chaque frame réinitialise ses preuves d'exécution.
`core_initialized` ne signifie pas que la feature a été créée ou évaluée.
Le dernier code API reste séparé du résultat de transport normalisé.

Les SHA256 correspondent au runtime vérifié **et verrouillé**, aux bytes d'entrée
effectivement évalués et à la sortie lue. Origine des modules NGX contrôlée ; aucune
fonction privée reconstruite appelée. Sortie inchangée ou RGB constant = résultat
inconclusif et refus d'un succès complet, même avec un code API positif.
Les comptages de pixels modifiés ne prouvent pas la qualité visuelle. Une première
frame sans pixel moteur valide peut démontrer une transformation, mais n'active
pas la composition du jeu. Les sorties acceptées sont sauvegardées brutes `.rgba`,
en vue `.bmp` et avec un JSON par frame. Temps CPU de bout en bout uniquement,
pas un temps d'inférence GPU. Qualité et présentation en jeu restent marquées fausses.

Les DLL sont vérifiées PE x64, Authenticode NVIDIA et révocation avec le cache Windows,
hashées et gardées ouvertes sans partage écriture/suppression. Si les preuves de
révocation manquent, échec explicite ; aucun flag ne désactive la vérification.
Utiliser un dossier privé dédié au backend, contenant uniquement ses DLL autorisées :
tous les DLL de ce dossier sont vérifiés avant chargement. Aucun placement système.

## Commandes Windows prêtes à lancer

Prérequis : Windows x64 natif, pilote NVIDIA, Visual Studio C++ x64 et Windows SDK,
CMake ≥3.20, SDK Streamline 2.14.16 autorisé avec son `external/ngx-sdk` ou un SDK NGX
équivalent explicitement sélectionné. Rust et target `x86_64-pc-windows-msvc` pour le jeu.
Les scripts ne téléchargent aucun runtime et n'installent aucun pilote.
Lire et accepter les licences correspondantes avant d'employer les switches d'acquiescement.

Depuis la racine du dépôt, ajuster uniquement les chemins SDK/runtime existants :

```powershell
$sdk = 'C:/SDK/streamline-sdk-v2.14.16'
$runtime = 'C:/AuthorizedNvidiaNR' # Pour NGX : vrai nvngx_dlssnr.dll signé, pas un proxy
$project = [Guid]::NewGuid().ToString() # Identifiant de votre propre projet ; le conserver
$capture = 'C:/Captures/reference-sequence' # Décompresser notre ZIP ou exporter sa partie

./scripts/neural/build-windows.ps1 -StreamlineSdk $sdk -AcceptSdkLicense -CheckRust
$neuralHost = (Resolve-Path './target/neural-windows/Release/openomsi-neural-host.exe').Path

./scripts/neural/diagnose-windows.ps1 -NeuralHost $neuralHost -Runtime $runtime

./scripts/neural/diagnose-windows.ps1 -NeuralHost $neuralHost -Runtime $runtime -Backend ngx -Initialize -Project $project -AcceptRuntimeLicenseAndExperimentalLdr -AcceptExperimentalNgx18

./scripts/neural/test-runtime.ps1 -NeuralHost $neuralHost -Runtime $runtime -Capture $capture -Project $project -Backend ngx -AcceptRuntimeLicenseAndExperimentalLdr -AcceptExperimentalNgx18
```

`build-windows.ps1` active NGX par défaut ; `-StreamlineOnly` construit uniquement
la route historique et le diagnostic. `-NgxSdk <path>` sélectionne un SDK séparé.
`-Generator` permet une autre version de Visual Studio installée. `-Offline` impose
le cache Cargo pour le check Rust optionnel. Sans ce switch, Cargo peut récupérer
ses dépendances verrouillées, jamais des runtimes NR.

Résultat attendu du diagnostic sans initialisation : versions OS/pilote, adaptateurs,
VRAM, capacité D3D12, fichiers, FileVersion, vérifications de signatures/hashes,
backends **compilés**, pas de prétendu support NR déduit du nom RTX.
Le test avec runtime fournit `target/neural-logs/replay-*/output/report.json`,
les rapports par frame et les sorties. Succès complet = dix étapes vraies, résultat
transport zéro et image récupérée ; inspecter ensuite les images et logs NVIDIA.
Un refus de support/ABI/format est un résultat de test à conserver, pas à contourner.

La CLI directe expose `--self-test`, `--inspect-frame <file-or-directory>`,
`--diagnose`, `--validate-runtime`, `--replay-frame <file-or-directory>`, `--serve`.
Les modes runtime exigent `--runtime`, `--project`, `--output <new-directory>`
et les acquiescements ci-dessus ; `--serve` exige également `--session` privé neuf.
Une sortie existante est refusée pour préserver les preuves. Les chemins Unicode
Windows passent par `wmain`, puis UTF-8 et filesystem wide.

Pour produire les entrées d'une vraie partie sous Windows :

```powershell
cargo build -p omsi-app --features neural-external --locked
./scripts/neural/run-openomsi.ps1 -GameExe './target/debug/openomsi.exe' -Mode Capture -CaptureFrames 4
```

Le nom exact de l'exécutable doit correspondre au build habituel du projet ; fournir
son vrai chemin à `-GameExe`. Le script ne remplace pas la configuration du jeu.
Après un replay NR réussi, tester la composition :

```powershell
./scripts/neural/run-openomsi.ps1 -GameExe './target/debug/openomsi.exe' -Mode Neural -NeuralHost $neuralHost -Runtime $runtime -Project $project -Backend ngx -AcceptRuntimeLicenseAndExperimentalLdr -AcceptExperimentalNgx18
```

Le script démarre **notre** hôte dans une session à ACL privée, attend `host.ready`,
active le transport pendant le processus du jeu, garde les logs, demande l'arrêt
de l'hôte puis restaure les variables du processus parent. `-GameArguments` transmet
les arguments habituels. Pour OFF : quitter et relancer normalement, ou utiliser
`Renderer::disable_external_neural()`. Un hôte absent/refusé/en timeout laisse le rendu
original. Aucun modèle NR simulé n'existe en production.

Le HUD reste séparé. Les girouettes, IBIS, panneaux, textes et feux 3D ne sont pas
protégés par un masque NVIDIA qualifié ; la scène simple reste le premier test.
Transparences, reflets, miroirs et displays complexes demandent une qualification
après une véritable preuve d'inférence.

## Tests réalisés et limites

| Catégorie / commande | Plateforme / GPU | Résultat et preuve |
|---|---|---|
| `cargo check --workspace --locked --offline` | Linux | Réussi ; build normal conservé |
| `cargo check --workspace --features omsi-app/neural-external --locked --offline` | Linux | Réussi |
| `cargo test -p omsi-render --features neural-external --lib --offline` | Linux CPU | 176 réussis, 81 ignorés ; aucun test NR réel |
| Même test filtré `nvidia::external -- --include-ignored --test-threads=1 --nocapture` | Linux RTX 4050 Vulkan | 11 réussis ; vrai staging/capture/HUD/fallback et fixtures explicites, aucun runtime NVIDIA |
| Même test filtré `temporal_gpu -- --include-ignored --test-threads=1 --nocapture` | Linux RTX 4050 Vulkan | 3 réussis : statique/mouvements/cut/pose CPU/rebase/recyclage/resize |
| Tests ciblés `fsr1_turned_on_while_running_upscales_the_picture` et `roof_occlusion_gpu` avec `OPENOMSI_ROOF_TEST_BACKEND=vulkan` | Linux GPU, roof sur RTX 4050 Vulkan | 1 + 1 réussis ; régressions de rendu, aucune inférence NR |
| `cargo run ... --example neural-capture ...` | Linux RTX 4050 Vulkan | 4 vrais paquets, historique et mouvement mesurés, PNG d'entrée vérifié |
| Cargo renderer `--target x86_64-pc-windows-gnu --features neural-external --locked --offline` avec CC/AR MinGW locaux | Cross Linux→Windows | Réussi ; pas d'exécution Windows déduite |
| CMake portable + CTest | Linux | 2 tests réussis ; guides hors domaine/NaN et sortie inchangée/sentinelle refusés |
| `scripts/neural/test-host.py` avec hôte portable et capture | Linux | 7 tests CLI réussis, dont corruptions réelles de paquets et refus des options inconnues |
| CMake Windows MinGW | Cross Linux→Windows | Hôte PE x64 lié, sans NGX ; SDK officiel NGX réservé au build MSVC |
| `-fsyntax-only` de `ngx.cpp`, `streamline.cpp`, `main.cpp` avec `OPENOMSI_WITH_NGX=1` et headers SDK | Cross MinGW | Réussi ; ce test ne valide pas le linkage MSVC/lib NGX |
| Même `test-host.py --wine` avec notre PE | Wine / pas de D3D12 NR | 9 tests CLI réussis : hash SHA256 connu, parseur, captures, corruptions, diagnostic et runtime absent. Rapports sans aucune étape NR vraie. |
| ParseFile des 5 scripts PowerShell via parseur Microsoft System.Management.Automation 7.6.9, .NET 10 | Linux | Syntaxe valide ; commandes Windows et ACL non exécutées ici |
| MSVC natif, scripts Windows complets, signatures NVIDIA et interrogation NR | Windows requis | **Non exécutés** |
| Inférence, image réelle composée, stabilité et performance NR | Runtime/GPU acceptés requis | **Non exécutés** ; LEVEL 3–5 non acquis |

Les logs CLI sont dans `target/dlss5-runtime/portable-cli-verified/` et
`target/dlss5-runtime/wine-cli-verified/`, avec `tests.json` et paquets volontairement
corrompus identifiés comme fixtures de validation. Les warnings préexistants du
workspace ne sont pas masqués. Le problème OpenGL `Parent device is lost` précédemment
signalé n'est pas traité par ce travail.

Reproduire les tests CLI avec Python 3, uniquement contre notre exécutable construit :

```bash
python3 scripts/neural/test-host.py --host target/dlss5-community/protocol-build/openomsi-neural-host --capture target/dlss5-runtime/reference-sequence --output target/new-cli-evidence
# Sous Linux/Wine : ajouter --wine, sélectionner le PE et un WINEPREFIX privé.
```

## Fichiers et fonctions de cette phase

| Création / modification | Fonctions ou rôle |
|---|---|
| Créé : [capture.rs](../crates/omsi-render/src/nvidia/external/capture.rs) | `Capture::new/write`, dossier exclusif, paquets atomiques, séquence et manifeste |
| Modifiés : [mod.rs](../crates/omsi-render/src/nvidia/external/mod.rs), [resources.rs](../crates/omsi-render/src/nvidia/external/resources.rs) | `request_neural_capture`, environnement capture, états Capturing/CaptureComplete, compteur séparé, export avant échange ; fallback/composition existants conservés |
| Modifiés : [tests.rs](../crates/omsi-render/src/nvidia/external/tests.rs), [Cargo.toml](../crates/omsi-render/Cargo.toml) ; exemple créé | Test capture exclusive/reset/séquence, extension du test GPU avec vraie capture, cible exemple conditionnée à la feature |
| Créés : [ngx.hpp](../tools/neural-host/ngx.hpp), [ngx.cpp](../tools/neural-host/ngx.cpp) | `Ngx18` init/query/release/record, profil observé, contrôle d'origine ; aucune exigence Streamline |
| Créés : [runtime_files.hpp](../tools/neural-host/runtime_files.hpp), [evidence.hpp](../tools/neural-host/evidence.hpp) | Validation DLL extraite du backend historique, versions/hashes, `Evidence::mark/next_frame/write`, `validate_output` |
| Modifiés : [streamline.hpp](../tools/neural-host/streamline.hpp), [streamline.cpp](../tools/neural-host/streamline.cpp) | `NeuralHost` partage le staging existant, branches indépendantes, barrières NGX, sentinelle output et preuves de completion |
| Révisé : [main.cpp](../tools/neural-host/main.cpp) ; créés : [diagnostics.hpp](../tools/neural-host/diagnostics.hpp), [serve.hpp](../tools/neural-host/serve.hpp) | CLI, `read_frame/sequence`, inventaire, replay/export, serveur existant conservé, Unicode Windows |
| Modifiés : [CMakeLists.txt](../tools/neural-host/CMakeLists.txt), [protocol.hpp](../tools/neural-host/protocol.hpp), [protocol_test.cpp](../tools/neural-host/protocol_test.cpp) | Backend NGX optionnel MSVC, builds portables et x64, validation numérique et tests de cohérence |
| Créés : [scripts/neural/](../scripts/neural/) | common, build, diagnose, test-runtime, run-openomsi et tests CLI Python |
| Créés/révisés : présent rapport, sources JSON, anciens rapports et README | Bilan actuel, provenance, procédures et distinction des niveaux de preuve |

Aucune dépendance Cargo ajoutée dans cette phase. Le host utilise les bibliothèques
Windows existantes ; ajout `version`, et pour NGX MSVC la bibliothèque locale NVIDIA
et les bibliothèques système shlwapi/shell32/ole32/advapi32. Aucun header SDK copié.
Python standard pour les tests CLI. Le parseur PowerShell Microsoft est un outil de
test temporaire sous target, pas une dépendance du jeu ou de l'hôte.

La prochaine preuve utile est le rapport natif du replay avec un runtime signé et
autorisé. Si la requête NGX refuse la RTX 4050 ou ne connaît pas Feature18, conserver
ce refus : ni l'état du renderer Rust ni la présence des DLL n'établissent une
compatibilité matérielle. Seulement après une véritable sortie GPU : tester la
composition en jeu, puis envisager textures partagées, fences et optimisation.

## Validation ciblée du 28 octobre 2026

La recherche Hub est consignée dans [DLSS5_FINAL_VALIDATION_SOURCES.json](DLSS5_FINAL_VALIDATION_SOURCES.json).
Révision Hub `b384f0342f71281d7c86cb0b0b7e7bc3e618a500` ; les révisions publiques
Encore et UniversalDLSS5 restent celles de ce rapport. Les dernières métadonnées RHI
mentionnent toujours NR 310.9.16 ; elles ne valident ni les bytes ni les droits.
Le Hub publie sa documentation et ses applications, sans code d'acquisition ni URL
NR autonome vérifiable. La signature annoncée n'autorise pas à elle seule l'usage
hors mod. Aucun runtime NR ou exécutable du Hub téléchargé/exécuté.

Changements ciblés :

- `pe.hpp` et `pe_test.cpp` : lecture bornée PE32+/AMD64, imports normaux et différés,
  RVA tronquées/ambiguës refusées, noms sans chemin ; résolution sans chargement.
  Les API sets et dépendances dynamiques restent sous contrôle du loader.
- `runtime_files.hpp`, `diagnostics.hpp` : empreinte du candidat explicitement non
  vérifié, puis Authenticode/éditeur/dépendances ; erreurs Win32/HRESULT hexadécimales.
- `evidence.hpp`, `ngx.cpp`, `streamline.cpp`, `main.cpp` : GPU/LUID/VRAM/pilote,
  reçus réels NGX/Streamline par opération, masque de support, création de feature
  conservée et réutilisation explicite ; historique de traces borné par frame.
- Une image de couleur uniforme avec des canaux RGB différents n'est plus confondue
  avec une variation spatiale. En replay, une sortie effectivement lue mais refusée
  est conservée en `.unvalidated.rgba/.bmp` avec échec, jamais certifiée/composée.
- `neural-capture.rs` accepte largeur/hauteur. Les douze paquets à trois résolutions
  sont contrôlés : matrices/inverses, base caméra, aspect, guides finis/reverse Z,
  historique, mouvement statique/caméra/objet et oracle de reprojection à 0,02 px.
- `build-windows.ps1` contrôle le vrai build-info MSVC/x64/NGX et la bibliothèque SDK.
  `neural-windows.yml` utilise le SDK officiel 2.14.16 et son SHA256 de release,
  exige un accord de licence manuel, produit uniquement des journaux publics.
- `package-validation.py` fournit hôte, sources C++, scripts, configuration, captures,
  documentation et manifeste SHA256. Le paquet Linux préparé contient l'hôte MinGW
  **sans NGX lié** ; ses sources permettent le build MSVC avec le SDK de l'opérateur.

| Contrôle effectué ici | Résultat |
|---|---|
| CMake/CTest Linux | 3 réussis, dont PE et cohérence de sortie |
| CLI Linux, vraies captures 256² | 8 réussis |
| CLI Linux, fixtures CPU pour runner sans GPU | 8 réussis ; aucune preuve GPU |
| CLI du PE Windows sous Wine | 11 réussis, dont NR manquant et PE invalide sans chargement |
| Inspection par notre test PE des imports du vrai hôte Windows | Réussie, 9 imports ; aucun DLL chargé par ce test |
| Captures GPU 512² et 1280×720 | 4 + 4 vraies frames Vulkan RTX 4050, toutes relues par l'hôte |
| Adaptateur Rust, tests ignorés inclus | 11 réussis, vrai staging GPU et fixtures IPC explicites |
| NGX avec les headers officiels | Syntaxe croisée vérifiée ; 13 symboles C nécessaires présents dans l'index de la bibliothèque SDK |
| Scripts PowerShell | 6 scripts parsés sans erreur avec le parseur Microsoft 7.6.9 |
| Workflow | YAML, déclenchement manuel/licence, runner et chemins d'artefacts logs-only contrôlés ; pas de run GitHub Actions |
| MSVC natif/driver/NR | Non exécutés : environnement Windows natif et runtime autorisé non disponibles |

Preuves locales : `target/dlss5-final/capture-validation.json`,
`cli-linux/`, `cli-windows-wine/`, `ci-cpu-test/`, `ngx-sdk-symbols.json`.
La compilation croisée et la présence des symboles ne remplacent pas le linkage MSVC.
