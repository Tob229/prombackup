# **Documentation pour l'installation et la configuration du service de snapshot Prometheus**
## **Introduction**
Ce guide décrit les étapes pour installer et configurer un service qui effectue des snapshots des données de Prometheus à l'aide d'un script Bash. Le service est configuré pour s'exécuter chaque lundi à 13h00 et réessaie toutes les 24 heures en cas d'échec.
## **Pré-requis**
- Un serveur avec Prometheus installé et configuré.
- Les outils suivants doivent être installés : bash, curl, jq, tar, systemd.
## **Étape 1 : Créer le script de snapshot**
Créez le fichier de script prometheus_snapshot.sh :
```bash
sudo nano /usr/local/bin/prometheus_snapshot.sh
```

Collez le code suivant dans le fichier :

```bash

#!/bin/bash

# Charger les paramètres du fichier de configuration
CONFIG_FILE="/etc/prometheus-snapshot.conf"
if [ ! -f "$CONFIG_FILE" ]; then
  echo "Fichier de configuration non trouvé: $CONFIG_FILE"
  exit 1
fi
source "$CONFIG_FILE"

# Créer le dossier de destination s'il n'existe pas
mkdir -p "$SNAPSHOT_DIR"

# Fonction pour réaliser le snapshot
take_snapshot() {
  RESPONSE=$(curl -s -u "$USERNAME:$PASSWORD" -XPOST "$HOST/api/v1/admin/tsdb/snapshot")
  STATUS=$(echo "$RESPONSE" | jq -r .status)
  SNAPSHOT_NAME=$(echo "$RESPONSE" | jq -r .data.name)
  if [ "$STATUS" == "success" ]; then
    # Copier le snapshot dans le dossier de destination
    DATE=$(date +'%d_%m_%Y-%H_%M_%S')
    TARGET_DIR="$SNAPSHOT_DIR/snapshot-$DATE"
    SNAPSHOT_SOURCE_DIR="$PROMETHEUS_DATA_DIR/snapshots/$SNAPSHOT_NAME"
    cp -r "$SNAPSHOT_SOURCE_DIR" "$TARGET_DIR"
    
    # Compresser le snapshot
    tar -czf "$TARGET_DIR.tar.gz" -C "$SNAPSHOT_DIR" "snapshot-$DATE"
    rm -rf "$TARGET_DIR"

    # Vérifier si la compression a réussi
    if [ -f "$TARGET_DIR.tar.gz" ]; then
      echo "$(date +'%Y-%m-%d %H:%M:%S') - Snapshot réussi: $TARGET_DIR.tar.gz" >> "$LOG_FILE"
      echo "Snapshot réussi: $TARGET_DIR.tar.gz"
      return 0
    else
      echo "$(date +'%Y-%m-%d %H:%M:%S') - Échec de la compression du snapshot: $SNAPSHOT_NAME" >> "$LOG_FILE"
      echo "Échec de la compression du snapshot: $SNAPSHOT_NAME"
      return 1
    fi
  else
    echo "$(date +'%Y-%m-%d %H:%M:%S') - Échec du snapshot: $RESPONSE" >> "$LOG_FILE"
    echo "Échec du snapshot: $RESPONSE"
    return 1
  fi
}

# Fonction pour restaurer un snapshot
restore_snapshot() {
  SNAPSHOT_FILE=$1
  if [ -z "$SNAPSHOT_FILE" ]; then
    echo "Nom du fichier de snapshot à restaurer non fourni."
    exit 1
  fi

  if [ ! -f "$SNAPSHOT_DIR/$SNAPSHOT_FILE" ]; then
    echo "Fichier de snapshot non trouvé: $SNAPSHOT_DIR/$SNAPSHOT_FILE"
    exit 1
  fi

  # Arrêter Prometheus
  docker stop prometheus

  # Supprimer le contenu actuel du répertoire de données de Prometheus
  rm -rf "$PROMETHEUS_DATA_DIR"/*

  # Extraire le snapshot
  tar -xzf "$SNAPSHOT_DIR/$SNAPSHOT_FILE" -C "$SNAPSHOT_DIR"
  SNAPSHOT_NAME=$(basename "$SNAPSHOT_FILE" .tar.gz)
  SNAPSHOT_SOURCE_DIR="$SNAPSHOT_DIR/$SNAPSHOT_NAME"
  
  # Copier le snapshot restauré dans le répertoire de données de Prometheus
  cp -r "$SNAPSHOT_SOURCE_DIR"/* "$PROMETHEUS_DATA_DIR/"
  
  # Nettoyer les fichiers extraits
  rm -rf "$SNAPSHOT_SOURCE_DIR"

  # Modifier les permissions pour que Prometheus puisse accéder aux fichiers
  chown -R 65534:65534 "$PROMETHEUS_DATA_DIR"
  
  # Redémarrer Prometheus
  docker start prometheus
  
  echo "$(date +'%Y-%m-%d %H:%M:%S') - Snapshot restauré: $SNAPSHOT_FILE" >> "$LOG_FILE"
  echo "Snapshot restauré: $SNAPSHOT_FILE"
}

# Vérifier si un argument est fourni pour la restauration
if [ "$1" == "restore" ]; then
  restore_snapshot "$2"
else
  # Essayer de réaliser le snapshot jusqu'à ce que ça réussisse
  while ! take_snapshot; do
    echo "Nouvelle tentative dans 24 heures..."
    sleep 24h
  done
fi


```

Rendez le script exécutable :
```bash
sudo chmod +x /usr/local/bin/prometheus_snapshot.sh
```
## **Étape 2 : Créer le fichier de configuration**
Créez le fichier de configuration /etc/prometheus-snapshot.conf :
```bash
sudo nano /etc/prometheus-snapshot.conf
```
Ajoutez le contenu suivant en fonction de vos configuration :
```bash
# /etc/prometheus-snapshot.conf
HOST=http://localhost:9090
USERNAME=admin
PASSWORD=admin
SNAPSHOT_DIR=/var/lib/prometheus-snapshot
LOG_FILE=/var/log/prometheus-snapshot.log
PROMETHEUS_DATA_DIR=/var/lib/docker/volumes/local-monitoring_prometheus_data/_data
```
## **Étape 3 : Créer le service systemd**
Créez le fichier de service systemd /etc/systemd/system/prometheus-snapshot.service :
```bash
sudo nano /etc/systemd/system/prometheus-snapshot.service
```
Ajoutez le contenu suivant :
```bash

[Unit]
Description=Snapshot Prometheus Data
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/prometheus_snapshot.sh

[Install]
WantedBy=multi-user.target

```
## **Étape 4 : Créer le timer systemd**
Créez le fichier timer systemd /etc/systemd/system/prometheus-snapshot.timer :
```bash
sudo nano /etc/systemd/system/prometheus-snapshot.timer
```
Ajoutez le contenu suivant :
```bash
[Unit]
Description=Run Prometheus Snapshot every Monday at 13:00

[Timer]
OnCalendar=Mon *-*-* 13:00:00
Persistent=true

[Install]
WantedBy=timers.target

```
## **Étape 5 : Activer et démarrer le service**
Pour activer et démarrer le timer, exécutez les commandes suivantes :
```bash
sudo systemctl daemon-reload

sudo systemctl enable prometheus-snapshot.timer

sudo systemctl start prometheus-snapshot.timer

sudo systemctl status prometheus-snapshot.timer
```
## **Conclusion**
Vous avez maintenant configuré un service qui effectue des snapshots des données de Prometheus chaque lundi à 13h00 et réessaie toutes les 24 heures en cas d'échec. Les paramètres de connexion et les chemins de dossier sont définis dans un fichier de configuration séparé pour plus de flexibilité et de sécurité.
