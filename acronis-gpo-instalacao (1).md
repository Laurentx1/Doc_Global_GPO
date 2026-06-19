# Instalacao do Agente Futura Cyber Protection (Acronis) via GPO

**Cliente:** Global Logistica (global.log)
**Objetivo:** Implantar o agente Futura Cyber Protection (Acronis rebrandeado) automaticamente nas maquinas do dominio via GPO.
**Servidor AD:** SRV-AD.global.log
**Maquina de teste:** PC-GPO (Windows 11 Pro, build 26200)
**Data de validacao:** 18/06/2026

---

## 1. Descobertas importantes

- O `.msi` (`BackupClient64.msi`) sozinho **nao instala** - da erro **1603**. Ele depende do `.exe` (`FuturaCyberProtection.exe`), que aplica permissoes e o token de registro.
- O `.exe` usa parametro com **duplo traco**: `--quiet` (nao `/quiet`).
- **Software Installation** via GPO (atribuir o MSI) falha com **1085/1603**. Descartado.
- GPO de Script aceita `.exe` direto, mas o parametro vai no campo separado **"Parametros de Script"**, nunca junto ao nome do arquivo.
- Sem `--quiet`, o instalador roda em **modo interativo** e abre janela na tela do usuario.
- O instalador baixa **~1GB da internet** (`https://dl.managed-protection.com`) a cada instalacao. Leva 3-5 minutos dependendo da conexao.
- Quando Acronis e Milvus instalam ao mesmo tempo no boot, o Acronis falha com erro `0x310000a` ("Ha outra instalacao em andamento"). Solucao: adicionar delay de 2 minutos no script do Acronis.
- A politica **"Sempre aguardar a rede na inicializacao"** e obrigatoria para o script conseguir acessar `\\Srv-files` no boot antes do login.
- **GPOs duplicadas** (Deploy Acronis, GPO - Acronis Antivirus) causavam conflito. Mantida apenas **Acronis Antivirus**.

---

## 2. Arquivos no compartilhamento

| Arquivo | Caminho |
|---|---|
| Instalador | `\\Srv-files\gpo\antivirus\acronis\FuturaCyberProtection.exe` |
| Script (.bat) | `\\Srv-files\gpo\antivirus\acronis\instala_acronis.bat` |
| MSI (nao usado sozinho) | `\\Srv-files\gpo\antivirus\acronis\BackupClient64.msi` |

---

## 3. Solucao validada para producao (Inicializacao / SYSTEM)

Roda como **AUTORIDADE NT\SISTEMA** no boot, antes do login. Nao pede senha de administrador independente do perfil do usuario logado.

### 3.1 Conteudo do script

Arquivo: `\\Srv-files\gpo\antivirus\acronis\instala_acronis.bat`

```bat
@echo off
timeout /t 120 /nobreak >nul
"\\Srv-files\gpo\antivirus\acronis\FuturaCyberProtection.exe" --quiet --log-dir=C:\acronis_boot_log
exit 0
```

> O `timeout /t 120` aguarda 2 minutos para o Milvus terminar de instalar antes de iniciar o Acronis. Sem esse delay, os dois instaladores conflitam e o Acronis falha.

### 3.2 Configuracao da GPO

**Caminho no GPMC:**

```
GPO Acronis Antivirus
  > Configuracao do Computador
    > Politicas
      > Configuracoes do Windows
        > Scripts (Inicializacao/Encerramento)
          > Inicializacao
            > Adicionar
```

| Campo | Valor |
|---|---|
| Nome do Script | `\\Srv-files\gpo\antivirus\acronis\instala_acronis.bat` |
| Parametros | (vazio) |

**Politica adicional obrigatoria:**

```
GPO Acronis Antivirus
  > Configuracao do Computador
    > Modelos Administrativos
      > Sistema
        > Logon
          > "Sempre aguardar a rede na inicializacao e no logon do computador"
            > Habilitado
```

**Garantir que Software Installation esteja vazio:**

```
GPO Acronis Antivirus
  > Configuracao do Computador
    > Configuracoes de Software
      > Instalacao de Software
        > (nenhum pacote listado)
```

---

## 4. Validacao pos-instalacao

Apos reiniciar a maquina (aguardar 5-8 minutos para o boot + instalacao completa):

```cmd
tasklist | findstr /i "futura acronis cyber"
```

Resultado esperado:
```
cyber-desktop-service.exe     Services
cyber-scripting-executor.     Services
```

Para acompanhar o progresso durante o boot:

```cmd
notepad C:\acronis_boot_log\bootstrapper.log
```

Para verificar se rodou como SYSTEM (linha `Current user` do log):
```
Current user: 'AUTORIDADE NT\SISTEMA'
```

---

## 5. Evidencia de validacao

Log do boot confirmou:

```
Current user: 'AUTORIDADE NT\SISTEMA'
Command line parameters: '--quiet --log-dir=C:\acronis_boot_log'
address: 'https://acronis.futuratecnologia.com.br'
result: '0'
Operation 'InstallMsiOperation' has been successfully completed.
```

- Rodou como SYSTEM (sem pedir senha)
- Conectou no datacenter da Futura e registrou o agente
- Instalacao concluida com sucesso (result 0)

---

## 6. Checklist de implantacao

- [ ] Script `instala_acronis.bat` salvo em `\\Srv-files\gpo\antivirus\acronis\`
- [ ] GPO Acronis Antivirus com script de Inicializacao apontando para o `.bat`
- [ ] Campo "Parametros" vazio na configuracao do script
- [ ] Politica "Sempre aguardar a rede na inicializacao" habilitada
- [ ] Software Installation vazio na GPO
- [ ] Nenhuma GPO duplicada de Acronis vinculada
- [ ] Maquina reiniciada (nao apenas gpupdate /force)
- [ ] Aguardar 5-8 minutos apos o boot
- [ ] Validar com tasklist
- [ ] Confirmar registro no painel web da Futura Cyber Protection

---

## 7. Observacoes para atualizacoes futuras

- Para atualizar o instalador, substitua o `FuturaCyberProtection.exe` na pasta mantendo **exatamente o mesmo nome**.
- O script nao verifica se ja esta instalado — se o Acronis for desinstalado, na proxima reinicializacao ele volta automaticamente.
- O log fica em `C:\acronis_boot_log\bootstrapper.log` em cada maquina — util para diagnostico.
- O delay de 120 segundos pode precisar de ajuste se o Milvus demorar mais para instalar em conexoes mais lentas.
