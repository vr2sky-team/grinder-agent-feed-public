README — Exploitation CPP (court)

Petit mémo d’exploitation pour l’agent grinder-cpp. Coller ce fichier dans docs/ du repo ou sur le serveur (/opt/grinder ou /var/lib/grinder) selon ton usage.

1) Vérifier service & timer
# statut
systemctl status grinder-cpp.service --no-pager
systemctl status grinder-cpp.timer  --no-pager

# relancer / forcer une exécution
sudo systemctl start grinder-cpp.service
sudo journalctl -u grinder-cpp.service -n 200 --no-pager

2) Vérifier les artefacts (local)
# contenu & méta
ls -l /var/lib/grinder/
jq '.updated_at, (.checklist|length // 0), .runs[0].id?' /var/lib/grinder/agent_feed.json
jq 'keys' /var/lib/grinder/agent_context.json
jq '.checklist' /var/lib/grinder/agent_feed.json

3) Vérifier le feed distant (GitHub API)
# utiliser le token stocké dans /etc/grinder-cpp.env (FEED_READ_TOKEN)
curl -sH "Authorization: token $(sudo awk -F= '/FEED_READ_TOKEN/{print $2}' /etc/grinder-cpp.env)" \
  "https://api.github.com/repos/vr2sky/grinder-agent-feed/contents/snapshots/main/latest/agent_feed.json" \
| jq -r '.content' | base64 -d | jq '.updated_at, (.checklist|length // 0), .runs[0].id?'

4) Logs et diagnostic rapides
# logs agent
sudo tail -n 200 /var/log/grinder/cpp-agent.log

# rechercher erreurs/warnings récents
sudo grep -E '\[OK\]|\[WARN\]|\[ERREUR\]' /var/log/grinder/cpp-agent.log | tail -n 80

5) Dépannage rapide (patchs courants)
# 1) Forcer refresh du service
sudo systemctl restart grinder-cpp.service

# 2) Si checklist absent -> vérifier workflow GitHub (publish-agent-feed.yml) et index.json
# 3) Appliquer fallback si script attend .architecture.checklist
sudo sed -n '1,160p' /opt/grinder/cpp-agent   # inspecter le script
# (ex: remplacer .architecture.checklist par (.architecture.checklist // .checklist))

# 4) Perms/exec
sudo chmod +x /opt/grinder/cpp-agent
sudo chown root:root /opt/grinder/cpp-agent   # ou herve:herve selon ta politique

Scénarios de panne courants & actions rapides

WARNING: checklist vide

Cause fréquente : le feed contient checklist à la racine mais le script cherche .architecture.checklist.

Action : ajouter fallback dans le script ou modifier le workflow pour publier la forme attendue.

Artefacts locaux non synchronisés (LOCAL != REMOTE)

Cause : problème réseau / token invalide / FEED_BASE_URL mal configurée.

Action : vérifier /etc/grinder-cpp.env (FEED_READ_TOKEN, FEED_BASE_URL) puis curl la ressource distante.

Service qui ne démarre pas (code sortie ≠ 0)

Cause : permissions, dépendances ou erreurs de parsing JSON.

Action : sudo journalctl -u grinder-cpp.service -e → corriger le script et tester localement (bash /opt/grinder/cpp-agent).

Workflow CI qui publie sans checklist

Cause : fail de step jq ou chemin différent dist/.

Action : relire le workflow, vérifier dist/*.json dans le job run (logs), corriger et re-run.

Permissions / propriétaires incorrects

Action : sudo chown herve:herve /var/lib/grinder/* et sudo chmod 644 /var/lib/grinder/*.json (ou selon ta politique).
