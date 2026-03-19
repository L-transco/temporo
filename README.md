this is an example of read me.


CREATE TABLE FLUX_CONFIG (
    NOM_SCHEMA   VARCHAR2(124) NOT NULL,
    NOM_TABLE    VARCHAR2(124) NOT NULL,
    LIBELLE_FLUX VARCHAR2(200),
    REGEX_NOM    VARCHAR2(500), -- pattern pour extraire la séquence de NOM_FICHIER
    POSITION_SEQ NUMBER DEFAULT 1,
    ACTIF        CHAR(1) DEFAULT 'O',
    COMMENTAIRE  VARCHAR2(500),
    CONSTRAINT PK_FLUX_CONFIG PRIMARY KEY (NOM_SCHEMA, NOM_TABLE)
);

CREATE TABLE FLUX_ANOMALIE_SEQ (
    ID_ANOMALIE    NUMBER GENERATED ALWAYS AS IDENTITY,
    NOM_SCHEMA     VARCHAR2(124) NOT NULL,
    NOM_TABLE      VARCHAR2(124) NOT NULL,
    TEC_CREATION   NUMBER        NOT NULL,  -- identifiant d'exécution
    SEQ_MANQUANTE  NUMBER        NOT NULL,  -- numéro de séquence absent
    NOM_PRECEDENT  VARCHAR2(500),           -- NOM_FICHIER juste avant le trou
    NOM_SUIVANT    VARCHAR2(500),           -- NOM_FICHIER juste après le trou
    DATE_DETECTION DATE DEFAULT SYSDATE,
    STATUT         VARCHAR2(50) DEFAULT 'OUVERT',
    COMMENTAIRE    VARCHAR2(500),
    CONSTRAINT PK_FLUX_ANOMALIE PRIMARY KEY (ID_ANOMALIE)
);

/*
Comment alimenter FLUX_CONFIG concrètement
Pour chaque couple NOM_SCHEMA / NOM_TABLE distinct dans ta table de log, tu fais l'analyse du pattern une fois :
sql-- D'abord, liste tous les flux distincts présents dans ta table
SELECT DISTINCT NOM_SCHEMA, NOM_TABLE
FROM TABLE_LOG
ORDER BY NOM_SCHEMA, NOM_TABLE;

Pour chaque ligne retournée, tu regardes quelques exemples de NOM_FICHIER :
sql-- Voir des exemples de fichiers pour un flux donné
SELECT DISTINCT NOM_FICHIER
FROM TABLE_LOG
WHERE NOM_SCHEMA = 'MON_SCHEMA'
AND NOM_TABLE   = 'MA_TABLE'
AND ROWNUM <= 10;

Tu identifies visuellement le numéro de séquence, tu construis ta regex, et tu insères :
sql INSERT INTO FLUX_CONFIG (NOM_SCHEMA, NOM_TABLE, LIBELLE_FLUX, REGEX_NOM, POSITION_SEQ)
VALUES ('MON_SCHEMA', 'MA_TABLE', 'Mon flux métier', 'PREFIX_(\d+)_SUFFIX\.zip', 1);

Tu as maintenant une architecture complète et cohérente avec ta vraie table. La jointure entre les deux tables se fera naturellement sur NOM_SCHEMA + NOM_TABLE, et le regroupement par exécution sur TEC_CREATION. La prochaine étape sera la procédure de détection quand tu seras prêt.
*/

Structure générale de la procédure
PROCEDURE DETECT_ANOMALIES_SEQ AS
BEGIN
  -- Etape 1 : Nettoyer les anomalies déjà résolues ou re-détecter ?
  -- (choix à faire, voir plus bas)

  -- Etape 2 : Boucler sur chaque flux actif dans FLUX_CONFIG

  -- Etape 3 : Pour chaque flux, boucler sur chaque TEC_CREATION
  --           non encore analysé

  -- Etape 4 : Extraire les numéros de séquence via REGEXP_SUBSTR
  --           en utilisant REGEX_NOM + POSITION_SEQ de FLUX_CONFIG

  -- Etape 5 : Détecter les trous avec LAG()

  -- Etape 6 : Expandre les trous avec CONNECT BY

  -- Etape 7 : Insérer dans FLUX_ANOMALIE_SEQ

END;
