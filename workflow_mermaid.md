flowchart TB
    subgraph SITE["SITE KANDI.JOBS"]
        T1["Typeform T1 - Nom, Prenom, Mail, Tel"]
        T1_5["Typeform T1.5 - Etapes informatives"]
        T2["Typeform T2 - CV + Paiement + Infos campagne"]
        
        T1 --> T1_5
        T1_5 --> T2
    end

    subgraph ZAP_T1["ZAP 314202131 - T1 vers BREVO"]
        Z1_TRIGGER["New Entry Typeform T1"]
        Z1_FIND["Find Contact Brevo"]
        Z1_JS["Run JavaScript"]
        Z1_CHECK{Contact existe}
        Z1_ADD["Add Contact - Liste T1"]
        Z1_SKIP["Skip - deja existant"]
        
        Z1_TRIGGER --> Z1_FIND --> Z1_JS --> Z1_CHECK
        Z1_CHECK -->|Non| Z1_ADD
        Z1_CHECK -->|Oui| Z1_SKIP
    end

    subgraph ZAP_T15["ZAP 324970867 - T1.5 vers SMS"]
        Z15_TRIGGER["New Entry Typeform T1.5"]
        Z15_FILTER["Filter - Besoin urgent"]
        Z15_URL["Shorten URL"]
        Z15_ADD["Add Contact - Liste SMS"]
        
        Z15_TRIGGER --> Z15_FILTER --> Z15_URL --> Z15_ADD
    end

    subgraph BREVO_AUTO_SMS["BREVO SCENARIO - SMS RELANCE"]
        direction TB
        LIST_SMS[("Liste: SMS Relance")]
        SMS_START["Entree dans scenario SMS"]
        SMS_SEND["Envoi SMS Relance"]
        
        LIST_SMS --> SMS_START --> SMS_SEND
    end

    subgraph ZAP_T2["ZAP 331776370 - T2 vers BREVO"]
        Z2_TRIGGER["New Entry Typeform T2"]
        Z2_FILTER["Filter - exclut etranger"]
        Z2_FIND["Find Contact"]
        Z2_JS1["JavaScript 1 - prep data"]
        Z2_JS2["JavaScript 2 - calculs"]
        Z2_EMAIL["Send Email - Campagne lancee"]:::email
        Z2_UPDATE["Update Contact - Liste T2 + Date Job Search"]
        Z2_WEBHOOK["Webhook vers Zap Analytics"]
        
        Z2_TRIGGER --> Z2_FILTER
        Z2_FILTER -->|Etranger| Z2_STOP["Stop - pas de campagne"]
        Z2_FILTER -->|France| Z2_FIND --> Z2_JS1 --> Z2_JS2 --> Z2_EMAIL --> Z2_UPDATE --> Z2_WEBHOOK
    end

    subgraph BREVO_AUTO_T1["BREVO AUTOMATISATION - T1 NOT CLIENTS"]
        direction TB
        LIST_T1[("Liste: T1 - NOT CLIENTS - 1st SEQUENCE")]
        AUTO_START["Entree dans automatisation"]
        DELAY_2H["Delay 2h"]
        CHECK_T2{User membre liste T2 Clients}
        EXIT_T1["SORTIE"]:::exit
        R1["Relance 1 D+0 - CV manquant"]:::email
        R2["Relance 2 D+2 - Besoin de ton CV"]:::email
        R3["Relance 3 D+3 - Proposition"]:::email
        
        LIST_T1 --> AUTO_START
        AUTO_START --> DELAY_2H
        DELAY_2H --> CHECK_T2
        CHECK_T2 -->|Oui| EXIT_T1
        CHECK_T2 -->|Non| R1
        R1 -->|Delai 2j| R2
        R2 -->|Delai 1j| R3
    end

    subgraph BREVO_AUTO_T2["BREVO AUTOMATISATION - POST T2 CARE"]
        direction TB
        LIST_T2[("Liste: T2 - CLIENTS")]
        CARE_START["Entree dans automatisation CARE"]
        CARE_GUIDE["D+0 Guide Comportement"]:::email
        CARE_ENTRETIENS["D+2 Guide Entretiens"]:::email
        
        LIST_T2 --> CARE_START
        CARE_START --> CARE_GUIDE
        CARE_GUIDE -->|Delai 2j| CARE_ENTRETIENS
    end

    subgraph ZAP_ANALYTICS["ZAP 334565159 - TRIGGER MAIL ANALYTICS"]
        ZA_HOOK["Webhook depuis T2"]
        ZA_DELAY["Delay 5 jours"]
        ZA_JS["JavaScript - Appel API Analytics"]
        ZA_PATH{Reponse API}
        
        PA_COND["Path A - Stats disponibles"]
        PA_EMAIL1["Email Stats 1 - J+5"]:::email
        PA_DELAY["Delay 4 jours"]
        PA_EMAIL2["Email Stats 2 - J+9"]:::email
        
        PB_COND["Path B - Stats pas dispo"]
        PB_DELAY1["Delay 3 jours - retry"]
        PB_EMAIL1["Email Stats 1 - J+8"]:::email
        PB_DELAY2["Delay 4 jours"]
        PB_EMAIL2["Email Stats 2 - J+12"]:::email
        
        PC_COND["Path C - Erreur API"]
        PC_ERROR["Gestion erreur"]
        
        ZA_HOOK --> ZA_DELAY --> ZA_JS --> ZA_PATH
        ZA_PATH --> PA_COND --> PA_EMAIL1 --> PA_DELAY --> PA_EMAIL2
        ZA_PATH --> PB_COND --> PB_DELAY1 --> PB_EMAIL1 --> PB_DELAY2 --> PB_EMAIL2
        ZA_PATH --> PC_COND --> PC_ERROR
    end

    subgraph BREVO_AUTO_FEEDBACK["BREVO AUTOMATISATION - FEEDBACK"]
        direction TB
        FB_TRIGGER["Trigger: JOB_SEARCH_TODAY - calcul date Brevo"]
        FB_DELAY["Delay 8 jours"]
        FB_EMAIL["Email demande feedback"]:::email
        FB_FORM["Lien Typeform feedback"]
        
        FB_TRIGGER --> FB_DELAY --> FB_EMAIL --> FB_FORM
    end

    %% CONNEXIONS PRINCIPALES
    T1 -->|Trigger| Z1_TRIGGER
    T1_5 -->|Trigger| Z15_TRIGGER
    T2 -->|Trigger| Z2_TRIGGER
    
    %% T1 vers liste et automatisation
    Z1_ADD --> LIST_T1
    
    %% T1.5 vers liste SMS et scenario Brevo
    Z15_ADD --> LIST_SMS
    
    %% T2 vers liste clients et automatisation CARE
    Z2_UPDATE --> LIST_T2
    
    %% T2 update Date Job Search qui trigger Feedback via custom attribute Brevo
    Z2_UPDATE -.->|Update Date Job Search| FB_TRIGGER
    
    %% T2 declenche aussi le Zap Analytics avec delay 5j
    Z2_WEBHOOK --> ZA_HOOK
    
    %% Les relances T1 redirigent vers T2
    R1 -.->|CTA vers T2| T2
    R2 -.->|CTA vers T2| T2
    R3 -.->|CTA vers T2| T2

    %% STYLES
    classDef email fill:#1a1a1a,stroke:#333,color:#fff
    classDef zapier fill:#ff9800,stroke:#e65100,color:#000
    classDef brevo fill:#4caf50,stroke:#2e7d32,color:#fff
    classDef typeform fill:#9c27b0,stroke:#6a1b9a,color:#fff
    classDef exit fill:#f44336,stroke:#c62828,color:#fff
    
    class ZAP_T1,ZAP_T15,ZAP_T2,ZAP_ANALYTICS zapier
    class BREVO_AUTO_T1,BREVO_AUTO_T2,BREVO_AUTO_FEEDBACK,BREVO_AUTO_SMS brevo
    class SITE typeform
