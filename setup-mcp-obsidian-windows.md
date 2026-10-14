---
tags: [obsidian, mcp, claude-code, windows, setup]
date: 2026-10-14
status: pendente
---

# Setup do mcp-obsidian no Claude Code (Windows)

> Passo a passo para instalar o plugin Local REST API no vault e registrar o MCP server mcp-obsidian no Claude Code, lendo a API key automaticamente do `data.json`.

Relacionado: [[Home]]

---

## Contexto

- MCP server: [mcp-obsidian](https://github.com/MarkusPfundstein/mcp-obsidian)
- Depende do plugin community [Local REST API](https://github.com/coddingtonbear/obsidian-local-rest-api)
- A API key fica em `<VAULT>\.obsidian\plugins\obsidian-local-rest-api\data.json` (campo `apiKey`)
- A lista de vaults fica em `%APPDATA%\obsidian\obsidian.json`
- Porta padrão: HTTPS 27124 (HTTP 27123 se `enableInsecureServer` estiver ativo)

## Status

Ainda não executado. A sessão de cloud do Claude Code roda em Linux e não tem acesso à máquina Windows. Rodar o prompt original no Claude Code local (CLI ou app desktop), ou executar os comandos abaixo no PowerShell.

## 1. Listar os vaults

```powershell
(Get-Content "$env:APPDATA\obsidian\obsidian.json" -Raw | ConvertFrom-Json).vaults.PSObject.Properties | % { $_.Value.path }
```

## 2. Instalar o plugin (só se ainda não existir)

Faz backup de `community-plugins.json` antes de editar.

```powershell
$vault = "C:\CAMINHO\DO\VAULT"
$plug  = "$vault\.obsidian\plugins\obsidian-local-rest-api"
if (-not (Test-Path $plug)) {
  New-Item -ItemType Directory $plug | Out-Null
  $rel = Invoke-RestMethod https://api.github.com/repos/coddingtonbear/obsidian-local-rest-api/releases/latest
  foreach ($n in 'main.js','manifest.json','styles.css') {
    $a = $rel.assets | ? name -eq $n
    if ($a) { Invoke-WebRequest $a.browser_download_url -OutFile "$plug\$n" }
  }
  $cp = "$vault\.obsidian\community-plugins.json"
  $list = @()
  if (Test-Path $cp) { Copy-Item $cp "$cp.bak"; $list = @(Get-Content $cp -Raw | ConvertFrom-Json) }
  if ($list -notcontains 'obsidian-local-rest-api') { $list += 'obsidian-local-rest-api' }
  [IO.File]::WriteAllText($cp, (ConvertTo-Json -InputObject @($list)))
}
```

## 3. Reiniciar o Obsidian

Fechar e reabrir o Obsidian para o plugin gerar a `apiKey`. Se o modo restrito estiver ativo, desativar em Configurações → Plugins da comunidade.

## 4–7. Ler a chave, registrar e validar

A chave é mascarada no output (`sk-...abcd`).

```powershell
$d    = Get-Content "$plug\data.json" -Raw | ConvertFrom-Json
$key  = $d.apiKey
$port = if ($d.port) { $d.port } else { 27124 }
"apiKey: sk-...$($key.Substring($key.Length-4))  port: $port  insecure: $($d.enableInsecureServer)"
$uvx  = (Get-Command uvx -ErrorAction SilentlyContinue).Source   # vazio → instalar uv: winget install --id=astral-sh.uv -e
claude mcp add mcp-obsidian --scope user --env OBSIDIAN_API_KEY=$key --env OBSIDIAN_HOST=127.0.0.1 --env OBSIDIAN_PORT=$port -- $uvx mcp-obsidian
curl.exe -k -s -o NUL -w "%{http_code}`n" -H "Authorization: Bearer $key" https://127.0.0.1:$port/
claude mcp list
```

## Critérios de aceite

- [ ] `data.json` do plugin existe e contém `apiKey`
- [ ] `claude mcp list` mostra `mcp-obsidian` como conectado
- [ ] `curl` autenticado retorna HTTP 200
- [ ] Nenhum arquivo `.md` do vault foi modificado

---

## Fontes

- https://github.com/MarkusPfundstein/mcp-obsidian
- https://github.com/coddingtonbear/obsidian-local-rest-api
