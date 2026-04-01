this is an example of read me.
# -----------------------------------------------------------------------------
# ÉTAPE 6 — Dépôt des CSV sur le NAS
# -----------------------------------------------------------------------------
step6_push_to_nas() {
    log "INFO" "=== ÉTAPE 6 : Copie locale des CSV vers $NAS_TODO ==="
    [ -z "$DQE_LOCAL_OUT" ] && die "Étape 6 : répertoire CSV local non défini (étape 5 non exécutée ?)"
 
    # Le script tourne sur la machine NAS : simple copie locale
    local count
    count=$(ls "$DQE_LOCAL_OUT"/*.csv 2>/dev/null | wc -l)
    [ "$count" -eq 0 ] && die "Étape 6 : aucun CSV trouvé dans $DQE_LOCAL_OUT"
 
    cp "$DQE_LOCAL_OUT"/*.csv "$NAS_TODO/" >> "$LOG_FILE" 2>&1 \
        || die "Échec copie locale étape 6 vers $NAS_TODO"
 
    log "INFO" "$count fichier(s) copié(s) dans $NAS_TODO."
}
 
# -----------------------------------------------------------------------------
