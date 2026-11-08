# Cybersecurity Homelab — De zéro jusqu'à Active Directory

Homelab complet construit avec VirtualBox pour pratiquer les compétences techniques demandées pour un stage en cybersécurité sur le marché québécois/canadien : exploitation, pentesting web, détection d'intrusion, SIEM et attaques Active Directory.

## Architecture du lab

Réseau isolé (Host-only Adapter, `192.168.56.0/24`) avec 5 machines virtuelles :

| Machine | Rôle | IP |
|---|---|---|
| Kali Linux | Machine d'attaque (Nmap, Metasploit, Burp Suite, Impacket, BloodHound) | 192.168.56.10 |
| Metasploitable2 | Cible vulnérable (services réseau + appli web) | 192.168.56.20 |
| Ubuntu Server | IDS (Suricata) + SIEM (Splunk) | 192.168.56.30 |
| Windows Server | Contrôleur de domaine Active Directory | 192.168.56.40 |
| Windows 11 | Poste client joint au domaine | 192.168.56.50 |

## Ce qui a été réalisé

### 1. Reconnaissance et exploitation
- Scan de ports et de services avec **Nmap** (`-sV`, `-sC`, `--script vuln`)
- Exploitation avec **Metasploit** : accès root via la backdoor **vsftpd 2.3.4** (CVE-2011-2523)
- Détail complet dans [`pentest-report.md`](./pentest-report.md)

### 2. Pentesting d'application web (OWASP Top 10)
Cible : application web vulnérable sur Metasploitable2, interception avec **Burp Suite**.
- Injection SQL (UNION-based)
- XSS Stored
- Upload de fichier malveillant (webshell PHP)
- Command Injection

### 3. Détection d'intrusion — Suricata
IDS déployé sur Ubuntu Server, avec une règle de détection personnalisée pour identifier les scans de ports (Nmap SYN scan). Voir [`suricata/local.rules`](./suricata/local.rules).

### 4. SIEM — Splunk
Centralisation et analyse des logs Suricata, avec recherches SPL et alertes automatisées. Voir [`splunk/queries.spl`](./splunk/queries.spl).

### 5. Active Directory — Enumération et attaques
Déploiement d'un domaine Active Directory complet (`lab.local`) avec contrôleur de domaine, poste client joint au domaine, et comptes de service.

- **Cartographie du domaine** avec BloodHound CE (collecte via SharpHound) pour visualiser les chemins d'attaque potentiels
- **Kerberoasting** : identification d'un compte de service avec SPN (`svc_sql`), extraction du hash Kerberos TGS via Impacket (`GetUserSPNs.py`)
- **Pass-the-Hash** : extraction de hashs NTLM (`secretsdump.py`), tests d'authentification via SMB — démonstration du principe de moindre privilège (le compte de service n'avait pas les droits d'administration nécessaires pour un mouvement latéral complet)
- **Détection** : analyse des Event Logs Windows pertinents (4769 — demande de ticket Kerberos, 4624/4625 — succès/échec d'authentification, 4776 — validation NTLM) pour corréler les attaques avec les traces laissées dans les journaux

Détails dans [`active-directory/kerberoasting-pth.md`](./active-directory/kerberoasting-pth.md).

## Outils utilisés

Nmap · Metasploit/Meterpreter · Burp Suite · John the Ripper · Suricata · Splunk · BloodHound CE / SharpHound · Impacket (GetUserSPNs, secretsdump, psexec, wmiexec) · PowerShell

## Méthodologie

Chaque vulnérabilité identifiée a été documentée selon une approche formelle : preuve de concept, sévérité, et recommandation de remédiation — voir [`pentest-report.md`](./pentest-report.md).

---
*Projet réalisé par Roy Nguiffo, étudiant finissant en informatique (concentration cybersécurité) à l'UQAC, dans le cadre d'une préparation à un stage en cybersécurité.*
