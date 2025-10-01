#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
Script de copias de seguridad automáticas
Autor: [Samira]
Fecha: 1/10/2025
Descripción:
    Permite realizar copias de seguridad de proyectos locales hacia:
    1. Un repositorio remoto en GitHub.
    2. Una carpeta de red (NAS o compartida).
"""

import os
import shutil
import subprocess
import hashlib
import logging
from pathlib import Path
import argparse
from datetime import datetime

# ===================== CONFIGURACIÓN =====================
# Carpeta base de proyectos locales
PROJECTS_DIR = Path.home() / "ProyectosDAM"

# Carpeta destino en NAS / red (ejemplo en Windows: Z:\Backups\ProyectosDAM)
NAS_DIR = Path(r"//NAS/Backups/ProyectosDAM")

# Configuración de logs
LOG_FILE = Path(__file__).parent / "backup.log"
logging.basicConfig(
    filename=LOG_FILE,
    level=logging.INFO,
    format="%(asctime)s - %(levelname)s - %(message)s"
)

# ===================== FUNCIONES =====================
def hash_file(path):
    """Genera un hash SHA256 de un archivo para comparar cambios."""
    h = hashlib.sha256()
    with open(path, "rb") as f:
        for chunk in iter(lambda: f.read(4096), b""):
            h.update(chunk)
    return h.hexdigest()

def backup_to_github(project_dir: Path):
    """Realiza commit y push automático en un repositorio GitHub."""
    try:
        if not (project_dir / ".git").exists():
            logging.warning(f"{project_dir} no es un repositorio git.")
            return
        
        # git add
        subprocess.run(["git", "-C", str(project_dir), "add", "."], check=True)
        # git commit (solo si hay cambios)
        result = subprocess.run(
            ["git", "-C", str(project_dir), "status", "--porcelain"],
            capture_output=True, text=True
        )
        if result.stdout.strip():
            msg = f"Backup automático {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}"
            subprocess.run(["git", "-C", str(project_dir), "commit", "-m", msg], check=True)
            subprocess.run(["git", "-C", str(project_dir), "push"], check=True)
            logging.info(f"Backup en GitHub completado para {project_dir}")
        else:
            logging.info(f"No hay cambios en {project_dir}, no se hizo commit.")
    except subprocess.CalledProcessError as e:
        logging.error(f"Error en GitHub backup {project_dir}: {e}")

def backup_to_nas(project_dir: Path, dest_dir: Path):
    """Copia incremental de proyectos a NAS evitando duplicados."""
    try:
        target = dest_dir / project_dir.name
        target.mkdir(parents=True, exist_ok=True)
        for root, _, files in os.walk(project_dir):
            rel_path = Path(root).relative_to(project_dir)
            dest_path = target / rel_path
            dest_path.mkdir(parents=True, exist_ok=True)
            for file in files:
                src_file = Path(root) / file
                dst_file = dest_path / file
                if dst_file.exists():
                    if hash_file(src_file) == hash_file(dst_file):
                        continue  # evitar duplicado
                shutil.copy2(src_file, dst_file)
        logging.info(f"Backup en NAS completado para {project_dir}")
    except Exception as e:
        logging.error(f"Error en NAS backup {project_dir}: {e}")

# ===================== MAIN =====================
def main():
    parser = argparse.ArgumentParser(description="Script de copias de seguridad automáticas")
    parser.add_argument("--dest", choices=["github", "nas"], required=True,
                        help="Destino de la copia de seguridad (github o nas)")
    args = parser.parse_args()

    if not PROJECTS_DIR.exists():
        logging.error(f"La carpeta {PROJECTS_DIR} no existe.")
        print(f"Error: la carpeta {PROJECTS_DIR} no existe.")
        return

    for project in PROJECTS_DIR.iterdir():
        if project.is_dir():
            if args.dest == "github":
                backup_to_github(project)
            elif args.dest == "nas":
                backup_to_nas(project, NAS_DIR)

    print("Backup finalizado. Verifique el log para más detalles.")

if __name__ == "__main__":
    main()
