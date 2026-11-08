# Active Directory — Enumération et attaques

## Environnement
- **Domaine :** `lab.local`
- **Contrôleur de domaine :** Windows Server (192.168.56.40)
- **Poste client :** Windows 11 joint au domaine (192.168.56.50)
- **Comptes de service :** `svc_sql` (SPN enregistré pour un service SQL)

## 1. Cartographie avec BloodHound CE

Collecte des données du domaine via **SharpHound** (collecteur exécuté sur un poste joint au domaine), puis import dans BloodHound CE (backend Neo4j) pour visualiser :
- Les relations d'appartenance aux groupes
- Les sessions actives
- Les chemins d'attaque potentiels (ex. : escalade de privilèges via délégation, ACL mal configurées)

## 2. Kerberoasting

**Principe :** tout compte utilisateur avec un SPN (Service Principal Name) enregistré peut se voir délivrer un ticket Kerberos (TGS) par n'importe quel utilisateur authentifié du domaine — ce ticket est chiffré avec le hash du mot de passe du compte de service. S'il est extrait, il peut être cracké hors ligne sans déclencher de verrouillage de compte.

**Commande utilisée (Impacket) :**
```
python3 GetUserSPNs.py 'lab.local/<utilisateur>:<mot_de_passe>' -request -dc-ip 192.168.56.40 -outputfile svc_sql_hash.txt
```

**Résultat :**
- ✅ SPN de `svc_sql` identifié et ticket demandé avec succès
- ✅ Hash Kerberos TGS extrait
- ❌ Cracking avec `rockyou.txt` (via John the Ripper, format `krb5tgs`) : échec — mot de passe suffisamment complexe pour résister à une attaque par dictionnaire standard

**Point technique notable :** Kerberos exige une synchronisation d'horloge stricte entre le client et le contrôleur de domaine (tolérance par défaut : 5 minutes). Une dérive d'horloge (date ou simplement fuseau horaire différent) produit l'erreur `KRB_AP_ERR_SKEW` et bloque toute requête Kerberos — un point de dépannage réseau/temps essentiel pour ce type d'attaque.

## 3. Pass-the-Hash

**Principe :** avec un hash NTLM valide, il est possible de s'authentifier sur un service compatible NTLM (SMB, WMI) sans connaître le mot de passe en clair.

**Extraction des hashs (sur le DC, après compromission) :**
```
python3 secretsdump.py lab.local/<admin>:<mot_de_passe>@192.168.56.40
```

**Tests effectués :**
- Hash local Administrateur du DC contre le poste Windows 11 → **échec (STATUS_ACCESS_DENIED)**, attendu : les hashs SAM locaux sont propres à chaque machine et ne se transfèrent pas entre machines différentes.
- Hash du compte de service `svc_sql` (domaine) via `psexec.py` / `wmiexec.py` contre le DC → **échec (ACCESS_DENIED / rpc_s_access_denied)**, attendu : `svc_sql` n'a aucun droit d'administration sur le DC.

**Conclusion :** ces échecs ne sont pas des échecs de méthodologie — ils démontrent concrètement le principe de **moindre privilège** : un compte de service compromis ne doit pas permettre de mouvement latéral si ses droits sont correctement restreints.

## 4. Détection — Event Logs Windows

Événements pertinents identifiés et corrélés avec les attaques ci-dessus :

| Event ID | Signification | Pertinence |
|---|---|---|
| 4769 | Demande de ticket de service Kerberos (TGS) | Trace directe d'une tentative de Kerberoasting (surtout en volume anormal sur un court intervalle) |
| 4624 | Connexion réussie | Confirme une authentification NTLM/Kerberos réussie (ex. après Pass-the-Hash) |
| 4625 | Échec de connexion | Signale les tentatives d'authentification refusées (ex. Pass-the-Hash avec droits insuffisants) |
| 4776 | Validation d'identifiants NTLM | Trace des authentifications NTLM, utile pour détecter du Pass-the-Hash |

## Recommandations

- Appliquer des mots de passe longs et aléatoires (type gMSA — Group Managed Service Account) pour tous les comptes de service avec SPN
- Limiter strictement les droits des comptes de service au minimum nécessaire
- Surveiller activement les Event ID 4769 pour détecter un volume anormal de demandes TGS (signature de Kerberoasting automatisé)
- Désactiver ou restreindre l'authentification NTLM lorsque Kerberos pur est disponible
