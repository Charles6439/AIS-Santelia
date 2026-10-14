# Projet Santélia — Cluster Proxmox en haute disponibilité

Projet réalisé dans le cadre du titre professionnel **Administrateur d'Infrastructures Sécurisées (AIS, niveau Bac+3)** au GRETA de l'Essonne — Bloc 2 « Conception et évolution d'une infrastructure ».

Travail réalisé en groupe de 4 stagiaires. Ce dépôt présente ma contribution et mon compte rendu personnel.

## Contexte

Le Groupe Santélia (cas d'étude) exploite trois cliniques privées en Essonne. Une panne serveur a interrompu le site d'Évry pendant plus de 6 heures et bloqué la prise de rendez-vous. Objectif : fiabiliser l'infrastructure (sauvegarde, supervision, continuité de service) avant un audit de certification, avec une petite DSI et un budget d'environ 15 000 €.

## Contenu du dépôt

| Document | Description |
|---|---|
| [`docs/Cahier_des_charges_Santelia.pdf`](docs/Cahier_des_charges_Santelia.pdf) | Cahier des charges fonctionnel et technique : besoins, exigences, architecture cible, sauvegarde 3-2-1, supervision (Zabbix), automatisation (Ansible), sécurité HDS, benchmark (Proxmox Backup Server), planning et recette |
| [`docs/TP_HAPCA_SeanceA-B_Charles_Maduba_FR.pdf`](docs/TP_HAPCA_SeanceA-B_Charles_Maduba_FR.pdf) | **Compte rendu final (séances A et B réunies)** : RTO et RPO mesurés, calculés et comparés à des cibles justifiées |
| [`docs/TP_HAPCA_SessionA-B_Charles_Maduba_EN.pdf`](docs/TP_HAPCA_SessionA-B_Charles_Maduba_EN.pdf) | Final report, sessions A and B (English version) |
| [`docs/TP_HAPCA_Santelia_Charles.pdf`](docs/TP_HAPCA_Santelia_Charles.pdf) | Compte rendu du TP « HD/PCA » : construction d'un cluster Proxmox de 3 nœuds et test de panne, avec captures d'écran |
| [`docs/TP_HAPCA_Santelia_Charles.docx`](docs/TP_HAPCA_Santelia_Charles.docx) | Même compte rendu, format Word |
| [`docs/TP_HAPCA_SeanceB_Santelia_Charles_FR.pdf`](docs/TP_HAPCA_SeanceB_Santelia_Charles_FR.pdf) | Compte rendu de la séance B : sauvegarde, restauration, RPO et plan de reprise d'activité (PRA) |
| [`docs/TP_HAPCA_SeanceB_Santelia_Charles_EN.pdf`](docs/TP_HAPCA_SeanceB_Santelia_Charles_EN.pdf) | Session B report (English version) |
| [`docs/TP_HAPCA_Santelia_Charles_EN.pdf`](docs/TP_HAPCA_Santelia_Charles_EN.pdf) | Session A report (English version) |
| [`docs/Key_concepts_Santelia.pdf`](docs/Key_concepts_Santelia.pdf) | Fiche de révision (EN) : CDC, clustering Proxmox, NFS, journaux, PCA/PRA |
| [`docs/Key_concepts_SeanceB.pdf`](docs/Key_concepts_SeanceB.pdf) | Fiche de révision (EN) : sauvegarde, restauration, règle 3-2-1, RTO/RPO, PRA |

## Le lab : cluster Proxmox VE de 3 nœuds

Trois serveurs Proxmox VE virtualisés dans VirtualBox, sur un réseau host-only `192.168.56.0/24`.

| Nœud | Adresse IP | Rôle |
|---|---|---|
| pve | 192.168.56.200 | Nœud 1, héberge le partage NFS |
| pve2 | 192.168.56.201 | Nœud 2 |
| pve3 | 192.168.56.202 | Nœud 3 |

### Étapes réalisées

1. **Préparation des nœuds** : nom, adresse IP et identifiant machine uniques, résolution des noms (`/etc/hosts`), synchronisation de l'heure.
2. **Création du cluster** `labcluster` (`pvecm create` / `pvecm add`) et **expérience du quorum** : avec 2 nœuds sur 3 le cluster fonctionne, avec 1 seul il se bloque (protection contre le split-brain).
3. **Stockage partagé NFS** : partage `/srv/nfs/vmstore` sur pve, déclaré dans le cluster (`pvesm add nfs`) et actif sur les 3 nœuds ; disque de la VM de test placé sur ce stockage ; migration vers pve2 en 6 secondes, sans copie de disque.
4. **Haute disponibilité** : VM ajoutée à la HA (`ha-manager`), vérification du watchdog (`softdog`), puis **arrêt brutal d'un nœud** pour mesurer le temps de reprise.

### Résultat du test de panne

| Mesure | Valeur |
|---|---|
| t0 — arrêt brutal de pve2 | 11:51:39 |
| t1 — service relancé par la HA | 11:53:52 |
| t2 — VM de nouveau démarrée | 11:53:56 |
| **RTO mesuré** | **2 min 17 s** |

L'essentiel du délai correspond à l'attente de sécurité (*fencing*) pendant laquelle Proxmox s'assure que le nœud est vraiment arrêté avant de relancer la VM ailleurs. À comparer avec la panne réelle de plus de 6 heures décrite dans le cas Santélia.

### Incidents rencontrés et leçons

- **Ressources limitées** : sur un seul PC hôte, la copie d'un disque de 32 Go a fait planter une VM (Guru Meditation VirtualBox). Solution adaptée : disque vide remplacé par un petit disque créé directement sur le stockage partagé.
- **`no quorum` sur `/etc/pve`** alors que le cluster était quorate : service de configuration resté en lecture seule, résolu par `systemctl restart pve-cluster`.
- **Dérive d'horloge** après une mise en veille de l'hôte : la HA a signalé `detected time drift`. La correction a déclenché l'auto-protection (watchdog) d'un nœud, puis la HA a tout remis en ordre seule. Leçon : la synchronisation NTP est indispensable dans un cluster.
- **Point d'amélioration identifié** : le partage NFS hébergé sur un des nœuds est un point unique de défaillance ; en production, il faudrait un stockage partagé redondant (NAS en HA ou Ceph).

## Séance B — Sauvegarde, restauration et plan de reprise

La haute disponibilité ne protège ni contre une VM supprimée, ni contre un rançongiciel, ni contre la perte du stockage. La séance B ajoute donc la sauvegarde et le plan de reprise.

| Étape | Résultat |
|---|---|
| Sauvegarde manuelle (`vzdump`, mode snapshot, ZSTD) | 10 s, la VM reste en service |
| Restauration de test sous un autre ID (`qmrestore --unique`) | 3,6 s, VM démarrée puis supprimée |
| Sauvegarde automatique | Chaque jour à 21:00, 7 sauvegardes conservées |
| RPO | ≈ 0 en cas de panne d'un nœud (HA) ; jusqu'à 24 h en cas de restauration depuis la sauvegarde |
| Règle 3-2-1 | Non respectée dans le lab (tout est sur un seul PC) : il faudrait un serveur de sauvegarde séparé et une copie hors site |
| PRA de Santélia | Périmètre, scénarios, procédure de restauration, rôles, objectifs comparés aux mesures, tests |

## Compétences mises en œuvre

- Rédaction d'un cahier des charges (besoins, exigences, benchmark, planning, recette)
- Virtualisation et clustering Proxmox VE (corosync, quorum, HA, fencing, watchdog)
- Stockage partagé NFS sous Linux (Debian)
- Diagnostic à partir des journaux (`journalctl`, `pvecm status`, `ha-manager status`)
- Notions de continuité et de reprise d'activité (PCA / PRA, RTO / RPO)
- Sauvegarde et restauration Proxmox (`vzdump`, `qmrestore`, tâches planifiées, rétention), règle 3-2-1, rédaction d'un PRA

## Auteur

**Charles Maduba** — stagiaire Administrateur d'Infrastructures Sécurisées, GRETA de l'Essonne (2026)

---

## English summary

**Context.** Santélia is a case study of a private healthcare group running three clinics in France. A server failure stopped one site for more than 6 hours and blocked online appointments. The goal is to make the infrastructure more reliable (backups, monitoring, service continuity) on a small budget and with a two-person IT team.

**What I built.** A 3-node **Proxmox VE** cluster (virtualized in VirtualBox) with:
- cluster creation and a live **quorum** experiment (2 of 3 nodes keep the cluster running, 1 of 3 blocks it to prevent split-brain);
- **NFS shared storage** available on all three nodes, so a VM can run on any of them;
- **high availability (HA)**: a VM is protected by `ha-manager`, with a software watchdog for fencing.

**Key result.** After a hard power-off of the node running the VM, HA restarted it on another node automatically. **Measured RTO: 2 min 17 s**, compared with the 6+ hour outage in the case study.

**Lessons learned.** Limited host resources (a VM crash during a 32 GB disk copy), a cluster filesystem stuck in read-only mode, and clock drift after the host went to sleep, which showed that time synchronization (NTP) is essential in a cluster. I also identified a single point of failure: the NFS share hosted on one node.

**Skills.** Requirements specification, Proxmox VE clustering (corosync, quorum, HA, fencing), Linux NFS, log-based troubleshooting (`journalctl`, `pvecm`, `ha-manager`), business continuity and disaster recovery concepts (RTO / RPO).

**Session B: backup and disaster recovery.** HA does not protect against deletion, ransomware or storage loss, so session B added backups: a manual snapshot backup (10 s, VM kept running), a proven test restore under another VM ID (3.6 s), a daily scheduled job at 21:00 keeping 7 backups, an RPO analysis per scenario, and a written disaster recovery plan (DRP) for Santélia. The lab does not meet the 3-2-1 rule (everything runs on one PC), which the report states explicitly.

*The requirements specification and the original reports are in French; English versions of both lab reports and two study guides are in `docs/`.*
