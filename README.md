# Projeto para migração de servidores

# Script de Migração: Databricks Hive para Unity Catalog

Este script C# (para Tabular Editor) automatiza a migração de strings de conexão em datasets do Power BI, alterando apontamentos do ambiente legado do Databricks para o novo ambiente Unity Catalog.

## 🚀 Funcionalidades Principais

1.  **Atualização de Servidor e Path (Com Regex)**
    * Identifica strings de conexão apontando para o servidor antigo (`adb-4279...`).
    * Substitui pelo novo servidor (`adb-1691...`).
    * Utiliza **Expressões Regulares (Regex)** para normalizar o `HTTP Path` do warehouse, garantindo que qualquer ID de cluster antigo seja substituído pelo novo ID padrão (`.../ab0b5faa...`).

2.  **Migração de Metastore (Hive > UC)**
    * Localiza referências ao catálogo legado `hive_metastore`.
    * Substitui automaticamente pelo novo catálogo Unity Catalog: `uc_ug_sandbox_prd`.

3.  **Validação Independente**
    * A lógica de substituição do *Metastore* é desacoplada da troca de *Servidor*.
    * **Benefício:** Corrige tabelas que já tiveram o servidor atualizado, mas que ainda apontavam incorretamente para o `hive_metastore`, evitando erros de "Object not found".

## 🛠 Como Usar
Execute este script na janela "C# Script" do Tabular Editor (v2 ou v3) com o modelo conectado. O script itera sobre **Expressões Compartilhadas** e **Partições de Tabelas** (Código M).
                     


``` C# 

using System.Text.RegularExpressions;

// --- CONFIGURAÇÕES ---
string antigoServer = "adb-4279600656705429.9.azuredatabricks.net";
string novoServer   = "adb-1691743261663716.16.azuredatabricks.net";
string novoPath     = "/sql/1.0/warehouses/ab0b5faa759e017e";

// Regex para garantir que o Path seja trocado mesmo se o ID for diferente
string patternPath = @"/sql/1\.0/warehouses/[a-zA-Z0-9]+";

// Strings específicas do Metastore
string buscaHive = "Name=\"hive_metastore\"";
string trocaHive = "Name=\"uc_ug_sandbox_prd\"";

int contador = 0;

// --- 1. EXPRESSÕES COMPARTILHADAS ---
foreach (var expression in Model.Expressions)
{
    string codigo = expression.Expression;
    bool alterado = false;

    // A. Troca de Servidor (Se existir o antigo)
    if (codigo.Contains(antigoServer))
    {
        codigo = codigo.Replace(antigoServer, novoServer);
        // Aplica o Regex no path sempre que achar o servidor antigo
        codigo = Regex.Replace(codigo, patternPath, novoPath);
        alterado = true;
    }

    // B. Troca de Hive Metastore (Verificação INDEPENDENTE)
    // Se encontrar "hive_metastore", troca para "uc_ug_sandbox_prd", mesmo que o servidor já seja o novo.
    if (codigo.Contains(buscaHive))
    {
        codigo = codigo.Replace(buscaHive, trocaHive);
        alterado = true;
    }

    if (alterado)
    {
        expression.Expression = codigo;
        contador++;
        string.Format("Expressão '{0}' atualizada.", expression.Name).Output();
    }
}

// --- 2. PARTIÇÕES DE TABELAS ---
foreach (var table in Model.Tables)
{
    foreach (var partition in table.Partitions)
    {
        if (partition.SourceType == PartitionSourceType.M)
        {
            string codigo = partition.Expression;
            bool alterado = false;

            // A. Troca de Servidor (Se existir o antigo)
            if (codigo.Contains(antigoServer))
            {
                codigo = codigo.Replace(antigoServer, novoServer);
                // Aplica o Regex no path
                codigo = Regex.Replace(codigo, patternPath, novoPath);
                alterado = true;
            }

            // B. Troca de Hive Metastore (Verificação INDEPENDENTE)
            // AQUI ESTÁ A CORREÇÃO: Ele vai entrar aqui mesmo se o servidor já estiver certo.
            if (codigo.Contains(buscaHive))
            {
                codigo = codigo.Replace(buscaHive, trocaHive);
                alterado = true;
            }

            if (alterado)
            {
                partition.Expression = codigo;
                contador++;
                string.Format("Tabela '{0}' atualizada.", table.Name).Output();
            }
        }
    }
}

string.Format("Fim do script. Total de itens alterados: {0}", contador).Output();



```








# Automação de Backup de Relatórios Power BI (.pbix)

## 📋 Descrição
Este script PowerShell automatiza o processo de **backup físico** dos relatórios do Power BI Service. Ele permite definir um Workspace pelo seu **Nome** (em vez de ID) e realiza o download de todos os arquivos `.pbix` viáveis para um diretório local.

## 🚀 Funcionalidades Principais

1.  **Resolução Automática de ID**
    * O usuário fornece apenas o *Nome do Workspace* (ex: "Vendas PRD").
    * O script consulta a API para localizar o GUID (ID) correspondente automaticamente.

2.  **Sanitização de Nomes de Arquivo**
    * Antes de salvar, o script trata o nome do relatório usando Regex.
    * Caracteres proibidos no Windows (como `/`, `\`, `:`, `*`, `?`, `"`, `<`, `>`, `|`) são substituídos por `_`, evitando falhas de gravação no disco.

3.  **Download em Massa (Loop)**
    * Itera sobre todos os relatórios do Workspace.
    * Utiliza o cmdlet `Export-PowerBIReport` para baixar o arquivo físico.

4.  **Tratamento de Erros**
    * Cria automaticamente o diretório de destino se não existir.
    * Captura falhas individuais (ex: relatórios criados apenas na Web que não suportam download) sem interromper o processo dos demais arquivos.

## 🛠 Pré-requisitos

* **PowerShell 5.1** ou superior (ou PowerShell Core).
* Módulo **MicrosoftPowerBIMgmt** instalado.
    ```powershell
    Install-Module -Name MicrosoftPowerBIMgmt -Force
    ```
* Permissões de **Membro** ou **Admin** no Workspace alvo.

## ⚙️ Como Utilizar

1.  Abra o script e edite as variáveis no topo:
    ```powershell
    $NomeWorkspace = "Nome do Seu Workspace"
    $PastaDestino  = "C:\Caminho\Para\Backup"
    ```
2.  Execute o script.
3.  Realize o login na janela interativa da Microsoft.
4.  Aguarde o processamento (o status de cada arquivo será exibido no console: `[OK]` ou `[FALHA]`).

## ⚠️ Limitações Conhecidas (API Microsoft)
O script pode falhar ao baixar relatórios nos seguintes cenários:
* Relatórios criados diretamente no navegador (sem PBIX original).
* Datasets com *Atualização Incremental* configurada.
* Relatórios grandes que excedem o timeout da API.


```Powershell
# --- CONFIGURAÇÕES ---
$NomeWorkspace = "Nome Do Seu Workspace"  # <--- Coloque o nome aqui
$PastaDestino  = "C:\Temp\Backup_PowerBI" # <--- Onde salvar os arquivos

# --- 1. PREPARAÇÃO ---
# Cria a pasta se não existir
if (!(Test-Path -Path $PastaDestino)) {
    New-Item -ItemType Directory -Force -Path $PastaDestino | Out-Null
    Write-Host "Pasta criada: $PastaDestino" -ForegroundColor Cyan
}

# Login
Write-Host "Conectando ao Power BI Service..." -ForegroundColor Cyan
Connect-PowerBIServiceAccount

try {
    # --- 2. DESCOBRIR O ID PELO NOME ---
    Write-Host "Procurando pelo Workspace: '$NomeWorkspace'..." -ForegroundColor Yellow
    $workspace = Get-PowerBIWorkspace -Name $NomeWorkspace

    if ($null -eq $workspace) {
        Write-Error "ERRO: Workspace '$NomeWorkspace' não encontrado."
        return
    }
    
    # Pega o ID (se tiver duplicado, pega o primeiro)
    $wsId = if ($workspace.Count -gt 1) { $workspace[0].Id } else { $workspace.Id }
    Write-Host "Workspace localizado! ID: $wsId" -ForegroundColor Green

    # --- 3. LISTAR RELATÓRIOS ---
    $relatorios = Get-PowerBIReport -WorkspaceId $wsId

    if ($relatorios.Count -eq 0) {
        Write-Host "Nenhum relatório encontrado para baixar." -ForegroundColor Red
        return
    }

    Write-Host "Iniciando download de $($relatorios.Count) relatórios..." -ForegroundColor Cyan

    # --- 4. LOOP DE DOWNLOAD ---
    foreach ($report in $relatorios) {
        
        # Limpa caracteres proibidos no nome do arquivo (ex: / \ : * ? " < > |)
        $nomeArquivoLimpo = $report.Name -replace '[\\/:*?"<>|]', '_'
        $caminhoCompleto = Join-Path -Path $PastaDestino -ChildPath "$nomeArquivoLimpo.pbix"

        Write-Host "Baixando: '$($report.Name)'..." -NoNewline

        try {
            # Tenta baixar o arquivo
            Export-PowerBIReport -Id $report.Id -WorkspaceId $wsId -OutFile $caminhoCompleto
            Write-Host " [OK]" -ForegroundColor Green
        }
        catch {
            # Captura erros comuns (ex: Relatórios criados apenas na Web não baixam como PBIX)
            Write-Host " [FALHA]" -ForegroundColor Red
            Write-Host "    Erro: $($_.Exception.Message)" -ForegroundColor DarkGray
        }
    }

    Write-Host "`nProcesso finalizado. Verifique a pasta: $PastaDestino" -ForegroundColor Cyan
}
catch {
    Write-Error "Erro crítico: $($_.Exception.Message)"
}

```

# Limpeza de Relatórios de Métricas de Uso (PowerShell)

## 📋 Descrição
Este script remove relatórios gerados automaticamente pelo sistema ("Usage Metrics Report") de um Workspace do Power BI.
Este script serve para limpar o workspace desses artefatos caso eles estejam poluindo a lista de conteúdos.

## 🚀 O que o script faz
1.  Conecta ao Workspace pelo **Nome**.
2.  Lista os relatórios e aplica um filtro buscando por nomes que contenham:
    * `*Usage Metrics*` (Padrão em inglês)
    * `*Métricas de uso*` (Padrão em português)
3.  Executa o comando `Remove-PowerBIReport` para excluir permanentemente esses itens.

## ⚠️ Atenção
* A exclusão é **permanente**.
* Se um usuário clicar novamente no botão "Ver métricas de uso" no Power BI Service, o relatório será **gerado novamente** pelo sistema automaticamente.


```Powershell

# --- CONFIGURAÇÕES ---
$NomeWorkspace = "Nome Do Seu Workspace" # <--- Insira o nome aqui

# --- 1. LOGIN ---
Write-Host "Conectando..." -ForegroundColor Cyan
Connect-PowerBIServiceAccount

try {
    # --- 2. BUSCAR WORKSPACE ---
    $workspace = Get-PowerBIWorkspace -Name $NomeWorkspace
    if ($null -eq $workspace) { Write-Error "Workspace não encontrado."; return }
    $wsId = if ($workspace.Count -gt 1) { $workspace[0].Id } else { $workspace.Id }

    # --- 3. FILTRAR RELATÓRIOS DE MÉTRICAS ---
    # Filtra tudo que contém "Usage Metrics" ou "Métricas de Uso" no nome
    Write-Host "Buscando relatórios de métricas em '$NomeWorkspace'..." -ForegroundColor Yellow
    $relatoriosLixo = Get-PowerBIReport -WorkspaceId $wsId | Where-Object { 
        $_.Name -like "*Usage Metrics*" -or $_.Name -like "*Métricas de uso*" 
    }

    if ($relatoriosLixo.Count -eq 0) {
        Write-Host "Nenhum relatório de métricas encontrado para deletar." -ForegroundColor Green
        return
    }

    # --- 4. DELETAR ---
    foreach ($report in $relatoriosLixo) {
        Write-Host "Deletando: '$($report.Name)' (ID: $($report.Id))..." -NoNewline
        
        try {
            # O comando Remove-PowerBIReport deleta o relatório
            Remove-PowerBIReport -Id $report.Id -WorkspaceId $wsId
            Write-Host " [DELETADO]" -ForegroundColor Green
        }
        catch {
            Write-Host " [ERRO]" -ForegroundColor Red
            Write-Host "   Detalhe: $($_.Exception.Message)" -ForegroundColor DarkGray
        }
    }
    
    Write-Host "`nLimpeza concluída." -ForegroundColor Cyan
}
catch {
    Write-Error "Erro: $($_.Exception.Message)"
}


```


# Limpeza em Lote de Relatórios de Métricas (PowerShell)

## 📋 Descrição
Este script realiza a limpeza massiva de relatórios automáticos de "Usage Metrics" (Métricas de Uso) em **múltiplos Workspaces** de uma só vez.

Ao contrário da versão individual, este script aceita uma **lista de IDs (GUIDs)** e itera sobre cada um, removendo os artefatos indesejados sem interrupção.

## 🚀 Funcionalidades
1.  **Processamento em Lote (Batch):** Aceita um array (`@()`) contendo múltiplos IDs de Workspaces.
2.  **Filtro Bilíngue:** Identifica e remove relatórios tanto em inglês (`*Usage Metrics*`) quanto em português (`*Métricas de uso*`).
3.  **Resiliência:** Se um Workspace da lista for inválido ou inacessível, o script registra o erro e **continua para o próximo** automaticamente, garantindo que o lote todo seja processado.

## ⚙️ Como Usar
1.  Abra o script PowerShell.
2.  Preencha a variável `$ListaDeIds` com os IDs dos workspaces alvo:
    ```powershell
    $ListaDeIds = @(
        "8340d90e-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
        "b129a81f-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    )
    ```
3.  Execute o script e realize o login.

``` Powershell

# --- CONFIGURAÇÕES ---
# Coloque aqui a lista de IDs dos Workspaces que você quer limpar
$ListaDeIds = @(
    "guid-id-workspace-01",
    "guid-id-workspace-02",
    "guid-id-workspace-03" 
    # Adicione quantos IDs precisar, separados por vírgula
)

# --- 1. LOGIN ---
Write-Host "Conectando ao Power BI Service..." -ForegroundColor Cyan
Connect-PowerBIServiceAccount

# --- 2. LOOP PELOS WORKSPACES ---
foreach ($wsId in $ListaDeIds) {
    Write-Host "---------------------------------------------------"
    Write-Host "Processando Workspace ID: $wsId" -ForegroundColor Yellow

    try {
        # Tenta buscar os relatórios deste workspace
        # O parâmetro -ErrorAction Stop garante que se o ID for inválido, ele caia no catch
        $todosRelatorios = Get-PowerBIReport -WorkspaceId $wsId -ErrorAction Stop

        # --- 3. FILTRAR RELATÓRIOS ---
        $relatoriosLixo = $todosRelatorios | Where-Object { 
            $_.Name -like "*Usage Metrics*" -or $_.Name -like "*Métricas de uso*" 
        }

        if ($relatoriosLixo.Count -eq 0) {
            Write-Host " > Nenhum relatório de métricas encontrado." -ForegroundColor DarkGray
        }
        else {
            # --- 4. DELETAR ---
            foreach ($report in $relatoriosLixo) {
                Write-Host " > Deletando: '$($report.Name)'..." -NoNewline
                
                try {
                    Remove-PowerBIReport -Id $report.Id -WorkspaceId $wsId -ErrorAction Stop
                    Write-Host " [SUCESSO]" -ForegroundColor Green
                }
                catch {
                    Write-Host " [FALHA]" -ForegroundColor Red
                    Write-Host "   Erro: $($_.Exception.Message)" -ForegroundColor DarkGray
                }
            }
        }
    }
    catch {
        Write-Host " > ERRO ao acessar o Workspace $wsId. Verifique se o ID existe e se você tem acesso." -ForegroundColor Red
        Write-Host "   Detalhe: $($_.Exception.Message)" -ForegroundColor DarkGray
    }
}

Write-Host "---------------------------------------------------"
Write-Host "Processamento em lote finalizado." -ForegroundColor Cyan


```

# Script de Migração em Massa (PowerShell + XMLA)

## 📋 Descrição
Este script executa a lógica de migração "Databricks Hive -> Unity Catalog" em **todos os Datasets** de um Workspace específico via PowerShell.

Ele utiliza a biblioteca TOM (Tabular Object Model) para iterar sobre cada modelo semântico e aplicar as correções no código M.

## 🚀 Lógica Aplicada (Idêntica ao C#)
1.  **Conexão XMLA:** Conecta ao workspace como se fosse um servidor Analysis Services.
2.  **Regex no Path:** Garante que qualquer ID de warehouse antigo seja substituído pelo novo.
3.  **Correção Independente de Metastore:**
    * Verifica e troca o servidor antigo.
    * Verifica e troca `hive_metastore` por `uc_ug_sandbox_prd` independentemente do servidor, corrigindo relatórios parcialmente migrados.

## ⚙️ Como Usar
1.  Edite a variável `$NomeWorkspace`.
2.  Garanta que o endpoint XMLA do workspace esteja habilitado para Leitura/Gravação.
3.  Execute no PowerShell ISE ou VS Code como Administrador.



