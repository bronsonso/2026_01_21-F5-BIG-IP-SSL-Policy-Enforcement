prod/ai_edcity_all_crt
ltm profile client-ssl prod/ai_edcity_all_crt {
    app-service none
    cert-key-chain {
        ai_edcity_all_crt_hkec.chain_0 {
            cert prod/ai_edcity_all_crt
            chain prod/hkec.chain.crt
            key prod/ai_edcity_all_crt
        }
    }
    defaults-from prod/hkec_high
    inherit-ca-certkeychain true
    inherit-certkeychain false
}

prod/edcity_all_crt
ltm profile client-ssl prod/edcity_all_crt {
    app-service none
    cert-key-chain {
        edcity_all_crt_hkec.chain_0 {
            cert prod/edcity_all_crt
            chain prod/hkec.chain.crt
            key prod/edcity_all_crt
        }
    }
    defaults-from prod/hkec_default
    inherit-ca-certkeychain true
    inherit-certkeychain false
}

prod/edcity_edb_all_crt
ltm profile client-ssl prod/edcity_edb_all_crt {
    app-service none
    cert-key-chain {
        edcity_edb_all_crt_hkec.chain_0 {
            cert prod/edcity_edb_all_crt
            chain prod/hkec.chain.crt
            key prod/edcity_edb_all_crt
        }
    }
    defaults-from prod/hkec_TLSv12-nokRSA
    inherit-ca-certkeychain true
    inherit-certkeychain false
}

prod/edcity_edb_all_crt_high
ltm profile client-ssl prod/edcity_edb_all_crt_high {
    app-service none
    cert-key-chain {
        edcity_edb_all_crt_hkec.chain_0 {
            cert prod/edcity_edb_all_crt
            chain prod/hkec.chain.crt
            key prod/edcity_edb_all_crt
        }
    }
    defaults-from prod/hkec_high
    inherit-ca-certkeychain true
    inherit-certkeychain false
}

prod/edcity_proj_all_crt
ltm profile client-ssl prod/edcity_proj_all_crt {
    app-service none
    cert-key-chain {
        edcity_proj_all_crt_hkec.chain_0 {
            cert prod/edcity_proj_all_crt
            chain prod/hkec.chain.crt
            key prod/edcity_proj_all_crt
        }
    }
    defaults-from prod/hkec_TLSv12-nokRSA
    inherit-ca-certkeychain true
    inherit-certkeychain false
}

prod/hkec_TLSv12-nokRSA
ltm profile client-ssl prod/hkec_TLSv12-nokRSA {
    app-service none
    cert-key-chain {
        default {
            cert Common/default.crt
            key Common/default.key
        }
    }
    cipher-group none
    ciphers
ECDHE-ECDSA-AES128-GCM-SHA256,ECDHE-ECDSA-AES256-GCM-SHA384,ECDHE-ECDSA-AES128-SHA
256,ECDHE-RSA-AES128-GCM-SHA256,ECDHE-RSA-AES256-GCM-SHA384,ECDHE-RSA-AES128-SHA25
6,DHE-RSA-AES128-GCM-SHA256,DHE-RSA-AES256-GCM-SHA384,DHE-RSA-AES128-SHA256,DHE-RS
A-AES256-SHA256
    defaults-from Common/clientssl
    inherit-ca-certkeychain true
    inherit-certkeychain true
    options { dont-insert-empty-fragments no-tlsv1.3 no-dtlsv1.2 no-sslv3 }
}

prod/hkec_TLSv12-nokRSA-noCBC
ltm profile client-ssl prod/hkec_TLSv12-nokRSA-noCBC {
    app-service none
    cert-key-chain {
        default {
            cert Common/default.crt
            key Common/default.key
        }
    }
    cipher-group none
    ciphers
ECDHE-ECDSA-AES128-GCM-SHA256,ECDHE-ECDSA-AES256-GCM-SHA384,ECDHE-RSA-AES128-GCM-S
HA256,ECDHE-RSA-AES256-GCM-SHA384,DHE-RSA-AES128-GCM-SHA256,DHE-RSA-AES256-GCM-SHA
384
    defaults-from Common/clientssl
    inherit-ca-certkeychain true
    inherit-certkeychain true
    options { dont-insert-empty-fragments no-tlsv1.3 no-dtlsv1.2 no-sslv3 }
}

prod/hkec_all_crt
ltm profile client-ssl prod/hkec_all_crt {
    app-service none
    cert-key-chain {
        hkec_all_crt_hkec.chain_0 {
            cert prod/hkec_all_crt
            chain prod/hkec.chain.crt
            key prod/hkec_all_crt
        }
    }
    defaults-from prod/hkec_default
    inherit-ca-certkeychain true
    inherit-certkeychain false
}

prod/hkec_all_crt_TLSv12-nokRSA
ltm profile client-ssl prod/hkec_all_crt_TLSv12-nokRSA {
    app-service none
    cert-key-chain {
        hkec_all_crt_hkec.chain_0 {
            cert prod/hkec_all_crt
            chain prod/hkec.chain.crt
            key prod/hkec_all_crt
        }
    }
    defaults-from prod/hkec_TLSv12-nokRSA
    inherit-ca-certkeychain true
    inherit-certkeychain false
}

prod/hkec_all_crt_high
ltm profile client-ssl prod/hkec_all_crt_high {
    app-service none
    cert-key-chain {
        hkec_all_crt_hkec.chain_0 {
            cert prod/hkec_all_crt
            chain prod/hkec.chain.crt
            key prod/hkec_all_crt
        }
    }
    defaults-from prod/hkec_high
    inherit-ca-certkeychain true
    inherit-certkeychain false
}

prod/hkec_all_crt_sadm
ltm profile client-ssl prod/hkec_all_crt_sadm {
    app-service none
    cert-key-chain {
        hkec_all_crt_hkec.chain_0 {
            cert prod/hkec_all_crt
            chain prod/hkec.chain.crt
            key prod/hkec_all_crt
        }
    }
    defaults-from prod/hkec_TLSv12-nokRSA-noCBC
    inherit-ca-certkeychain true
    inherit-certkeychain false
}

prod/hkec_all_crt_wmng
ltm profile client-ssl prod/hkec_all_crt_wmng {
    app-service none
    cert-key-chain {
        hkec_all_crt_hkec.chain_0 {
            cert prod/hkec_all_crt
            chain prod/hkec.chain.crt
            key prod/hkec_all_crt
        }
    }
    defaults-from prod/hkec_default
    inherit-ca-certkeychain true
    inherit-certkeychain false
}

prod/hkec_default
ltm profile client-ssl prod/hkec_default {
    app-service none
    cert-key-chain {
        default {
            cert Common/default.crt
            key Common/default.key
        }
    }
    cipher-group none
    ciphers
ECDHE-ECDSA-AES128-GCM-SHA256,ECDHE-ECDSA-AES256-GCM-SHA384,ECDHE-ECDSA-AES128-SHA
256,ECDHE-RSA-AES128-GCM-SHA256,ECDHE-RSA-AES256-GCM-SHA384,ECDHE-RSA-AES128-SHA25
6,DHE-RSA-AES128-GCM-SHA256,DHE-RSA-AES256-GCM-SHA384,DHE-RSA-AES128-SHA256,DHE-RS
A-AES256-SHA256,DHE-RSA-AES128-SHA,DHE-RSA-AES256-SHA,AES128-GCM-SHA256,AES256-GCM
-SHA384,AES128-SHA256,AES256-SHA256,AES128-SHA,AES256-SHA
    defaults-from Common/clientssl
    inherit-ca-certkeychain true
    inherit-certkeychain true
    options { dont-insert-empty-fragments no-tlsv1.3 no-dtlsv1.2 no-sslv3 }
}

prod/hkec_edb_all_crt_high
ltm profile client-ssl prod/hkec_edb_all_crt_high {
    app-service none
    cert-key-chain {
        hkec_edb_all_crt_hkec.chain_0 {
            cert prod/hkec_edb_all_crt
            chain prod/hkec.chain.crt
            key prod/hkec_edb_all_crt
        }
    }
    defaults-from prod/hkec_high
    inherit-ca-certkeychain true
    inherit-certkeychain false
}

prod/hkec_high
ltm profile client-ssl prod/hkec_high {
    app-service none
    cert-key-chain {
        default {
            cert Common/default.crt
            key Common/default.key
        }
    }
    cipher-group none
    ciphers
ECDHE-ECDSA-AES128-GCM-SHA256,ECDHE-ECDSA-AES256-GCM-SHA384,ECDHE-RSA-AES128-GCM-S
HA256,ECDHE-RSA-AES256-GCM-SHA384,DHE-RSA-AES128-GCM-SHA256,DHE-RSA-AES256-GCM-SHA
384
    defaults-from Common/clientssl
    inherit-ca-certkeychain true
    inherit-certkeychain true
    options { dont-insert-empty-fragments no-tlsv1.3 no-dtlsv1.2 no-sslv3 }
}

prod/hkec_proj_all_crt
ltm profile client-ssl prod/hkec_proj_all_crt {
    app-service none
    cert-key-chain {
        hkec_proj_all_crt_hkec.chain_0 {
            cert prod/hkec_proj_all_crt
            chain prod/hkec.chain.crt
            key prod/hkec_proj_all_crt
        }
    }
    defaults-from prod/hkec_default
    inherit-ca-certkeychain true
    inherit-certkeychain false
}

prod/hkec_star_all_crt
ltm profile client-ssl prod/hkec_star_all_crt {
    app-service none
    cert-key-chain {
        hkec_star_all_crt_hkec.chain_0 {
            cert prod/hkec_star_all_crt
            chain prod/hkec.chain.crt
            key prod/hkec_star_all_crt
        }
    }
    defaults-from prod/hkec_default
    inherit-ca-certkeychain true
    inherit-certkeychain false
}

prod/hkrc_all_crt
ltm profile client-ssl prod/hkrc_all_crt {
    app-service none
    cert-key-chain {
        hkrc_all_crt_hkec.chain_0 {
            cert prod/hkrc_all_crt
            chain prod/hkec.chain.crt
            key prod/hkrc_all_crt
        }
    }
    defaults-from prod/hkec_default
    inherit-ca-certkeychain true
    inherit-certkeychain false
}

uat/u_ai_edcity_all_crt
ltm profile client-ssl uat/u_ai_edcity_all_crt {
    app-service none
    cert-key-chain {
        u_ai_edcity_all_crt_u_hkec.chain_0 {
            cert uat/u_ai_edcity_all_crt
            chain uat/u_hkec.chain.crt
            key uat/u_ai_edcity_all_crt
        }
    }
    defaults-from uat/u_hkec_high
    inherit-ca-certkeychain true
    inherit-certkeychain false
}

uat/u_edcity_all_crt
ltm profile client-ssl uat/u_edcity_all_crt {
    app-service none
    cert-key-chain {
        u_edcity_all_crt_u_hkec.chain_0 {
            cert uat/u_edcity_all_crt
            chain uat/u_hkec.chain.crt
            key uat/u_edcity_all_crt
        }
    }
    defaults-from uat/u_hkec_default
    inherit-ca-certkeychain true
    inherit-certkeychain false
}

uat/u_hkec_TLSv12-nokRSA
ltm profile client-ssl uat/u_hkec_TLSv12-nokRSA {
    app-service none
    cert-key-chain {
        default {
            cert Common/default.crt
            key Common/default.key
        }
    }
    cipher-group none
    ciphers
ECDHE-ECDSA-AES128-GCM-SHA256,ECDHE-ECDSA-AES256-GCM-SHA384,ECDHE-ECDSA-AES128-SHA
256,ECDHE-RSA-AES128-GCM-SHA256,ECDHE-RSA-AES256-GCM-SHA384,ECDHE-RSA-AES128-SHA25
6,DHE-RSA-AES128-GCM-SHA256,DHE-RSA-AES256-GCM-SHA384,DHE-RSA-AES128-SHA256,DHE-RS
A-AES256-SHA256
    defaults-from Common/clientssl
    inherit-ca-certkeychain true
    inherit-certkeychain true
    options { dont-insert-empty-fragments no-tlsv1.3 no-dtlsv1.2 no-sslv3 }
}

uat/u_hkec_all_crt
ltm profile client-ssl uat/u_hkec_all_crt {
    app-service none
    cert-key-chain {
        u_hkec_all_crt_u_hkec.chain_0 {
            cert uat/u_hkec_all_crt
            chain uat/u_hkec.chain.crt
            key uat/u_hkec_all_crt
        }
    }
    defaults-from uat/u_hkec_default
    inherit-ca-certkeychain true
    inherit-certkeychain false
}

uat/u_hkec_all_crt_TLSv12-nokRSA
ltm profile client-ssl uat/u_hkec_all_crt_TLSv12-nokRSA {
    app-service none
    cert-key-chain {
        u_hkec_all_crt_u_hkec.chain_0 {
            cert uat/u_hkec_all_crt
            chain uat/u_hkec.chain.crt
            key uat/u_hkec_all_crt
        }
    }
    defaults-from uat/u_hkec_TLSv12-nokRSA
    inherit-ca-certkeychain true
    inherit-certkeychain false
}

uat/u_hkec_all_crt_high
ltm profile client-ssl uat/u_hkec_all_crt_high {
    app-service none
    cert-key-chain {
        u_hkec_all_crt_u_hkec.chain_0 {
            cert uat/u_hkec_all_crt
            chain uat/u_hkec.chain.crt
            key uat/u_hkec_all_crt
        }
    }
    defaults-from uat/u_hkec_high
    inherit-ca-certkeychain true
    inherit-certkeychain false
}

uat/u_hkec_default
ltm profile client-ssl uat/u_hkec_default {
    app-service none
    cert-key-chain {
        default {
            cert Common/default.crt
            key Common/default.key
        }
    }
    cipher-group none
    ciphers
ECDHE-ECDSA-AES128-GCM-SHA256,ECDHE-ECDSA-AES256-GCM-SHA384,ECDHE-ECDSA-AES128-SHA
256,ECDHE-RSA-AES128-GCM-SHA256,ECDHE-RSA-AES256-GCM-SHA384,ECDHE-RSA-AES128-SHA25
6,DHE-RSA-AES128-GCM-SHA256,DHE-RSA-AES256-GCM-SHA384,DHE-RSA-AES128-SHA256,DHE-RS
A-AES256-SHA256,DHE-RSA-AES128-SHA,DHE-RSA-AES256-SHA,AES128-GCM-SHA256,AES256-GCM
-SHA384,AES128-SHA256,AES256-SHA256,AES128-SHA,AES256-SHA
    defaults-from Common/clientssl
    inherit-ca-certkeychain true
    inherit-certkeychain true
    options { dont-insert-empty-fragments no-tlsv1.3 no-dtlsv1.2 no-sslv3 }
}

uat/u_hkec_edb_all_crt_high
ltm profile client-ssl uat/u_hkec_edb_all_crt_high {
    app-service none
    cert-key-chain {
        u_hkec_edb_all_crt_u_hkec.chain_0 {
            cert uat/u_hkec_edb_all_crt
            chain uat/u_hkec.chain.crt
            key uat/u_hkec_edb_all_crt
        }
    }
    defaults-from uat/u_hkec_high
    inherit-ca-certkeychain true
    inherit-certkeychain false
}

uat/u_hkec_high
ltm profile client-ssl uat/u_hkec_high {
    app-service none
    cert-key-chain {
        default {
            cert Common/default.crt
            key Common/default.key
        }
    }
    cipher-group none
    ciphers
ECDHE-ECDSA-AES128-GCM-SHA256,ECDHE-ECDSA-AES256-GCM-SHA384,ECDHE-RSA-AES128-GCM-S
HA256,ECDHE-RSA-AES256-GCM-SHA384,DHE-RSA-AES128-GCM-SHA256,DHE-RSA-AES256-GCM-SHA
384
    defaults-from Common/clientssl
    inherit-ca-certkeychain true
    inherit-certkeychain true
    options { dont-insert-empty-fragments no-tlsv1.3 no-dtlsv1.2 no-sslv3 }
}

uat/u_hkec_proj_all_crt
ltm profile client-ssl uat/u_hkec_proj_all_crt {
    app-service none
    cert-key-chain {
        u_hkec_proj_all_crt_u_hkec.chain_0 {
            cert uat/u_hkec_proj_all_crt
            chain uat/u_hkec.chain.crt
            key uat/u_hkec_proj_all_crt
        }
    }
    defaults-from uat/u_hkec_default
    inherit-ca-certkeychain true
    inherit-certkeychain false
}

uat/u_hkec_star_all_crt
ltm profile client-ssl uat/u_hkec_star_all_crt {
    app-service none
    cert-key-chain {
        u_hkec_star_all_crt_u_hkec.chain_0 {
            cert uat/u_hkec_star_all_crt
            chain uat/u_hkec.chain.crt
            key uat/u_hkec_star_all_crt
        }
    }
    defaults-from uat/u_hkec_default
    inherit-ca-certkeychain true
    inherit-certkeychain false
}

uat/u_hkec_uat_all_crt
ltm profile client-ssl uat/u_hkec_uat_all_crt {
    app-service none
    cert-key-chain {
        u_hkec_uat_all_crt_u_hkec.chain_0 {
            cert uat/u_hkec_uat_all_crt
            chain uat/u_hkec.chain.crt
            key uat/u_hkec_uat_all_crt
        }
    }
    defaults-from uat/u_hkec_default
    inherit-ca-certkeychain true
    inherit-certkeychain false
}

uat/u_hkec_uat_all_crt_TLSv12-nokRSA
ltm profile client-ssl uat/u_hkec_uat_all_crt_TLSv12-nokRSA {
    app-service none
    cert-key-chain {
        u_hkec_uat_all_crt_u_hkec.chain_0 {
            cert uat/u_hkec_uat_all_crt
            chain uat/u_hkec.chain.crt
            key uat/u_hkec_uat_all_crt
        }
    }
    defaults-from uat/u_hkec_TLSv12-nokRSA
    inherit-ca-certkeychain true
    inherit-certkeychain false
}

uat/u_hkrc_all_crt
ltm profile client-ssl uat/u_hkrc_all_crt {
    app-service none
    cert-key-chain {
        u_hkrc_all_crt_u_hkec.chain_0 {
            cert uat/u_hkrc_all_crt
            chain uat/u_hkec.chain.crt
            key uat/u_hkrc_all_crt
        }
    }
    defaults-from uat/u_hkec_default
    inherit-ca-certkeychain true
    inherit-certkeychain false
}
