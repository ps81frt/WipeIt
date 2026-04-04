# BUG FIX PROTOCOL

## Problème

- Application : `WipeIt` (WPF .NET 8)
- Module natif : `WipeCore.dll`
- Fichier concerné : `WipeCore/src/FileWiper.cpp`
- Crash initial : `0xC0000005` (violation d'accès)
- Point de plantage : `LookupAccountNameW(nullptr, name, nullptr, &uLen, domain, &dLen, &use)`

## Cause racine

La fonction `FileWiper::TakeOwnership` appelait `LookupAccountNameW` avec des tampons invalides :

- `pSid` non alloué
- `domain` initialisé mais pas correctement utilisé
- `uLen` / `dLen` passés sans passer par un premier appel pour récupérer les tailles nécessaires

Cela entraînait un accès mémoire invalide par la fonction native Windows, et donc un crash.

## Correction appliquée

### Fichier modifié

- `WipeCore/src/FileWiper.cpp`

### Changements

- Récupération du nom d'utilisateur via `GetUserNameW`
- Premier appel à `LookupAccountNameW(nullptr, name, nullptr, &uLen, nullptr, &dLen, &use)` pour obtenir les tailles requises
- Allocation de mémoire avec `LocalAlloc` pour :
  - `pSid`
  - `domain`
- Second appel à `LookupAccountNameW` avec buffers valides
- Libération correcte de `domain` et `pSid` via `LocalFree`

### Résultat attendu

- `TakeOwnership` ne plante plus dans `LookupAccountNameW`
- Le crash `0xC0000005` dans `filewiper.cpp` devrait être résolu

## Vérification

### Étapes de reproduction

1. Ouvrir la solution `WipeIt.sln` dans Visual Studio
2. Sélectionner la configuration `Debug` et la plateforme `x64`
3. Exécuter `WipeIt` en tant qu'administrateur
4. Lancer une opération de suppression ou de nettoyage nécessitant le `FileWiper`

### Résultats attendus

- Aucune violation d'accès native
- L'application démarre et fonctionne sans crash natif
- `WipeCore.dll` est chargé correctement et les symboles du projet sont disponibles

## Notes supplémentaires

- Le journal du debugger peut afficher des exceptions .NET non fatales (`ArgumentException`) ou des messages Windows système sans que le bug lui-même ne soit de nouveau présent.
- Il est recommandé de reconstruire proprement après la modification :
  - `WipeCore` en `Debug|x64`
  - `WipeIt` en `Debug|x64`

## Historique des modifications

- `WipeCore/src/FileWiper.cpp` : correction de l'appel `LookupAccountNameW` et gestion mémoire propre
