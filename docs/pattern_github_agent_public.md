Pattern GitHub ↔ Agent via miroir public
(Canal de synchronisation sécurisé entre CI et agent ChatGPT)
🎯 Objectif

Permettre à un agent externe (ChatGPT, bot, dashboard) de lire l’état du projet sans accès au repo privé, via un miroir public minimal maintenu automatiquement par GitHub Actions.

1️⃣ Structure du pattern
Élément	Rôle	Exemple
Repo privé producteur	Source de vérité (CI, workflows, secrets)	vr2sky/grinder
Workflow CI (publish-feed.yml)	Génère et publie les fichiers JSON d’état	/dist/*.json
Repo public miroir	Canal de lecture ouvert, sans token	vr2sky/grinder-agent-feed-public
Agent externe	Consommateur des données, lecture web-only	ChatGPT Chef de projet (CPP)
2️⃣ Fichiers synchronisés
Fichier	Contenu	Exemple de lecture
agent_feed.json	Métadonnées d’exécution CI (runs, checks, checklist)	updated_at, runs[0].id
agent_context.json	Contexte synthétique projet (ou placeholder)	_meta, context
index.json	Manifest du run (ref, repo, sha, timestamp)	ref, sha, run_id
3️⃣ URLs publiques (exemples)
https://raw.githubusercontent.com/vr2sky/grinder-agent-feed-public/main/latest/agent_feed.json
https://raw.githubusercontent.com/vr2sky/grinder-agent-feed-public/main/latest/agent_context.json
https://raw.githubusercontent.com/vr2sky/grinder-agent-feed-public/main/latest/index.json


Ces URL sont :

stables et versionnées (main/latest)

accessibles sans token

mises à jour automatiquement à chaque run CI

non exploitables (pas de secrets, pas de data utilisateur)

4️⃣ Schéma de fonctionnement
[ GitHub Actions - repo privé ]
        │
        │ (push dist/*.json)
        ▼
[ grinder-agent-feed-public ]
        │
        │ (lecture web, JSON)
        ▼
[ Agent Chef de projet ChatGPT ]

5️⃣ Avantages clés

✅ Aucun secret exposé – le token reste côté CI
✅ Lecture universelle – accessible depuis n’importe quel agent IA
✅ Auditabilité totale – historique dans le repo public
✅ Reproductibilité – même logique applicable à d’autres agents (DEV, OPS, TEST…)
✅ Robustesse CI/CD – pas de dépendance à l’état du connecteur ChatGPT

🔒 Option : durcir la publication

Ne publier que agent_feed.json et index.json (si agent_context.json reste vide)

Activer branch protection sur main du miroir public

Ajouter un fichier NOTICE.md précisant :

“Ce dépôt ne contient que des métadonnées techniques anonymes pour usage d’agents automatisés. Aucune donnée sensible n’y est stockée.”

---

## 🧰 Maintenance et reprise

### 🔁 Forcer une republlication manuelle du miroir public
Si le miroir public `grinder-agent-feed-public` n’est pas à jour (ex : décalage de timestamp, échec de push CI) :

```bash
# Depuis le repo privé producteur
gh workflow run "Publish Agent Feed" --ref main

Cela relancera la génération de dist/*.json et leur push automatique vers le miroir public.

🧼 Réparer un miroir désynchronisé

Si un fichier du miroir public est manquant ou obsolète :

cd ~/github/grinder-agent-feed-public
git pull
rm -rf latest
mkdir latest
# Copier depuis le repo privé ou dist local
cp ~/github/grinder/dist/*.json latest/
git add latest
git commit -m "fix: resync latest feed"
git push


🧩 Vérification côté agent

Pour tester que le Chef de projet (CPP) lit bien les bonnes données :

[CPP] Lis ces URLs publiques :
- agent_feed.json : https://raw.githubusercontent.com/vr2sky/grinder-agent-feed-public/main/latest/agent_feed.json
- agent_context.json : https://raw.githubusercontent.com/vr2sky/grinder-agent-feed-public/main/latest/agent_context.json
- index.json : https://raw.githubusercontent.com/vr2sky/grinder-agent-feed-public/main/latest/index.json

Dis-moi :
1) La branche (ref) et le run_id.
2) Vérifie qu'une checklist>0 existe.
3) Donne une synthèse ≤3 lignes (pas de dump JSON).

✅ Si la réponse contient ref=main, checklist=3, et run_id cohérent, la liaison CI ↔ Agent est valide.

Bonnes pratiques d’exploitation

Lancer un Publish Agent Feed après toute mise à jour importante du projet.

Vérifier périodiquement les logs du service grinder-cpp :

sudo tail -n 100 /var/log/grinder/cpp-agent.log | grep '\[OK\]'


Conserver le miroir public “latest” en mode lecture seule (pas d’édition manuelle).

Protéger la branche main du miroir public pour éviter tout push direct.

### 🔁 Forcer une republlication manuelle du miroir public
Si le miroir public `grinder-agent-feed-public` n’est pas à jour (ex : décalage de timestamp, échec de push CI) :

Maintenance validée – Octobre 2025
Pattern stable et prêt à réutiliser pour les autres agents (DEV, OPS, TEST, DOC).
