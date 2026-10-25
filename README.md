# FogWatch : détection collaborative d'attaques SSH distribuées sur une architecture fog

Projet académique. Plusieurs sites « fog » surveillent chacun leur trafic avec **Suricata** et partagent
leurs observations par **MQTT**. Un **corrélateur** additionne ce que chaque site voit et détecte une
attaque discrète répartie sur plusieurs sites, qu'aucun site ne verrait seul. L'IP de l'attaquant est
alors bloquée sur **tous** les sites.

## Le problème

Un IDS classique surveille son propre site avec un seuil (par exemple 5 tentatives SSH). Un attaquant
« low and slow » reste sous ce seuil sur chaque site (2 ou 3 tentatives par site) : aucun site ne lève
d'alerte. Si les sites se partagent leurs comptes, la même IP apparaît sur plusieurs sites et l'attaque
devient visible.

## Architecture

```
                 Machine attaquante (hors cluster, hors allowlist)
                  |  2-3 tentatives SSH par site
        +---------+----------+
        v                    v                     v
   Site A (master)        Site B                Site C
   Suricata               Suricata              Suricata
   fog-agent              fog-agent             fog-agent
   response-agent         response-agent        response-agent
        |  fog/<site>/events (comptes)              |
        +-------------------+-----------------------+
                            v
                  Broker MQTT (Mosquitto, sur le master)
                            v
                       Corrélateur (master)
                            |  fog/alerts + fog/<site>/block
                            v
                 response-agent de chaque site : IP bloquée (nftables)
```

| Composant | Dossier | Rôle |
|---|---|---|
| Règles Suricata | `detection/` | `local.rules` : 1 alerte (SID 9000001) par tentative de connexion SSH (SYN) |
| fog-agent | `fog-agent/` | Compte les tentatives par IP (fenêtre glissante), alerte locale au seuil, publie sur MQTT |
| Corrélateur | `correlator/` | Garde le dernier compte de chaque site par IP, déclenche l'alerte globale, ordonne le blocage |
| response-agent | `response-agent/` | Écoute `fog/<site>/block` et bloque l'IP (table nftables `fogwatch`, expiration automatique) |
| Attaque et mesures | `attack-scripts/` | `distributed_attack.py` (démo), `campaign.py` (campagne de mesures) |
| Déploiement | `k8s/` | Manifests K3s : ConfigMap commune + un Deployment par site |
| ML (optionnel) | `ml/` | Classifieur de flux SSH entraîné sur CICIDS2017 (pas encore branché dans l'agent) |

## Prérequis

- Ubuntu Server (24.04 ou plus récent), **Tailscale** sur toutes les machines (même tailnet)
- Suricata, **même version sur les 3 sites**
- Python 3 et `paho-mqtt` (`sudo apt install -y python3-paho-mqtt`), `nftables`, `mosquitto-clients`
- Une **4e machine** sur Tailscale pour l'attaquant : elle ne doit être ni un site, ni un nœud K3s

Adresses utilisées dans ce document (à adapter) : master/site A `100.103.105.18`, site B `100.75.234.57`,
site C `100.78.123.31`.

## Installation d'un site (A, B et C)

1. Nom unique : `sudo hostnamectl set-hostname site-X` puis `sudo tailscale set --hostname=site-X`
2. `git clone https://github.com/kammounimen/FogWatch.git && cd FogWatch`
3. `bash detection/install-suricata.sh`
4. Règles : `sudo cp detection/local.rules /var/lib/suricata/rules/local.rules`
5. Dans `/etc/suricata/suricata.yaml` : `interface: tailscale0` et `HOME_NET: "[<IP Tailscale de CETTE VM>/32]"`
   (la seule valeur différente d'un site à l'autre). Pour le module ML, ajouter `- flow` dans `eve-log` → `types`.
6. `sudo suricata -T -c /etc/suricata/suricata.yaml && sudo systemctl restart suricata`
7. `sudo apt install -y python3-paho-mqtt nftables mosquitto-clients`

## Master : broker et corrélateur

```bash
sudo apt install -y mosquitto mosquitto-clients
echo -e "listener 1883 0.0.0.0\nallow_anonymous true" | sudo tee /etc/mosquitto/conf.d/lab.conf
sudo systemctl restart mosquitto && sudo systemctl enable mosquitto
```

> Mosquitto est ouvert **sans authentification** : acceptable en laboratoire sur un réseau Tailscale privé,
> à ne pas reproduire ailleurs.

```bash
cd correlator
python3 correlator.py --selftest        # 12 tests de logique, sans broker
ALLOWLIST=100.103.105.18,100.75.234.57,100.78.123.31 COOLDOWN_SECONDS=20 python3 correlator.py
```

## Lancer un site (manuel)

```bash
nc -zv 100.103.105.18 1883              # le broker doit répondre "succeeded"

cd fog-agent
sudo SITE_ID=B MQTT_BROKER=100.103.105.18 WINDOW_SECONDS=30 LOCAL_ALERT_THRESHOLD=5 python3 agent.py

cd response-agent
sudo SITE_ID=B MQTT_BROKER=100.103.105.18 BLOCK_SECONDS=20 \
     ALLOWLIST=100.103.105.18,100.75.234.57,100.78.123.31 python3 respond.py
```

Sur le master, `MQTT_BROKER=100.103.105.18` fonctionne aussi pour son propre agent (site A).
Pour observer le bus : `mosquitto_sub -h localhost -t 'fog/#' -v`.

## Démonstration

Depuis la **machine attaquante** (dossier `attack-scripts/`) :

```bash
python3 distributed_attack.py           # 3 tentatives par site, 5 s entre sites
```

Résultat attendu : chaque site affiche `count: 3` sans `local_alert` ; le corrélateur affiche
`[ALERTE GLOBALE]` puis un `[BLOCAGE]` par site ; `sudo nft list set inet fogwatch blocked` montre l'IP
de l'attaquant sur chaque site. Débloquer à la main : `sudo nft flush set inet fogwatch blocked`.

## Campagne de mesures

Réglages nécessaires pour enchaîner les essais avec la même IP attaquante : corrélateur
`COOLDOWN_SECONDS=20`, `respond.py` `BLOCK_SECONDS=20`, et `--cooldown` de la campagne supérieur à
`WINDOW_SECONDS + REPORT_SECONDS` (45 s par défaut).

```bash
cd attack-scripts
python3 campaign.py --broker 100.103.105.18 --label test --tentatives 1,3 --repeat 2        # vérification
python3 campaign.py --broker 100.103.105.18 --label collab --tentatives 1,2,3,4,6 --repeat 10
```

Chaque essai écrit une ligne dans `results_<label>.csv` : `local_detected` (un site alerte seul, donc
« sites isolés »), `global_detected` (alerte du corrélateur), `detected_any` (« collaboratif »),
latences et comptes maximaux par site. Contrôle : relancer le corrélateur avec `MODE=isolated`, la
colonne `global_detected` doit rester à 0. Avec 1 tentative par site, toute détection est un faux positif.

## Configuration

**fog-agent**

| Variable | Défaut | Rôle |
|---|---|---|
| `SITE_ID` | `A` | Identifiant du site (A, B, C) |
| `MQTT_BROKER` | vide | IP du broker ; vide = affichage seulement |
| `WINDOW_SECONDS` | 30 | Fenêtre glissante de comptage par IP |
| `LOCAL_ALERT_THRESHOLD` | 5 | Tentatives qui déclenchent l'alerte locale (immédiate, à la 5e) |
| `REPORT_SECONDS` | 5 | Période de rapport des observations |
| `REPORT_MIN_COUNT` | 2 | Compte minimal pour publier une observation (ignore les connexions isolées) |
| `EVE_PATH` | `/var/log/suricata/eve.json` | Journal Suricata |

**Les 3 sites doivent avoir le même fichier `agent.py` et les mêmes valeurs.**

**Corrélateur**

| Variable | Défaut | Rôle |
|---|---|---|
| `MODE` | `collaborative` | `isolated` : aucune alerte globale (ligne de base) |
| `MIN_PER_SITE` | 2 | Tentatives minimales pour qu'un site compte |
| `MIN_SITES` | 2 | Sites touchés pour déclencher |
| `GLOBAL_THRESHOLD` | 5 | Total minimal |
| `STALE_SECONDS` | 45 | Un compte non rafraîchi est oublié |
| `COOLDOWN_SECONDS` | 300 | Pas de nouvelle alerte pour la même IP |
| `ALLOWLIST` | vide | IP jamais corrélées ni bloquées |
| `SITES` | `A,B,C` | Sites qui reçoivent l'ordre de blocage |
| `LOG_FILE` | `correlator_log.jsonl` | Journal des alertes (pour les mesures) |

**response-agent** : `SITE_ID`, `MQTT_BROKER`, `ALLOWLIST` (obligatoire, y mettre master, sites et machine
d'administration, **jamais l'attaquant**), `BLOCK_SECONDS` (300), `DRY_RUN=1` (affiche les commandes nft sans les exécuter).

Ces valeurs sont des choix d'ingénierie, pas des constantes tirées de la littérature : le rapport en étudie
la sensibilité (voir *Mesures*).

## Messages MQTT

| Topic | Émetteur | Contenu (exemple) |
|---|---|---|
| `fog/<site>/events` | fog-agent | `{"site_id":"B","src_ip":"100.1.2.3","type":"ssh_fail","count":3,"window_start":"...","window_end":"...","local_alert":false,"kind":"observation"}` |
| `fog/alerts` | corrélateur | `{"action":"global_alert","ip":"100.1.2.3","sites_touched":["A","B"],"total_count":6,"latency_s":2.0,...}` |
| `fog/<site>/block` | corrélateur | `{"ip":"100.1.2.3","reason":"distributed_ssh_fail"}` |

`count` est le nombre de tentatives de l'IP **sur ce site pendant les `WINDOW_SECONDS` dernières secondes** :
les fenêtres se recouvrent, donc le corrélateur garde seulement le dernier compte de chaque site et ne
additionne jamais les messages d'un même site. `kind` vaut `local_alert` (alerte immédiate au seuil) ou
`observation` (rapport périodique, `count: 0` quand l'IP n'est plus vue).

## Déploiement K3s

```bash
# nœud agent (sites B et C) : même IP Tailscale que pour l'installation du serveur
curl -sfL https://get.k3s.io | K3S_URL=https://100.103.105.18:6443 K3S_TOKEN=<TOKEN> \
  sh -s - agent --node-ip $(tailscale ip -4) --flannel-iface tailscale0

kubectl label node site-a site=A && kubectl label node site-b site=B && kubectl label node site-c site=C
docker build -t fog-agent:v3 fog-agent/     # le Dockerfile doit installer paho-mqtt
sudo docker save fog-agent:v3 | sudo k3s ctr images import -     # sur CHAQUE nœud (imagePullPolicy: Never)
kubectl create namespace fogwatch
kubectl apply -f k8s/fog-agent-manifests.yaml
kubectl -n fogwatch get pods -o wide
```

Les manifests partagent une ConfigMap (`fog-agent-config`) : les 3 sites ont donc les mêmes paramètres.
`MQTT_BROKER` y est commenté : à activer **seulement** quand `nc -zv 100.103.105.18 1883` répond
`succeeded` depuis chaque nœud (sinon l'agent s'arrête au démarrage et le pod redémarre en boucle), puis
`kubectl -n fogwatch rollout restart deployment --all`.

## Module ML (optionnel)

Classifieur Random Forest : flux SSH normal contre SSH-Patator, entraîné sur le fichier du mardi de
**CICIDS2017** (`MachineLearningCSV.zip`, jeu de données du CIC de l'UNB), avec 8 features que Suricata
peut fournir (durée, paquets et octets dans chaque sens, flags SYN/FIN/RST).

- Entraînement : `ml/train_ssh.ipynb` (Colab) ou `ml/train.py`. Validation croisée à 5 plis : F1 ≈ 0,96.
- Utilisation sur un site : `ml/score_flows.py` lit les événements `flow` de `eve.json` et affiche un score.
- **Installer exactement la même version de scikit-learn que celle de l'entraînement** (1.6.1), sinon le
  fichier `.joblib` peut échouer au chargement.
- **Réserve :** le modèle n'est pas encore intégré au fog-agent, et on n'a pas vérifié que les flux de
  Suricata ressemblent à ceux de CICFlowMeter (la durée du flux, variable la plus importante, est la plus fragile).

## Limites connues (à reprendre dans le rapport)

- **Corrélation par adresse IP** : un attaquant qui change d'IP échappe au système (perspective : corrélation par comportement).
- **Échelle de laboratoire**, une seule IP attaquante et un seul type d'attaque (tentatives SSH).
- **Broker unique** : si le master tombe, la corrélation s'arrête (les alertes locales continuent).
- **Ordres de blocage** publiés en QoS 1 mais reçus en QoS 0 par `respond.py` : un ordre peut se perdre si un site est déconnecté.
- **Suricata voit les paquets avant le pare-feu** : après blocage, les tentatives continuent d'être comptées.
- **Un attaquant à une seule tentative par site** n'est pas détecté avec `MIN_PER_SITE=2` (compromis avec les faux positifs).
- **Horloges** : les agents utilisent l'heure d'arrivée de chaque alerte ; garder les VMs synchronisées (`chrony`).

## Pièges connus

- Les règles Suricata vont dans `/var/lib/suricata/rules/`, pas dans `/etc/`.
- `HOME_NET` = IP Tailscale de la VM en `/32`, différent sur chaque site.
- Hostname unique **avant** d'installer K3s (deux nœuds `ubuntu` se bloquent).
- Ne pas activer Tailscale SSH.
- L'attaquant ne doit pas être un site ni le master : un SSH vers soi-même ne passe pas par `tailscale0`, et son IP serait dans l'allowlist.
- Avec `sudo`, Python n'utilise pas les paquets installés avec `pip` pour ton utilisateur : utiliser `python3-paho-mqtt` (apt).
- `nmap -p 22,80` n'envoie qu'un SYN vers le port 22 : utiliser `distributed_attack.py` pour compter 3 tentatives par site.
- Ne **jamais** committer le token K3s, les logs, ni les données CICIDS2017 (le dépôt est public).

## Règle d'or

Aucun réglage manuel durable sur une VM : tout passe par Git (règles, manifests, paramètres), pour que les
3 sites restent identiques.

## Structure du dépôt

```
detection/        local.rules, install-suricata.sh
fog-agent/        agent.py, Dockerfile
response-agent/   respond.py
correlator/       correlator.py
attack-scripts/   distributed_attack.py, campaign.py
k8s/              fog-agent-manifests.yaml
ml/               train_ssh.ipynb, train.py, score_flows.py
README.md
```

## Mesures et résultats

À compléter après la campagne : taux de détection isolé et collaboratif selon le nombre de tentatives par
site, faux positifs, latence, étude de sensibilité (fenêtre, seuils, `MIN_PER_SITE`). Les données brutes
viennent de `attack-scripts/results_*.csv` et du journal `correlator_log.jsonl`.

## Références

- Vasilomanolakis, Karuppayah, Mühlhäuser, Fischer. *Taxonomy and Survey of Collaborative Intrusion Detection*. ACM Computing Surveys 47(4), 2015. DOI 10.1145/2716260
- Javed, Paxson. *Detecting Stealthy, Distributed SSH Brute-Forcing*. ACM CCS 2013
- Jung, Paxson, Berger, Balakrishnan. *Fast Portscan Detection Using Sequential Hypothesis Testing*. IEEE S&P 2004. DOI 10.1109/SECPRI.2004.1301325
- Sharafaldin, Habibi Lashkari, Ghorbani. *Toward Generating a New Intrusion Detection Dataset and Intrusion Traffic Characterization*. ICISSP 2018. DOI 10.5220/0006639801080116
- Engelen, Rimmer, Joosen. *Troubleshooting an Intrusion Detection Dataset: the CICIDS2017 Case Study*. IEEE SPW 2021. DOI 10.1109/SPW53761.2021.00009

## Équipe

À compléter (noms, rôles par site, encadrant).
