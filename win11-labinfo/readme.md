# Modification du ISO de Windows 11 pour LabInfo

Dans le cours de Systèmes d'exploitation (420-1X6-EM), les étudiant(e)s doivent installer Windows 11 sur une machine virtuelle dans l'environnement Labinfo. Or, Windows 11 exige la présence de la puce TPMv2 et active de manière automatique le chiffrement BitLocker dès son installation. Cela nuit considérablement au mécanisme de déduplication qui optimise le stockage grâce à la répétition de blocs de données identique entre les disques durs virtuels. Par ailleurs, Windows 11 force l'exécution des mises à jour automatiques, ce qui peut engendrer des lenteurs pendant et à tout moment après l'installation.

La solution proposée répond aux exigences suivantes:
- Permettre l'installation de Windows 11 sans vTPM
- Désactiver le chiffrement automatique Bitlocker au moment de l'installation
- Désactiver les mises à jour automatiques pendant et après l'installation, sans empêcher leur installation manuelle
- Préserver l'expérience utilisateur de l'installation de Windows 11 au moyen d'un médium d'installation officiel

Un fichier ISO modifié a été déposé dans la bibliothèque de contenu: `Logiciels\MS\Windows 11\Windows11_24H2_FR_NoTPM_NoAU.iso`


## Procédure de modification

Voici comment procéder pour modifier un fichier ISO de cette manière. Il se peut toutefois que cette méthode ne fonctionne plus dans une version de Windows 11 ultérieure à 24H2.


### Étape 1: Installation des outils

Téléchargez et installez la plus récente version de Windows ADK.

Lien: https://learn.microsoft.com/en-us/windows-hardware/get-started/adk-install

Vous devez installer les deux paquets, dans cet ordre:
1. Windows ADK
2. Windows PE add-on for the Windows ADK

Vous pouvez laisser toutes les options par défaut.

Il est impératif que la version de l'ADK installée corresponde à la version de Windows que vous tentez de modifier (par exemple, 24H2).


### Étape 2: Modification du média d'installation

Vous devez ensuite télécharger le fichier ISO. Téléchargez celui "Consumer" en français à partir du portail Azure pour l'éducation (ce média contient plusieurs éditions dont Education). Puis extrayez son contenu sur un répertoire temporaire (par exemple, `C:\Temp\Win11\ISO`). Vous pouvez utiliser 7Zip pour extraire le ISO, ou le monter sous Windows, peu importe.

À la racine du répertoire où le ISO a été extrait (là où se trouve `setup.exe`), créez un fichier nommé `autounattend.xml`. Collez-y le texte suivant:

```xml
<?xml version="1.0" encoding="utf-8"?>
<unattend xmlns="urn:schemas-microsoft-com:unattend"
    xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State">
    <settings pass="offlineServicing"></settings>
    <settings pass="windowsPE">
        <component name="Microsoft-Windows-Setup" processorArchitecture="amd64"
            publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
            <RunSynchronous>
                <RunSynchronousCommand wcm:action="add">
                    <Order>1</Order>
                    <Path>reg.exe add "HKLM\SYSTEM\Setup\LabConfig" /v "BypassTPMCheck" /t REG_DWORD
                        /d 1 /f</Path>
                </RunSynchronousCommand>
                <RunSynchronousCommand wcm:action="add">
                    <Order>2</Order>
                    <Path>reg.exe add "HKLM\SYSTEM\Setup\LabConfig" /v "BypassSecureBootCheck" /t
                        REG_DWORD /d 1 /f</Path>
                </RunSynchronousCommand>
                <RunSynchronousCommand wcm:action="add">
                    <Order>3</Order>
                    <Path>reg.exe add "HKLM\SYSTEM\Setup\LabConfig" /v "BypassRAMCheck" /t REG_DWORD
                        /d 1 /f</Path>
                </RunSynchronousCommand>
            </RunSynchronous>
        </component>
    </settings>
    <settings pass="generalize"></settings>
    <settings pass="specialize">
        <component name="Microsoft-Windows-Deployment" processorArchitecture="amd64"
            publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
            <RunSynchronous>
                <RunSynchronousCommand wcm:action="add">
                    <Order>1</Order>
                    <Path>reg.exe add "HKLM\SYSTEM\Setup\MoSetup" /v
                        "AllowUpgradesWithUnsupportedTPMOrCPU" /t REG_DWORD /d 1 /f </Path>
                </RunSynchronousCommand>
                <RunSynchronousCommand wcm:action="add">
                    <Order>2</Order>
                    <Path>reg.exe add "HKLM\SYSTEM\CurrentControlSet\Control\BitLocker" /v
                        "PreventDeviceEncryption" /t REG_DWORD /d 1 /f</Path>
                </RunSynchronousCommand>
                <RunSynchronousCommand wcm:action="add">
                    <Order>3</Order>
                    <Path>reg.exe add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\OOBE" /v
                        "BypassNRO" /t REG_DWORD /d 1 /f</Path>
                </RunSynchronousCommand>
                <RunSynchronousCommand wcm:action="add">
                    <Order>4</Order>
                    <Path>reg.exe add "HKLM\Software\Policies\Microsoft\Windows\WindowsUpdate\AU" /v
                        "AUOptions" /t REG_DWORD /d 2 /f</Path>
                </RunSynchronousCommand>
                <RunSynchronousCommand wcm:action="add">
                    <Order>5</Order>
                    <Path>reg.exe add "HKLM\Software\Policies\Microsoft\Windows\WindowsUpdate\AU" /v
                        "NoAutoUpdate" /t REG_DWORD /d 1 /f</Path>
                </RunSynchronousCommand>
            </RunSynchronous>
        </component>
    </settings>
    <settings pass="auditSystem"></settings>
    <settings pass="auditUser"></settings>
    <settings pass="oobeSystem"></settings>
</unattend>
```

Sauvegardez le fichier.

Explication:
- La passe WindowsPE contient des commandes à lancer lors du chargement de l'environnement d'installation. Il s'agit d'entrées au registre qui modifieront le comportement de l'assistant d'installation.
  - `BypassTPMCheck`: L'installation peut procéder même si aucun TPM n'a été détecté
  - `BypassSecureBootCheck`: L'installation peut procéder même si SecureBoot n'est pas supporté
  - `BypassRAMCheck`: L'installation peut procéder même si le système n'a pas suffisemment de RAM
- La passe de spécialisation a lieu après le premier démarrage, lorsque l'instance de Windows copiée sur le disque dur démarre pour la première fois. Il s'agit là aussi d'entrées au registre.
  - `AllowUpgradesWithUnsupportedTPMOrCPU`: Windows peut être installé même sur du matériel non supporté
  - `PreventDeviceEncryption`: Indique explicitement à Windows de **ne pas** activer BitLocker
  - `BypassNRO`: Permet l'installation même lorsque la machine est déconnectée d'Internet
  - AUOptions et NoAutoUpdate: Désactive les mises à jour automatiques en arrière-plan


### Étape 3: Production du nouvel ISO

Dans le menu Démarrer, démarrer la console de l'ADK. Elle se nomme "**Environnement de déploiement et d’outils de création d’images**" et est située dans le dossier "**Windows Kits**". Cette console nécessite une **élévation en tant qu'administrateur**.

Puis lancez la commande suivante:

```
oscdimg -u1 -pEF -m -h -bc:\temp\Win11\iso\efi\microsoft\boot\efisys.bin c:\temp\Win11\iso c:\temp\Win11\Windows_11_24H2_FR_NoTPM_NoAU.iso
```

Explication:
- `-u1` détermine le système de fichiers du ISO (dans ce cas, UDF + ISO9660)
- `pEF` détermine l'identifiant de plateforme dans le catalogue El Torito (0xEF désigne UEFI)
- `-m` ignore la taille maximale pour une image de DVD (l'image de Windows pèse plus de 4,7 Go)
- `-h` inclut tous les fichiers et dossiers cachés
- `-b<Chemin>` spécifie le fichier à déposer dans le secteur de démarrage El Torito (`efisys.bin`)
- Ensuite, le chemin de la racine du média à produire
- Finalement, le chemin du fichier ISO qui sera créé

On peut ensuite tester le ISO sur une VM et, si ça fonctionne, le téléverser sur `\\ED5DEPINFO\Logiciels\MS\Windows 11`.

Si possible, suivre la nomenclature suivante:

`Windows_11_<Release>_<Langue>_NoTPM_NoAU.ISO`


## Support

Pour questions, commentaires ou préoccupations, adressez-vous à Vincent Carrier.
