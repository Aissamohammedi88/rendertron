#!/usr/bin/env bash
# ═══════════════════════════════════════════════════════════════════════════════
# NEXUS STRATEGIC BROWSER - INSTALLATION & ARCHITECTURE
# Objectif : Préparer le terrain pour un navigateur plus rapide que Chrome
# Stack : Rust (Tauri) + WebKitGTK + Frontend Local
# Auteur : Aissa Mohammedi (DGK) - NEXUS-OPEN-3.0
# ═══════════════════════════════════════════════════════════════════════════════

set -euo pipefail

readonly C_R='\033[1;31m'
readonly C_G='\033[1;32m'
readonly C_Y='\033[1;33m'
readonly C_C='\033[1;36m'
readonly C_M='\033[1;35m'
readonly C_N='\033[0m'

echo -e "${C_M}╔══════════════════════════════════════════════════════════════╗${C_N}"
echo -e "${C_M}║   NEXUS STRATEGIC BROWSER - DÉPLOIEMENT                       ║${C_N}"
echo -e "${C_M}╚══════════════════════════════════════════════════════════════╝${C_N}"

# ==============================================================================
# 1. INSTALLATION DES PACKAGES STRATÉGIQUES (Lignes 20-50)
# ==============================================================================

echo -e "\n${C_C}[1/4] Installation des packages système...${C_N}"

# Mise à jour des dépôts
sudo apt-get update -qq

# Packages de compilation et de rendu (WebKitGTK 4.1 = le plus récent)
sudo apt-get install -y -qq \
    build-essential \
    curl \
    wget \
    file \
    libssl-dev \
    libgtk-3-dev \
    libwebkit2gtk-4.1-dev \
    libayatana-appindicator3-dev \
    librsvg2-dev \
    patchelf \
    libjavascriptcoregtk-4.1-dev \
    gir1.2-webkit2-4.1 \
    sqlite3 \
    jq \
    rsync

echo -e "${C_G}[OK] Packages système installés.${C_N}"

# Installation de Rust (Le moteur du navigateur)
if ! command -v rustc &>/dev/null; then
    echo -e "${C_Y}[*] Installation de Rust (Cargo)...${C_N}"
    curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
    source "$HOME/.cargo/env"
else
    echo -e "${C_G}[OK] Rust est déjà installé.${C_N}"
fi

# Installation de Node.js (Pour le frontend si nécessaire)
if ! command -v node &>/dev/null; then
    echo -e "${C_Y}[*] Installation de Node.js...${C_N}"
    curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
    sudo apt-get install -y -qq nodejs
else
    echo -e "${C_G}[OK] Node.js est déjà installé.${C_N}"
fi

# Installation de Tauri CLI (Le framework d'application)
if ! command -v cargo-tauri &>/dev/null; then
    echo -e "${C_Y}[*] Installation de Tauri CLI...${C_N}"
    cargo install tauri-cli
fi

# ==============================================================================
# 2. CRÉATION DE L'ARBORESCENCE (Paths Spéciaux) (Lignes 51-100)
# ==============================================================================

echo -e "\n${C_C}[2/4] Création des paths stratégiques...${C_N}"

BROWSER_DIR="$HOME/NexusBrowser"
mkdir -p "$BROWSER_DIR"/{src-tauri,src,public,assets,data,cache,logs}

cd "$BROWSER_DIR"

# Création du fichier de configuration Tauri
cat << 'EOF' > src-tauri/Cargo.toml
[package]
name = "nexus-browser"
version = "1.0.0"
description = "Nexus Strategic Browser"
authors = ["Aissa Mohammedi (DGK)"]
edition = "2021"

[build-dependencies]
tauri-build = { version = "1.5", features = [] }

[dependencies]
tauri = { version = "1.5", features = ["shell-open"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
EOF

# Création du fichier de configuration principal Tauri
cat << 'EOF' > src-tauri/tauri.conf.json
{
  "build": {
    "beforeBuildCommand": "",
    "beforeDevCommand": "",
    "devPath": "../src",
    "distDir": "../src"
  },
  "package": {
    "productName": "NexusBrowser",
    "version": "1.0.0"
  },
  "tauri": {
    "allowlist": {
      "shell": {
        "all": true,
        "open": true
      }
    },
    "bundle": {
      "active": true,
      "targets": "all",
      "identifier": "com.nexus.browser"
    },
    "security": {
      "csp": "default-src 'self'; img-src 'self' data: https:; script-src 'self'"
    },
    "windows": [
      {
        "fullscreen": false,
        "height": 800,
        "resizable": true,
        "title": "Nexus Browser",
        "width": 1280
      }
    ]
  }
}
EOF

echo -e "${C_G}[OK] Arborescence et configuration Tauri créées dans $BROWSER_DIR.${C_N}"

# ==============================================================================
# 3. GÉNÉRATION DU CODE SOURCE (Rust + HTML) (Lignes 101-150)
# ==============================================================================

echo -e "\n${C_C}[3/4] Génération du code source...${C_N}"

# Code Rust (Point d'entrée)
cat << 'EOF' > src-tauri/src/main.rs
#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]

fn main() {
    tauri::Builder::default()
        .invoke_handler(tauri::generate_handler![search_web])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}

#[tauri::command]
fn search_web(query: String) -> String {
    // Ici, on pourrait appeler l'API Nexus sur le port 8190
    // Pour l'instant, on redirige vers une recherche locale
    format!("https://duckduckgo.com/?q={}", query)
}
EOF

# Interface HTML (Frontend ultra-léger)
cat << 'EOF' > src/index.html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <title>Nexus Browser</title>
    <style>
        body { margin: 0; background: #0a0a0f; color: #fff; font-family: monospace; display: flex; flex-direction: column; height: 100vh; }
        .bar { display: flex; padding: 10px; background: #151520; border-bottom: 1px solid #333; }
        input { flex: 1; padding: 10px; background: #000; border: 1px solid #00f3ff; color: #00f3ff; border-radius: 5px; outline: none; }
        button { padding: 10px 20px; margin-left: 10px; background: #00f3ff; color: #000; border: none; border-radius: 5px; font-weight: bold; cursor: pointer; }
        iframe { flex: 1; border: none; width: 100%; }
    </style>
</head>
<body>
    <div class="bar">
        <input type="text" id="url" value="https://duckduckgo.com" onkeypress="if(event.key==='Enter') loadPage()">
        <button onclick="loadPage()">GO</button>
    </div>
    <iframe id="frame" src="https://duckduckgo.com"></iframe>

    <script>
        function loadPage() {
            const url = document.getElementById('url').value;
            document.getElementById('frame').src = url;
        }
    </script>
</body>
</html>
EOF

echo -e "${C_G}[OK] Code source généré.${C_N}"

# ==============================================================================
# 4. LANCEMENT (Lignes 151-200)
# ==============================================================================

echo -e "\n${C_C}[4/4] Compilation et lancement...${C_N}"
echo -e "${C_Y}Note : La première compilation peut prendre 2 à 3 minutes.${C_N}"

# Lancement de Tauri en mode dev (compilation + ouverture de la fenêtre)
cd "$BROWSER_DIR"
cargo tauri dev
