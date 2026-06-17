# Correção da Tela de Bloqueio (Lock Screen) via GPO - Global Logística

**##NÃO USADO MAIS DESSA FORMA**

## Informações do Chamado

| Campo | Valor |
|---|---|
| Cliente | Global Logística |
| Domínio | global.log |
| Chamados | 0626-000345 e 0626-000347 |
| Técnico responsável | Talles Geovani |
| Data | 15/06/2026 |
| GPO afetada | Tela de Bloqueio e Plano de Fundo |
| GUID da GPO | {06047A9B-B307-4DFC-8D97-7716ABEBEF9B} |
| Vínculo da GPO | global.log (raiz do domínio) |
| Sistema operacional das estacoes | Windows 11 Pro (build 26100) |

---

## 1. Problema Relatado

Em todo o parque de máquinas da Global Logística, o plano de fundo da área de trabalho (wallpaper) era trocado normalmente pela GPO, porém a imagem da tela de bloqueio (lock screen) não era aplicada em nenhuma máquina, tanto nas antigas quanto nas recém adicionadas ao Active Directory.

Não havia mensagem de erro na tela das estacoes. A tela de bloqueio simplesmente não trocava para a arte definida pela equipe.

Observação do cliente: a arte da tela de bloqueio e trocada mensalmente e usada pelo RH como canal de informações básicas, portanto a função e considerada essencial.

---

## 2. Diagnostico Realizado

Foram verificados todos os pontos da configuração, e todos estavam corretos:

1. A GPO estava habilitada e vinculada a raiz do domínio (global.log), alcançando todas as máquinas.
2. O filtro de segurança estava como Usuários Autenticados (correto).
3. O relatório `gpresult /h` confirmou que a GPO Tela de Bloqueio e Plano de Fundo estava em GPOs Aplicadas, no escopo do computador.
4. O arquivo da arte existia localmente na máquina em `C:\bg\lockScreen.jpg`, com data recente (arte nova).
5. A chave de registro `HKLM\SOFTWARE\Policies\Microsoft\Windows\Personalization\LockScreenImage` estava gravada apontando para `C:\bg\lockScreen.jpg`.
6. A chave `LockScreenOverlaysDisabled` estava com valor 1 (Windows Spotlight desativado).
7. A máquina foi reiniciada e mesmo assim a arte não foi aplicada.

Conclusão do diagnostico: a GPO estava chegando, sendo processada e gravando o registro corretamente. O problema não era permissão, não era a GPO e não era o arquivo. Por isso, refazer a GPO do zero não resolveria.

---

## 3. Causa Raiz

A política utilizada era:

```
Configuração do Computador
  > Políticas
    > Modelos Administrativos
      > Painel de Controle
        > Personalização
          > Forcar uma imagem padrão especifica de tela de bloqueio e de logon
```

Essa política funciona apenas nas edições Enterprise, Education e Server do Windows. Em máquinas Windows 11 Pro, ela até grava a chave de registro e aparece como aplicada no gpresult, mas o sistema ignora essa configuração na hora de renderizar a tela de bloqueio.

Como todo o parque da Global Logística utiliza Windows 11 Pro, a tela de bloqueio nunca era aplicada. O plano de fundo da área de trabalho continuava funcionando porque utiliza um mecanismo diferente, que o Windows Pro respeita.

Resumo da causa raiz: limitação de edição do Windows. A política de tela de bloqueio por Modelos Administrativos não e imposta em edições pro.

---

## 4. Solução Aplicada

A solução foi substituir o método de aplicação da imagem por chaves de registro do PersonalizationCSP, que funcionam em todas as edições do Windows, incluindo a Pro. O arquivo da arte continua sendo copiado do servidor para a pasta local, garantindo funcionamento também fora da rede da empresa.

A configuração foi feita dentro da própria GPO Tela de Bloqueio e Plano de Fundo, em três partes.

### 4.1. Cópia do arquivo (já existente, mantida)

```
Configuração do Computador
  > Preferencias
    > Configurações do Windows
      > Arquivos
```

| Ação | Origem | Destino |
|---|---|---|
| Substituir | \\Srv-files\gpo\papeldeparede\lockScreen.jpg | C:\bg\lockScreen.jpg |
| Substituir | \\Srv-files\gpo\papeldeparede\backgroundScreen.jpg | C:\bg\backgroundScreen.jpg |

A ação substituir garante que, quando a arte for trocada no servidor, a máquina recebe a versão nova no próximo ciclo de atualização da GPO.

### 4.2. Criação da pasta (já existente, mantida)

```
Configuração do Computador
  > Preferencias
    > Configurações do Windows
      > Pastas
```

| Ação | Caminho |
|---|---|
| Atualizar | C:\bg\ |

### 4.3. Chaves de registro do PersonalizationCSP (novo, o que resolveu)

```
Configuração do Computador
  > Preferencias
    > Configurações do Windows
      > Registro
```

Foram criados três itens de registro, todos com Ação Atualizar, Hive HKEY_LOCAL_MACHINE, no caminho:

```
SOFTWARE\Microsoft\Windows\CurrentVersion\PersonalizationCSP
```

| Nome do Valor | Tipo | Dado |
|---|---|---|
| LockScreenImagePath | REG_SZ | C:\bg\lockScreen.jpg |
| LockScreenImageUrl | REG_SZ | C:\bg\lockScreen.jpg |
| LockScreenImageStatus | REG_DWORD | 1 |

### 4.4. Políticas mantidas habilitadas

As políticas abaixo foram mantidas, pois impedem que o usuário altere a imagem manualmente:

- Impedir alteração de imagem da tela de bloqueio e de logon (Habilitado)
- Impedir alteração de plano de fundo do menu Iniciar (Habilitado)

A política forçar uma imagem padrão especifica de tela de bloqueio pode ser mantida habilitada sem prejuízo, mas quem efetivamente aplica a imagem no Windows Pro são as chaves do PersonalizationCSP.

---

## 5. Validação na Máquina de Teste

Antes de confiar apenas na GPO, as chaves foram aplicadas manualmente em uma estação Windows 11 Pro para confirmar que o método funciona. Comandos executados no CMD como administrador:

```cmd
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\PersonalizationCSP" /v LockScreenImagePath /t REG_SZ /d "C:\bg\lockScreen.jpg" /f
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\PersonalizationCSP" /v LockScreenImageUrl /t REG_SZ /d "C:\bg\lockScreen.jpg" /f
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\PersonalizationCSP" /v LockScreenImageStatus /t REG_DWORD /d 1 /f
```

Em seguida, a tela foi bloqueada com a combinação de teclas Windows + L. A arte correta foi exibida com sucesso, confirmando a solução.

Para conferir as chaves gravadas em qualquer máquina:

```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\PersonalizationCSP"
```

---

## 6. Como Propagar para Todo o Parque

A GPO se propaga sozinha. Cada máquina busca a atualização automaticamente em duas situações:

1. Na inicialização (boot) da máquina.
2. No ciclo automático de atualização da Política de Grupo, a cada 90 minutos aproximadamente.

As máquinas que estiverem desligadas recebem a configuração no próximo boot.

### Opções para acelerar a aplicação (opcional)

Forcar a atualização em uma máquina específica (CMD como administrador na própria máquina):

```cmd
gpupdate /force
```

Forcar a atualização em todas as máquinas de uma Unidade Organizacional, pelo Console de Gerenciamento de Política de Grupo (gpmc.msc): clicar com o botão direito na OU COMPUTADORES e selecionar Atualização de Política de Grupo.

Forcar a atualização em todas as máquinas do domínio via PowerShell no servidor (como administrador):

```powershell
Get-ADComputer -Filter * | ForEach-Object { Invoke-GPUpdate -Computer $_.Name -Force -RandomDelayInMinutes 0 }
```

Observação: as opções de atualização remota dependem de o firewall das máquinas estar liberado para gerenciamento remoto (regras de Tarefas Agendadas Remotas e WMI) e só atingem máquinas ligadas no momento. O método via PowerShell pode demorar, pois aguarda o tempo limite de cada máquina desligada antes de seguir para a próxima.

---

## 7. Manutenção Mensal da Arte

Para trocar a arte da tela de bloqueio nos meses seguintes:

1. Substituir o arquivo no servidor em `\\Srv-files\gpo\papeldeparede\lockScreen.jpg`, mantendo exatamente o nome de arquivo.
2. Não é necessário alterar a GPO.
3. Como a ação de cópia esta como substituir, cada máquina copia a versão nova no próximo ciclo de atualização da GPO e a tela de bloqueio passa a exibir a arte atualizada.

---

## 8. Checklist de Verificação

- [ ] GPO Tela de Bloqueio e Plano de Fundo habilitada e vinculada a global.log
- [ ] Preferência de Pasta criando C:\bg\ (Ação Atualizar)
- [ ] Preferência de Arquivo copiando lockScreen.jpg do servidor para C:\bg\ (Ação Substituir)
- [ ] Preferência de Arquivo copiando backgroundScreen.jpg do servidor para C:\bg\ (Ação Substituir)
- [ ] Item de Registro LockScreenImagePath (REG_SZ) = C:\bg\lockScreen.jpg
- [ ] Item de Registro LockScreenImageUrl (REG_SZ) = C:\bg\lockScreen.jpg
- [ ] Item de Registro LockScreenImageStatus (REG_DWORD) = 1
- [ ] Políticas de impedir alteracao mantidas habilitadas
- [ ] Arquivo lockScreen.jpg presente em C:\bg\ na máquina de teste
- [ ] Chaves do PersonalizationCSP confirmadas com reg query na máquina de teste
- [ ] Tela de bloqueio exibindo a arte correta após Windows + L
- [ ] Validado em pelo menos uma estação que recebeu a configuração via GPO (sem o reg add manual)

