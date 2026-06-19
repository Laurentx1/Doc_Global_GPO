# Instalacao do Agente Futura Cyber Protection (Acronis) via GPO

**Cliente:** Global Logistica (global.log)  
**Objetivo:** Implantar o agente Futura Cyber Protection (Acronis rebrandeado) automaticamente nas máquinas do domínio via GPO.  
**Servidor AD:** SRV-AD.global.log  
**Máquina de teste:** PC-GPO (Windows 11 Pro, build 26200)  
**Data de validação:** 18/06/2026  

---

## 1. Descobertas importantes

- O `.msi` (`BackupClient64.msi`) sozinho **não instala** - dá erro **1603**. Ele depende do `.exe` (`FuturaCyberProtection.exe`), que aplica permissões e o token de registro.  
- O `.exe` usa parâmetro com **duplo traço**: `--quiet` (não `/quiet`).  
- Software Installation via GPO falha com **1085/1603**.  
- GPO de Script aceita `.exe` direto, mas parâmetros devem ir no campo separado.  
- Sem `--quiet`, o instalador roda em modo interativo.  
- O instalador baixa ~1GB por instalação.  
- Conflito com Milvus resolvido com delay de 120s.  
- Política "Sempre aguardar a rede na inicialização" obrigatória.  
- GPO duplicadas removidas.  

---

## 2. Arquivos no compartilhamento

| Arquivo | Caminho |
|---|---|
| Instalador | \\Srv-files\gpo\antivirus\acronis\FuturaCyberProtection.exe |
| Script (.bat) | \\Srv-files\gpo\antivirus\acronis\instala_acronis.bat |
| MSI | \\Srv-files\gpo\antivirus\acronis\BackupClient64.msi |

---

## 3. Script de instalação

```bat
@echo off
timeout /t 120 /nobreak >nul
"\\Srv-files\gpo\antivirus\acronis\FuturaCyberProtection.exe" --quiet --log-dir=C:\acronis_boot_log
exit 0
```

---

## 4. Configuração GPO

Inicialização do Computador:
- Scripts → Inicialização → adicionar `.bat`
- Parâmetros: vazio

Política obrigatória:
- Sempre aguardar a rede na inicialização = Habilitado

---

## 5. Validação

```cmd
tasklist | findstr /i "futura acronis"
```

Log:
```
C:cronis_boot_logootstrapper.log
```

---

## 6. Resultado

- Instalado como SYSTEM  
- Registro automático na Futura  
- Resultado 0 (sucesso)  

---

## 7. Observações

- Substituir EXE mantém funcionamento  
- Reinstala automaticamente se removido  
- Delay pode ser ajustado conforme rede  
