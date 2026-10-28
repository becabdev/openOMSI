# DLSS 5 : prochaines actions

État au 28 octobre 2026 : **LEVEL 2.5, aucune inférence NVIDIA démontrée**.
Capture GPU, historique, transport, replay, composition/fallback et tests fonctionnent.
Les captures 256², 512² et 1280×720 sont préparées ; ces tailles restent des candidats NR.
Le build natif MSVC/NGX, le support de la RTX 4050 et la vraie sortie NR restent à tester.

## Ce que vous devez fournir

1. **Windows x64 natif**, PowerShell 7 (`pwsh`), pilote NVIDIA, Visual Studio Build Tools 2022
   (« Développement Desktop en C++ » et Windows SDK), CMake et runtime Visual C++ x64.
   Rust MSVC est nécessaire pour le jeu, pas pour le replay autonome C++.
2. Votre SDK officiel Streamline 2.14.16, avec `external/ngx-sdk` et
   `nvsdk_ngx_d.lib`, sous les conditions de sa licence.
3. Un **`nvngx_dlssnr.dll` authentique, avec des droits couvrant ce test hors mod**,
   dans un dossier privé, par exemple `$env:LOCALAPPDATA/openOMSI-NR/runtime`.
   Notre profil NGX expérimental admet FileVersion 310.8.0.* ou 310.9.16.* ; version
   et signature ne garantissent ni les droits ni la compatibilité des paramètres.
   **`sl.dlss_nr.dll` n'est pas requis pour NGX.** Il ne faut pas renommer SR/RR en NR.

Le [Hub RTX Encore](https://github.com/SilyNoMeta/rtx-encore-hub/blob/b384f0342f71281d7c86cb0b0b7e7bc3e618a500/docs/USER-GUIDE.fr.md)
annonce NR 310.9.16 depuis son stockage communautaire, vérifié par signature et build.
Son code d'acquisition est privé ; les publications inspectées proposent l'application,
pas une DLL NR autonome. L'édition portable du Hub n'est pas un runtime autonome.
URL exacte du stockage, algorithme effectif de validation et autorisation de réutilisation
hors mod ne sont pas établis par ces sources. **Aucun runtime n'a été téléchargé.**
Demandez à NVIDIA/à votre fournisseur autorisé l'accès NR pour votre moteur et les
conditions correspondantes ; point d'entrée : [NVIDIA Developer DLSS](https://developer.nvidia.com/rtx/dlss).
Ce portail ne garantit pas que son SDK public contient NR. Si vous disposez déjà
d'une copie obtenue légalement, vérifiez que ses conditions couvrent cet usage.

## Compiler et tester sous Windows

Décompressez le paquet de validation dans un dossier utilisateur. Il contient
notre hôte de diagnostic **MinGW, sans NGX lié**, les sources C++ nécessaires au
build natif, les scripts et douze vrais paquets GPU. Aucun DLL NVIDIA inclus.
Depuis la racine du paquet ou du dépôt :

```powershell
$sdk = 'C:/SDK/streamline-sdk-v2.14.16' # votre SDK existant
$runtime = Join-Path $env:LOCALAPPDATA 'openOMSI-NR/runtime' # votre dossier privé existant
$project = [Guid]::NewGuid().ToString() # identifiant propre à votre projet, à conserver

./scripts/neural/build-windows.ps1 -StreamlineSdk $sdk -AcceptSdkLicense
$neuralHost = (Resolve-Path './target/neural-windows/Release/openomsi-neural-host.exe').Path
& $neuralHost --build-info # attendu : windows_x64=true, msvc>0, ngx_compiled=true

# Intégrité uniquement : aucun runtime NR chargé, aucun GPU certifié
./scripts/neural/diagnose-windows.ps1 -NeuralHost $neuralHost -Runtime $runtime -Backend ngx -VerifyOnly

# Requête réelle de support et initialisation ; pas encore d'image
./scripts/neural/diagnose-windows.ps1 -NeuralHost $neuralHost -Runtime $runtime -Backend ngx -Initialize -Project $project -AcceptRuntimeLicenseAndExperimentalLdr -AcceptExperimentalNgx18

# Dans le paquet : premier replay, commencer à 512²
./scripts/neural/validate-bundle.ps1 -NeuralHost $neuralHost -Runtime $runtime -Resolution 512 -Replay -Project $project -AcceptRuntimeLicenseAndExperimentalLdr -AcceptExperimentalNgx18
```

Les acquiescements attestent votre acceptation des conditions et du profil expérimental ;
ils ne confèrent pas de droits supplémentaires. Ne désactivez pas SmartScreen,
Authenticode ou les contrôles du pilote, et ne copiez rien dans les dossiers système.
Une révocation non vérifiable dans le cache de certificats Windows reste un échec.

Depuis le dépôt, sélectionner directement une capture :

```powershell
./scripts/neural/test-runtime.ps1 -NeuralHost $neuralHost -Runtime $runtime -Capture './target/dlss5-final/reference-720p' -Project $project -Backend ngx -AcceptRuntimeLicenseAndExperimentalLdr -AcceptExperimentalNgx18
# Autres candidats : target/dlss5-runtime/reference-sequence et target/dlss5-final/reference-512
```

## Lire le résultat et choisir l'action suivante

Les logs sont sous `target/neural-logs/`. `output/report.json` et les JSON par frame
conservent GPU/LUID/pilote, SHA256/version du runtime, codes réels des appels,
masque de support et création/réutilisation de feature. Une opération absente
de `api_calls` n'a pas été exécutée. Un code positif de requête NGX ne remplace pas
un masque de support zéro. `--verify-runtime` ne vérifie pas les droits d'usage.

| Échec | Preuve à lire et prochaine action |
|---|---|
| A : runtime absent | `MissingRuntime`, fichier nommé. Fournir le vrai NR autorisé dans `-Runtime`. Le SDK seul ne suffit pas. |
| B : fichier invalide | PE AMD64/DLL, `WinVerifyTrust` et son code, FileVersion, signature NVIDIA ou `MissingDependency`. Obtenir une copie intacte/compatible ou installer ses prérequis officiels ; ne pas modifier la signature. |
| C : chargement impossible | `LoadLibraryExW`/`GetProcAddress`, `GetLastError` exact. 126 peut indiquer une dépendance, 193 un format incompatible ; relire l'inventaire d'imports. Les dépendances dynamiques restent décidées par le loader/API. |
| D : feature/support indisponible | `api_calls`, `support_mask`, GPU/pilote. Conserver le refus. Une requête inconnue/NotImplemented peut venir du dispatcher, sans prouver un refus matériel. Aucun spoof ni fallback privé. |
| E : création/évaluation refusée | Appel exact et code, résolution, formats et profil. Essayer séparément 512²/720p, puis demander le contrat d'entrée au fournisseur ; aucune de ces tailles n'est certifiée ici. |
| F : appel positif, sortie non valide | Étapes SUBMITTED/COMPLETED/READ/VALIDATED, erreur de fence/device, sortie inchangée ou spatialement constante. Ne pas déclarer une image NR valide ; conserver les logs pour vérifier états et formats. |
| G : sortie valide, rien dans le jeu | Vérifier session/frame/génération, pixels valides, resets et compteurs d'évaluation/composition. Le HUD reste séparé. Une pause >0,5 s peut invalider l'historique. |

Pour une preuve NR : dix étapes vraies, véritable appel d'évaluation tracé,
completion GPU, `.rgba` et `.bmp` récupérés avec SHA256. Comparer visuellement avec
l'entrée ; le contrôle de cohérence ne qualifie pas encore la qualité du modèle.
Le rapport de l'hôte garde `presented_in_game=false` : il ne voit pas l'affichage du jeu.

Après ce succès seulement, dans le **dépôt complet**, construire le jeu avec
`cargo build -p omsi-app --features neural-external --locked`, puis :

```powershell
./scripts/neural/run-openomsi.ps1 -GameExe './target/debug/openomsi.exe' -Mode Neural -NeuralHost $neuralHost -Runtime $runtime -Project $project -Backend ngx -AcceptRuntimeLicenseAndExperimentalLdr -AcceptExperimentalNgx18
```

LEVEL 4 demande la vraie sortie visible dans la fenêtre, les compteurs de composition
GPU et une comparaison ON/OFF. Quitter puis lancer normalement remet OFF.
L'hôte seul ne prouve pas LEVEL 4. Notre session Linux ne l'a pas testé.

## MSVC par GitHub Actions et GPU de comparaison

Le workflow [neural-windows.yml](../.github/workflows/neural-windows.yml) est prêt,
**non déclenché ici**. Publier ces modifications et rendre le workflow disponible
depuis la branche par défaut, puis Actions → **Neural host MSVC validation** →
Run workflow, choisir la branche et accepter la licence SDK après lecture.
Il vérifie le ZIP officiel NVIDIA par SHA256, compile MSVC avec NGX, exécute les tests
CPU/CLI et les checks Rust Windows. Artefact : `neural-msvc-build-evidence`, logs uniquement.
Aucun GPU NR ni exécutable lié au SDK propriétaire n'est publié par ce workflow.
Le test CI utilise des paquets **synthétiques CPU explicitement identifiés**, jamais
comme preuve de rendu ou d'inférence ; le paquet utilisateur contient les vraies captures.

La RTX 4050 n'a pas été interrogée par NR. NVIDIA indique RTX 50 comme support officiel
et une extension RTX 40 envisagée, ce qui ne prouve pas ce backend privé.
Si le vrai runtime refuse la carte, reprendre **les mêmes commandes et captures**
sur une RTX 50 Windows, par exemple RTX 5080/5090, avec runtime/pilote autorisés,
sans présumer que le dispatcher Reserved18 sera accepté.
Source : [FAQ NVIDIA](https://www.nvidia.com/en-us/geforce/forums/game-%20ready-drivers/13/583738/dlss-5-faq/).
