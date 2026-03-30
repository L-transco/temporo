this is an example of read me.

#!/bin/bash
# =============================================================================
# MISTRAX - Automatisation workflow DQE
# =============================================================================
# Usage : ./mistrax_workflow.sh
# Logs  : /var/log/mistrax_workflow.log (ou chemin ci-dessous)
# =============================================================================

# -----------------------------------------------------------------------------
# CONFIGURATION — à adapter avant la 1ère exécution
# -----------------------------------------------------------------------------

# Chemins
ODI_HOME="/chemin/vers/odi"                        # ex: /opt/oracle/odi/agent
IN_DIR="/IN/TEST/XXXX"                             # répertoire d'arrivée source
OUT_DIR="/OUT/XXXXX/XXXX/XXXX"                     # répertoire de sortie à zipper
NAS_TODO="/IN/staging/dataintex/brokers/dqe/todo"  # dépôt NAS étape 6
TRVO_DIR="/IN/staging/dataintegrator/PROSPECT/TRV0/todo"
TRVI_DIR="/IN/staging/datainteX/brokers/prospect/todo"

# DQE (974)
DQE_HOST="ip-ou-hostname-dqe"
DQE_USER="user_dqe"
DQE_REMOTE_IN="/rxnvp/in"
DQE_REMOTE_OUT="/rxnvp/out"
# Décommente la ligne adaptée à ta situation :
# SSH_KEY="$HOME/.ssh/id_rsa"   # option clé SSH
# SFTP_PASS="motdepasse"        # option mot de passe (nécessite sshpass)

# NAS
NAS_HOST="ip-ou-hostname-nas"
NAS_USER="user_nas"
# SSH_KEY_NAS="$HOME/.ssh/id_rsa"

# ODI
STARTLP="$ODI_HOME/bin/startloadplan.sh"

# Loadplans (vérifie les noms exacts avec espaces/underscores)
LP_OUT_PLAN="BMP MKT ODS MISTRAX OUT"
LP_OUT_CTX="CTX_BMP_MKX"
LP_OUT_AGENT="Agent_BMX"

LP_IN_PLAN="BMP MKT ODS MISTRAX IN"
LP_IN_CTX="CTX_BMP_MKX"
LP_IN_AGENT="Agent_BMX"

LP_PEGA_PLAN="BMP MKT PRX Chargmt Alim PUB PEGA_AXX"
LP_PEGA_CTX="CTX_BMP_MKX"
LP_PEGA_AGENT="Agent_VEROX"

LP_VRN_PLAN="VRN PRX Chargmt_TRVO"
LP_VRN_CTX="CTX_PROX"
LP_VRN_AGENT="Agent_VEROX"

# Polling et timing
POLL_INTERVAL=120          # secondes entre chaque vérification (étape 1)
POLL_TIMEOUT=7200          # timeout global étape 1 : 2h
DQE_WAIT_HOUR_1=15         # 1ère fenêtre d'attente : 15h
DQE_WAIT_MIN_1=10
DQE_WAIT_HOUR_2=3          # 2ème fenêtre : 3h du matin
DQE_WAIT_MIN_2=10
DQE_POLL_INTERVAL=300      # vérification toutes les 5 min après l'heure cible
DQE_POLL_MAX_ATTEMPTS=24   # 24 × 5 min = 2h max d'attente après l'heure cible

# Logs
LOG_FILE="/var/log/mistrax_workflow.log"

# -----------------------------------------------------------------------------
# FONCTIONS UTILITAIRES
# -----------------------------------------------------------------------------

log() {
    local level="$1"; shift
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$level] $*" | tee -a "$LOG_FILE"
}

die() {
    log "ERREUR" "$*"
    exit 1
}

confirm() {
    # Validation manuelle optionnelle (mode semi-auto)
    local msg="$1"
    log "INFO" ">>> VALIDATION REQUISE : $msg"
    read -r -p "    Appuie sur ENTRÉE pour continuer, ou tape 'non' pour arrêter : " rep
    [[ "$rep" == "non" ]] && die "Arrêt demandé par l'opérateur à l'étape : $msg"
}

# -----------------------------------------------------------------------------
# ÉTAPE 1 — Vérification présence fichier source
# -----------------------------------------------------------------------------
step1_wait_source_file() {
    log "INFO" "=== ÉTAPE 1 : Attente fichier dans $IN_DIR ==="
    local elapsed=0

    while [ $elapsed -lt $POLL_TIMEOUT ]; do
        local count
        count=$(ls "$IN_DIR" 2>/dev/null | wc -l)
        if [ "$count" -gt 0 ]; then
            log "INFO" "Fichier(s) détecté(s) dans $IN_DIR ($count fichier(s)). On continue."
            ls "$IN_DIR" | tee -a "$LOG_FILE"
            return 0
        fi
        log "INFO" "Pas encore de fichier dans $IN_DIR — attente ${POLL_INTERVAL}s (${elapsed}s écoulées)"
        sleep "$POLL_INTERVAL"
        elapsed=$((elapsed + POLL_INTERVAL))
    done

    die "Timeout étape 1 : aucun fichier détecté dans $IN_DIR après ${POLL_TIMEOUT}s"
}

# -----------------------------------------------------------------------------
# ÉTAPE 2 — Lancement loadplan ODI OUT
# -----------------------------------------------------------------------------
step2_run_odi_out() {
    log "INFO" "=== ÉTAPE 2 : Loadplan ODI OUT ==="
    confirm "Lancement du plan '$LP_OUT_PLAN' (CTX=$LP_OUT_CTX / INSTANCE=$LP_OUT_AGENT)"

    "$STARTLP" "-INSTANCE=$LP_OUT_AGENT" "$LP_OUT_PLAN" "$LP_OUT_CTX" \
        >> "$LOG_FILE" 2>&1 \
        || die "Échec loadplan étape 2 : $LP_OUT_PLAN"

    log "INFO" "Loadplan OUT terminé avec succès."
}

# -----------------------------------------------------------------------------
# ÉTAPE 3 — Zip des fichiers de sortie
# -----------------------------------------------------------------------------
step3_zip_files() {
    log "INFO" "=== ÉTAPE 3 : Zip des fichiers dans $OUT_DIR ==="

    cd "$OUT_DIR" || die "Impossible d'accéder à $OUT_DIR"

    local count
    count=$(ls 2>/dev/null | wc -l)
    [ "$count" -eq 0 ] && die "Aucun fichier à zipper dans $OUT_DIR"

    # Nom du zip horodaté
    local zipname
    zipname="mistrax_dqe_$(date '+%Y%m%d_%H%M%S').zip"

    log "INFO" "Création de $zipname ($count fichier(s))"
    zip "$zipname" ./* >> "$LOG_FILE" 2>&1 \
        || die "Échec zip étape 3"

    log "INFO" "Zip créé : $OUT_DIR/$zipname"
    echo "$OUT_DIR/$zipname"  # retourne le chemin pour l'étape 4
}

# -----------------------------------------------------------------------------
# ÉTAPE 4 — Dépôt SFTP vers DQE (974)
# -----------------------------------------------------------------------------
step4_sftp_to_dqe() {
    local zipfile="$1"
    log "INFO" "=== ÉTAPE 4 : Dépôt SFTP vers DQE $DQE_HOST:$DQE_REMOTE_IN ==="

    # --- Option A : clé SSH (recommandé) ---
    if [ -n "${SSH_KEY:-}" ]; then
        sftp -i "$SSH_KEY" -b - "$DQE_USER@$DQE_HOST" << EOF >> "$LOG_FILE" 2>&1
cd $DQE_REMOTE_IN
put $zipfile
bye
EOF

    # --- Option B : mot de passe via sshpass ---
    elif [ -n "${SFTP_PASS:-}" ]; then
        sshpass -p "$SFTP_PASS" sftp -b - "$DQE_USER@$DQE_HOST" << EOF >> "$LOG_FILE" 2>&1
cd $DQE_REMOTE_IN
put $zipfile
bye
EOF

    else
        die "Étape 4 : aucune méthode d'authentification SFTP configurée (SSH_KEY ou SFTP_PASS)"
    fi

    [ $? -ne 0 ] && die "Échec dépôt SFTP étape 4"
    log "INFO" "Fichier déposé sur DQE avec succès."
}

# -----------------------------------------------------------------------------
# ÉTAPE 5 — Attente traitement DQE puis récupération des CSV
# -----------------------------------------------------------------------------
step5_wait_and_get_dqe_output() {
    log "INFO" "=== ÉTAPE 5 : Attente traitement DQE ==="

    # Calcule le prochain créneau cible (15h10 ou 3h10)
    local now_h now_m target_h target_m wait_secs now_ts target_ts
    now_h=$(date '+%H')
    now_m=$(date '+%M')
    now_ts=$(date '+%s')

    # On choisit la prochaine fenêtre non encore passée
    local t1_ts t2_ts
    t1_ts=$(date -d "today ${DQE_WAIT_HOUR_1}:${DQE_WAIT_MIN_1}:00" '+%s' 2>/dev/null \
         || date -v${DQE_WAIT_HOUR_1}H -v${DQE_WAIT_MIN_1}M -v0S '+%s')  # macOS fallback
    t2_ts=$(date -d "today ${DQE_WAIT_HOUR_2}:${DQE_WAIT_MIN_2}:00" '+%s' 2>/dev/null \
         || date -v${DQE_WAIT_HOUR_2}H -v${DQE_WAIT_MIN_2}M -v0S '+%s')

    # Si 3h est déjà passée, on la reporte au lendemain
    [ $t2_ts -lt $now_ts ] && t2_ts=$((t2_ts + 86400))
    [ $t1_ts -lt $now_ts ] && t1_ts=$((t1_ts + 86400))

    # On prend la fenêtre la plus proche
    if [ $t1_ts -le $t2_ts ]; then
        target_ts=$t1_ts
        target_h=$DQE_WAIT_HOUR_1; target_m=$DQE_WAIT_MIN_1
    else
        target_ts=$t2_ts
        target_h=$DQE_WAIT_HOUR_2; target_m=$DQE_WAIT_MIN_2
    fi

    wait_secs=$((target_ts - now_ts))
    if [ $wait_secs -gt 0 ]; then
        log "INFO" "Sleep jusqu'à ${target_h}h${target_m} (${wait_secs}s)"
        sleep "$wait_secs"
    else
        log "INFO" "Heure cible déjà passée, on vérifie directement."
    fi

    # Polling post-heure-cible : on attend l'apparition des CSV
    log "INFO" "Polling sur $DQE_REMOTE_OUT (max $DQE_POLL_MAX_ATTEMPTS tentatives)"
    local attempt=0
    local remote_count

    while [ $attempt -lt $DQE_POLL_MAX_ATTEMPTS ]; do
        attempt=$((attempt + 1))

        # Compte les CSV présents côté DQE
        if [ -n "${SSH_KEY:-}" ]; then
            remote_count=$(ssh -i "$SSH_KEY" "$DQE_USER@$DQE_HOST" \
                "ls $DQE_REMOTE_OUT/*.csv 2>/dev/null | wc -l")
        else
            remote_count=$(sshpass -p "$SFTP_PASS" ssh "$DQE_USER@$DQE_HOST" \
                "ls $DQE_REMOTE_OUT/*.csv 2>/dev/null | wc -l")
        fi

        if [ "${remote_count:-0}" -gt 0 ]; then
            log "INFO" "$remote_count fichier(s) CSV détecté(s) dans $DQE_REMOTE_OUT — récupération..."
            _download_dqe_csv
            return 0
        fi

        log "INFO" "Tentative $attempt/$DQE_POLL_MAX_ATTEMPTS : aucun CSV encore. Attente ${DQE_POLL_INTERVAL}s"
        sleep "$DQE_POLL_INTERVAL"
    done

    die "Timeout étape 5 : aucun CSV apparu sur DQE après $DQE_POLL_MAX_ATTEMPTS tentatives"
}

_download_dqe_csv() {
    # Télécharge tous les CSV de /rxnvp/out vers un répertoire local temporaire
    DQE_LOCAL_OUT="/tmp/dqe_out_$(date '+%Y%m%d_%H%M%S')"
    mkdir -p "$DQE_LOCAL_OUT"

    if [ -n "${SSH_KEY:-}" ]; then
        sftp -i "$SSH_KEY" -b - "$DQE_USER@$DQE_HOST" << EOF >> "$LOG_FILE" 2>&1
lcd $DQE_LOCAL_OUT
cd $DQE_REMOTE_OUT
mget *.csv
bye
EOF
    else
        sshpass -p "$SFTP_PASS" sftp -b - "$DQE_USER@$DQE_HOST" << EOF >> "$LOG_FILE" 2>&1
lcd $DQE_LOCAL_OUT
cd $DQE_REMOTE_OUT
mget *.csv
bye
EOF
    fi

    [ $? -ne 0 ] && die "Échec téléchargement CSV DQE étape 5"
    log "INFO" "CSV récupérés dans $DQE_LOCAL_OUT"
}

# -----------------------------------------------------------------------------
# ÉTAPE 6 — Dépôt des CSV sur le NAS
# -----------------------------------------------------------------------------
step6_push_to_nas() {
    log "INFO" "=== ÉTAPE 6 : Dépôt sur NAS $NAS_HOST:$NAS_TODO ==="
    [ -z "$DQE_LOCAL_OUT" ] && die "Étape 6 : répertoire CSV local non défini (étape 5 non exécutée ?)"

    if [ -n "${SSH_KEY_NAS:-}" ]; then
        sftp -i "$SSH_KEY_NAS" -b - "$NAS_USER@$NAS_HOST" << EOF >> "$LOG_FILE" 2>&1
cd $NAS_TODO
mput $DQE_LOCAL_OUT/*.csv
bye
EOF
    else
        # Sans clé : ajuste selon ton infra NAS (montage réseau, scp, etc.)
        scp "$DQE_LOCAL_OUT"/*.csv "$NAS_USER@$NAS_HOST:$NAS_TODO/" \
            >> "$LOG_FILE" 2>&1
    fi

    [ $? -ne 0 ] && die "Échec dépôt NAS étape 6"
    log "INFO" "Fichiers déposés sur NAS."
}

# -----------------------------------------------------------------------------
# ÉTAPE 7 — Loadplan ODI IN
# -----------------------------------------------------------------------------
step7_run_odi_in() {
    log "INFO" "=== ÉTAPE 7 : Loadplan ODI IN ==="
    confirm "Lancement du plan '$LP_IN_PLAN' (CTX=$LP_IN_CTX / INSTANCE=$LP_IN_AGENT)"

    "$STARTLP" "-INSTANCE=$LP_IN_AGENT" "$LP_IN_PLAN" "$LP_IN_CTX" \
        >> "$LOG_FILE" 2>&1 \
        || die "Échec loadplan étape 7 : $LP_IN_PLAN"

    log "INFO" "Loadplan IN terminé — vérification présence TRVO et TRVI..."

    # Vérification que les deux fichiers ont bien été générés
    local trvo_count trvi_count
    trvo_count=$(ls "$TRVO_DIR"/*.csv 2>/dev/null | wc -l)
    trvi_count=$(ls "$TRVI_DIR"/*.csv 2>/dev/null | wc -l)

    log "INFO" "TRVO : $trvo_count fichier(s) dans $TRVO_DIR"
    log "INFO" "TRVI : $trvi_count fichier(s) dans $TRVI_DIR"

    [ "$trvo_count" -eq 0 ] && die "Étape 7 : aucun fichier TRVO généré dans $TRVO_DIR"
    [ "$trvi_count" -eq 0 ] && die "Étape 7 : aucun fichier TRVI généré dans $TRVI_DIR"
}

# -----------------------------------------------------------------------------
# ÉTAPE 8 — Suppression ligne d'entête du fichier TRVO
# -----------------------------------------------------------------------------
step8_remove_header_trvo() {
    log "INFO" "=== ÉTAPE 8 : Suppression entête TRVO ==="

    cd "$TRVO_DIR" || die "Impossible d'accéder à $TRVO_DIR"

    local file
    for file in MISTRAX_TRVO_PROSPECT_*.csv; do
        [ -f "$file" ] || die "Étape 8 : aucun fichier TRVO trouvé (pattern: MISTRAX_TRVO_PROSPECT_*.csv)"
        log "INFO" "sed -i '1d' sur $file"
        sed -i '1d' "$file" >> "$LOG_FILE" 2>&1 \
            || die "Échec sed étape 8 sur $file"
    done

    log "INFO" "Entête(s) supprimée(s)."
}

# -----------------------------------------------------------------------------
# ÉTAPE 9 — Loadplan TRVI (PEGA_AXX)
# -----------------------------------------------------------------------------
step9_run_odi_pega() {
    log "INFO" "=== ÉTAPE 9 : Loadplan PEGA_AXX (TRVI) ==="
    confirm "Lancement du plan '$LP_PEGA_PLAN' (CTX=$LP_PEGA_CTX / INSTANCE=$LP_PEGA_AGENT)"

    "$STARTLP" "-INSTANCE=$LP_PEGA_AGENT" "$LP_PEGA_PLAN" "$LP_PEGA_CTX" \
        >> "$LOG_FILE" 2>&1 \
        || die "Échec loadplan étape 9 : $LP_PEGA_PLAN"

    log "INFO" "Loadplan PEGA_AXX terminé."
}

# -----------------------------------------------------------------------------
# ÉTAPE 10 — Loadplan TRVO (VRN PRX)
# -----------------------------------------------------------------------------
step10_run_odi_vrn() {
    log "INFO" "=== ÉTAPE 10 : Loadplan VRN PRX (TRVO) ==="
    confirm "Lancement du plan '$LP_VRN_PLAN' (CTX=$LP_VRN_CTX / INSTANCE=$LP_VRN_AGENT)"

    "$STARTLP" "-INSTANCE=$LP_VRN_AGENT" "$LP_VRN_PLAN" "$LP_VRN_CTX" \
        >> "$LOG_FILE" 2>&1 \
        || die "Échec loadplan étape 10 : $LP_VRN_PLAN"

    log "INFO" "Loadplan VRN PRX terminé."
}

# -----------------------------------------------------------------------------
# MAIN
# -----------------------------------------------------------------------------
main() {
    log "INFO" "========================================================"
    log "INFO" "Démarrage workflow MISTRAX DQE — $(date)"
    log "INFO" "========================================================"

    step1_wait_source_file

    step2_run_odi_out

    local zipfile
    zipfile=$(step3_zip_files)
    log "INFO" "Zip produit : $zipfile"

    step4_sftp_to_dqe "$zipfile"

    step5_wait_and_get_dqe_output

    step6_push_to_nas

    step7_run_odi_in

    step8_remove_header_trvo

    step9_run_odi_pega

    step10_run_odi_vrn

    log "INFO" "========================================================"
    log "INFO" "Workflow MISTRAX DQE terminé avec succès — $(date)"
    log "INFO" "========================================================"
}

main "$@"

Étape 1 — Vérification fichier
Polling bash : attente présence dans /IN/TEST/XXXX
①
Étape 2 — Loadplan ODI OUT
startloadplan.sh BMP MKT ODS MISTRAX OUT (CTX BMP_MKX / Agent_BMX)
②
Étape 3 — Zip des fichiers (Linux)
zip fichier_DQX.zip fichier_DQX* dans /OUT/XXXXX/…
③
Étape 4 — Dépôt SFTP vers DQE (974)
sftp → /rxnvp/in (remplace FileZilla)
④
Étape 5 — Attente traitement DQE + récupération
Polling sur /rxnvp/out jusqu'à apparition des fichiers
⑤
Étape 6 — Dépôt sur NAS
sftp → /IN/staging/dataintex/brokers/dqe/todo
Étape 7 — Loadplan ODI IN
startloadplan.sh BMP MKT ODS MISTRAX IN → génère TRVO + TRVI
⑦
Étape 8 — Suppression entête TRVO (Linux)
sed -i '1d' MISTRAX_TRVO_PROSPECT_*.csv
⑧
Étape 9 — Loadplan TRVI (PEGA_AXX)
startloadplan.sh BMP MKT PRX Chargmt Alim PUB PEGA_AXX (CTX BMP_MKX / Agent_VEROX)
⑨
Étape 10 — Loadplan TRVO (VRN PRX)
startloadplan.sh VRN PRX Chargmt_TRVO (CTX_PROX / Agent_VEROX)
⑩

