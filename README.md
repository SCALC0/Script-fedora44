#!/usr/bin/env bash

################################################################################
# Fedora Complete Maintenance Script
# Autor: ChatGPT
# Versão: 1.0
################################################################################

set -e

GREEN="\e[32m"
YELLOW="\e[33m"
RED="\e[31m"
BLUE="\e[34m"
NC="\e[0m"

echo -e "${BLUE}"
echo "==========================================================="
echo "        FEDORA COMPLETE UPDATE & MAINTENANCE"
echo "==========================================================="
echo -e "${NC}"

################################################################################
# Verifica Root
################################################################################

if [[ $EUID -ne 0 ]]; then
    echo "Execute como root:"
    echo "sudo $0"
    exit 1
fi

################################################################################
# Atualização DNF
################################################################################

echo -e "\n${GREEN}==> Atualizando Fedora...${NC}"
dnf upgrade --refresh -y

################################################################################
# Flatpak
################################################################################

if command -v flatpak >/dev/null; then
    echo -e "\n${GREEN}==> Atualizando Flatpaks...${NC}"
    flatpak update -y
fi

################################################################################
# Snap
################################################################################

if command -v snap >/dev/null; then
    echo -e "\n${GREEN}==> Atualizando Snaps...${NC}"
    snap refresh
fi

################################################################################
# Firmware
################################################################################

if command -v fwupdmgr >/dev/null; then
    echo -e "\n${GREEN}==> Verificando firmware...${NC}"
    fwupdmgr refresh --force
    fwupdmgr get-updates || true
fi

################################################################################
# Remover pacotes desnecessários
################################################################################

echo -e "\n${GREEN}==> Removendo dependências órfãs...${NC}"
dnf autoremove -y

################################################################################
# Limpeza do cache
################################################################################

echo -e "\n${GREEN}==> Limpando cache DNF...${NC}"
dnf clean all

################################################################################
# Limpeza de pacotes antigos
################################################################################

echo -e "\n${GREEN}==> Limpando kernels antigos...${NC}"

dnf remove -y $(dnf repoquery --installonly --latest-limit=-2 -q) 2>/dev/null || true

################################################################################
# Limpeza de logs antigos
################################################################################

echo -e "\n${GREEN}==> Limpando logs antigos...${NC}"

journalctl --vacuum-time=14d

################################################################################
# Atualiza banco locate
################################################################################

if command -v updatedb >/dev/null; then
    echo -e "\n${GREEN}==> Atualizando banco locate...${NC}"
    updatedb
fi

################################################################################
# Saúde SSD SMART
################################################################################

echo -e "\n${GREEN}==> Verificando saúde do SSD...${NC}"

if command -v smartctl >/dev/null; then

    for disk in /dev/nvme0n1 /dev/sda /dev/sdb; do

        if [[ -b $disk ]]; then

            echo
            echo "Dispositivo: $disk"

            smartctl -H "$disk" || true

        fi

    done

elif command -v nvme >/dev/null; then

    nvme smart-log /dev/nvme0 || true

else

    echo "smartmontools não instalado."

fi

################################################################################
# Serviços com erro
################################################################################

echo -e "\n${GREEN}==> Verificando serviços com falha...${NC}"

FAILED=$(systemctl --failed --no-legend)

if [[ -z "$FAILED" ]]; then
    echo "Nenhum serviço com falha."
else
    echo "$FAILED"
fi

################################################################################
# Espaço em disco
################################################################################

echo -e "\n${GREEN}==> Espaço em disco${NC}"
df -h /

################################################################################
# Uso memória
################################################################################

echo -e "\n${GREEN}==> Memória${NC}"
free -h

################################################################################
# ZRAM
################################################################################

if [[ -e /dev/zram0 ]]; then
echo -e "\n${GREEN}==> Uso da ZRAM${NC}"
swapon --show
fi

################################################################################
# Reinicialização necessária?
################################################################################

echo -e "\n${GREEN}==> Verificando necessidade de reinicialização...${NC}"

if needs-restarting -r >/dev/null 2>&1; then
    echo "Reinicialização NÃO necessária."
else
    echo "Recomenda-se reiniciar o computador."
fi

################################################################################
# Resumo
################################################################################

echo
echo "============================================================"
echo "                 RESUMO FINAL"
echo "============================================================"

echo "Sistema atualizado"

echo "Flatpaks atualizados"

echo "Snaps atualizados"

echo "Cache limpo"

echo "Pacotes órfãos removidos"

echo "Logs antigos removidos"

echo "Banco locate atualizado"

echo "Saúde do SSD verificada"

echo "Serviços verificados"

echo

echo "Kernel:"
uname -r

echo

echo "Tempo ligado:"
uptime -p

echo

echo "Espaço livre:"
df -h /

echo

echo "Fim da manutenção."

echo "============================================================"
