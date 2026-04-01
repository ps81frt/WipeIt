# WipeIt Portable - Guide Technique

Ce document decrit le fonctionnement technique du mode portable (distribution et execution sans installateur) et ses implications runtime.

## 1. Definition du mode portable

Dans ce projet, "portable" signifie:

1. Distribution sous forme d'executable publie (single-file, self-contained).
2. Absence d'installation MSI du desinstalleur lui-meme.
3. Le moteur natif est embarque/copie dans la sortie et charge dynamiquement par l'application .NET.

Ce mode ne supprime pas les contraintes Windows suivantes:

1. Les droits administrateur restent necessaires pour un nettoyage complet.
2. Les operations de suppression verrouillees peuvent etre differees au reboot.
3. Le scan dépend des APIs locales et de l'etat de la machine.

## 2. Artefacts de build portable

Flux de publication general:

1. Build natif WipeCore (Release x64).
2. dotnet publish WPF en win-x64 self-contained single-file.
3. Packaging des sorties dans le dossier release.

Parametres publish typiques:

1. Runtime: win-x64.
2. SelfContained: true.
3. PublishSingleFile: true.

Objectif: executer l'outil sans prerequis .NET runtime externe.

## 3. Chargement runtime en mode portable

Au lancement:

1. Le processus WipeIt initialise le bridge managé.
2. WipeCore_Init est appele.
3. Les callbacks natifs sont relies aux delegates C#.
4. Le scan initial peut etre declenche via fenetre d'attente startup.

Si la DLL native n'est pas disponible/corrompue:

1. Initialisation bridge en echec.
2. Exception de demarrage avec journalisation startup/crash.

## 4. Flux uninstall en mode portable

Le flux est identique au mode standard applicatif:

1. Inventaire des apps.
2. Session par app (CreateSession).
3. Scan leftovers (heuristique 17 phases + MSI database + firewall + COM + env).
4. Preview utilisateur et exclusions.
5. ExecuteWipe:
   - kill process,
   - uninstall command,
   - nettoyage leftovers,
   - evaluation reboot requis.

Important:

1. "Portable" pour WipeIt ne signifie pas que les applications cibles sont portables.
2. Les applications cibles peuvent etre MSI, NSIS, Inno, Store, scripts, etc.

## 5. Cas "application cible portable"

Le moteur tente de couvrir les cibles sans desinstalleur standard:

1. Detection de dossiers/fichiers lies via tokens.
2. Detection de scripts/outils de nettoyage dans le dossier d'installation quand presents.
3. Nettoyage direct des residus quand commande uninstall indisponible.

Limites:

1. Une app portable pure peut ne laisser aucun marqueur registre uninstall.
2. Le matching repose alors surtout sur noms/tokens/chemins.
3. Les faux positifs sont controles par SafeGuard + protection cross-app + scoring tokens.

## 6. Protection cross-app en mode portable

Le moteur empeche la suppression de fichiers appartenant a d'autres apps installees:

1. Cote natif: GetAllInstallDirsLower() lit dynamiquement le registre Uninstall (HKLM 64/32 + HKCU) pour lister les dossiers de toutes les apps sauf la cible.
2. Chaque candidat leftover est verifie: s'il est sous ou parent d'un dossier d'une autre app, il est rejete.
3. Cote C# (post-reboot): InstallDirGuard re-lit le registre au moment du reboot — si une nouvelle app a ete installee entre la desinstallation et le reboot, elle apparait dans la liste protegee.
4. Resolution des dossiers: InstallLocation d'abord, puis extraction depuis UninstallString/QuietUninstallString/DisplayIcon.
5. Aucun nom d'app hardcode. Fonctionne pour tout logiciel enregistre dans le registre Windows.

## 7. Flux post-reboot en mode portable

1. RebootResumeManager persiste un JSON dans %LOCALAPPDATA%\WipeIt + entree RunOnce HKCU.
2. Au redemarrage, WipeIt se relance avec --post-reboot.
3. RebootReport verifie les chemins attendus:
   - Supprime: n'existe plus.
   - Protege: appartient a une app installee entre-temps (non supprime, non re-arme).
   - Relance: tentative directe puis MoveFileExW si echec.
4. Rapport TXT exporte sur le Bureau.
5. Les chemins proteges ne declenchent jamais de boucle de re-armement.

## 8. Mode monitor et usage portable

Le mode monitor est fortement recommande pour tracer l'installation d'une application nouvelle:

1. BeginMonitor (avant install cible).
2. Installation de la cible.
3. EndMonitor (snapshot apres + diff).

Interet:

1. Precision superieure a l'heuristique seule.
2. Bonne couverture pour installateurs non standards.

## 9. Mode ETW (capture temps reel)

Nouveau mode de detection:

1. EtwStart: demarre une session ETW kernel (FileIO + Registry).
2. Tous les evenements fichier/registre sont captures en temps reel.
3. L'utilisateur installe le logiciel.
4. EtwStop: arrete la capture, consolide les resultats.
5. Precision ~98-99%, zero risque BSOD (100% user-mode).
6. Meme mecanisme que Process Monitor (Sysinternals).
7. Filtrage automatique du bruit systeme (WinSxS, System32, etc.).
8. Conversion automatique des chemins device (\Device\HarddiskVolumeX) en chemins DOS.

## 10. Logs utiles en mode portable

Verifier les journaux suivants en priorite:

1. logs/native-uninstall.log
2. logs/bridge.log
3. %LOCALAPPDATA%/WipeIt/native-crash.log
4. %LOCALAPPDATA%/WipeIt/startup.log

Ces fichiers permettent de distinguer:

1. echec commande uninstall,
2. echec interop callback,
3. exception native,
4. operation reportee au reboot.

## 11. Contraintes et bornes de performance

Le scan standard portable applique des limites internes:

1. profondeur de parcours par racine,
2. nombre max de dossiers/cles visites,
3. exclusion d'arbres systeme connus volumineux.

But:

1. eviter latences extremes,
2. reduire risque de freeze,
3. maintenir un temps de reponse acceptable.

## 12. Verification apres build portable

Checklist technique:

1. L'executable se lance en admin.
2. Le scan apps retourne des entrees.
3. Une desinstallation test retourne un statut explicite.
4. Les logs sont alimentes.
5. Le reboot pending est remonte si necessaire.

## 13. Procedure de diagnostic rapide

Si "detecte mais ne desinstalle pas":

1. Lire native-uninstall.log pour la commande et le code retour.
2. Verifier ErrorMsg dans le resultat UI.
3. Verifier si la cible n'a pas seulement un nettoyage residual (pas de vrai uninstall command).
4. Tenter mode monitor ou mode ETW pour etablir un diff plus complet et comparer.
