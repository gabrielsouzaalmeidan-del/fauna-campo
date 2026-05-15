# CLAUDE.md — WildLog

Guia completo do projeto para o Claude Code. Leia este arquivo antes de qualquer edição.

---

## O que é o WildLog

App Android de coleta de dados de fauna silvestre em campo, desenvolvido para biólogos e consultores ambientais brasileiros. Funciona **100% offline** (IndexedDB + SheetJS embutido), sem backend, sem login.

- **Plataforma:** Android (Capacitor + HTML/CSS/JS single-file)
- **Repositório:** `github.com/gabrielsouzaalmeidan-del/fauna-campo`
- **Build:** GitHub Actions (`.github/workflows/build-apk.yml`)
- **Arquivo principal:** `index.html` (~1.2MB, ~9.300 linhas)
- **Versão atual:** v2.0

---

## Arquitetura

```
fauna-campo/
├── index.html              ← ÚNICO ARQUIVO DO APP (tudo inline)
├── CLAUDE.md               ← Este arquivo
├── .github/
│   └── workflows/
│       └── build-apk.yml   ← Gera APK via GitHub Actions
└── (gerado pelo Capacitor)
    ├── android/
    └── package.json
```

### Por que um único arquivo?

O Capacitor copia `www/index.html` para o APK. O SheetJS (XLSX) está **embutido inline** no HTML para garantir funcionamento offline. Não há bundler, webpack ou build step — o HTML é o app.

---

## Estrutura do index.html

```
<head>
  <style> ← CSS completo (~1.500 linhas)
  <script> ← SheetJS embutido (~640KB)

<body>
  <div class="header">        ← Header fixo (logo, dark mode)
  <div id="ctx-bar">          ← Barra de contexto ativo
  <div id="offline-bar">      ← Indicador offline
  <div id="sync-indicator">   ← Toast de sync

  <!-- 7 PANELS (abas) -->
  <div class="panel" id="panel-escanear">
  <div class="panel" id="panel-manual">
  <div class="panel" id="panel-biometria">
  <div class="panel" id="panel-campanhas">
  <div class="panel" id="panel-dashboard">
  <div class="panel" id="panel-registros">
  <div class="panel" id="panel-enviar">

  <nav class="nav-bar">       ← Nav FORA dos panels (root level)
  <div id="sync-indicator">
  <div id="toast">

  <script> ← JS completo (~5.000 linhas)
```

### Regra crítica de estrutura HTML

A `<nav class="nav-bar">` DEVE estar no **root level** (depth=0), fora de qualquer `<div>`. Se ficar dentro de um panel (que tem `display:none`), a nav some da tela. Sempre verificar após edições:

```python
# Verificar depth antes da nav
nav_before = html[:html.find('<nav class="nav-bar">')]
# remover scripts e styles, contar <div vs </div> — deve ser 0
```

---

## Módulos do App (7 abas)

| Aba | Panel ID | Função principal |
|-----|----------|-----------------|
| 📷 Scan | `panel-escanear` | OCR de fichas via câmera + IA |
| 📝 Manual | `panel-manual` | Formulário de registro + MacKinnon inline |
| 📏 Bio | `panel-biometria` | Morfometria por grupo (5 sub-abas) |
| 🗂️ Camp. | `panel-campanhas` | Projetos, campanhas e pontos amostrais |
| 📊 Resumo | `panel-dashboard` | Dashboard gráfico (donut + barras) |
| 📋 Dados | `panel-registros` | Lista de registros com filtros rápidos |
| 📤 Enviar | `panel-enviar` | Exportação XLSX + perfil do consultor |

---

## Formulário Manual — Estrutura Crítica

```html
<div class="panel" id="panel-manual">
  <div class="card">
    <!-- Seção Projeto -->
    <!-- Seção Local e Tempo -->
    <!-- Seção Metodologia -->
    <div class="field-group">
      <label>Método</label>
      <select id="m-metodo" onchange="onMetodoChange(this.value)"
                            oninput="onMetodoChange(this.value)">
    </div>
    <div class="field-group" id="manual-tipo-group">  ← some no MacKinnon
      <label>Tipo de registro</label>
      <select id="m-tipo">
    </div>

    <!-- MacKinnon INLINE (antes do bio-wrap) -->
    <div id="mck-inline-block" style="display:none">
      ... módulo MacKinnon completo ...
    </div>

    <!-- Dados Biológicos (some no MacKinnon) -->
    <div id="manual-bio-wrap">
      <div class="card">
        <div class="card-title">
          <span id="bio-section-icon">🦜</span>
          <span id="bio-section-label"> Dados biológicos</span>
        </div>
        ... campos classe, ordem, familia, especie ...
      </div>
    </div>

  </div><!-- /card -->
</div><!-- /panel-manual -->
```

### onMetodoChange — lógica central

```javascript
function onMetodoChange(val) {
  const isMck = val === 'Listas de Mackinnon';
  // MacKinnon selecionado: mostrar mck, ocultar bio + tipo
  mckBlock.style.display  = isMck ? 'block' : 'none';
  biowrap.style.display   = isMck ? 'none'  : '';
  tipoGroup.style.display = isMck ? 'none'  : '';
  if (isMck) setTimeout(() => { mckIniciar(); ... }, 50);
}
```

**Esta função é chamada em 3 lugares:**
1. `onchange`/`oninput` do select `#m-metodo`
2. No `setTimeout` do `init()` ao restaurar o método salvo
3. Em `irPara('manual', ...)` ao navegar para a aba

---

## Funções JavaScript Críticas

### Inicialização
```javascript
async function init()              // ponto de entrada — carrega IDB, config, perfil
function irPara(panel, btn)        // navegação entre abas
function loadDarkMode()            // restaura preferência de dark mode
```

### Dados / IDB
```javascript
async function idbPut(store, obj)       // salvar no IndexedDB
async function idbGetAll(store)         // ler todos do store
async function salvarDados()            // persiste registros[]
async function carregarDados()          // carrega registros[] do IDB
function normalizarRegistro(r)          // padroniza campos (ATENÇÃO: renomeia horario→hora)
```

### ⚠️ Armadilha: normalizarRegistro

```javascript
// normalizarRegistro DELETA r.horario e cria r.hora
// gerarXLSX deve usar: r.hora || r.horario
if (n.horario !== undefined && n.hora === undefined) {
  n.hora = n.horario;
  delete n.horario;  // ← horario deletado!
}
```

### Formulário Manual
```javascript
async function adicionarManual()        // salva registro (usa await idbPut)
function autoFillFromSpecies(s)         // preenche classe/ordem/familia + ícone + status
function acOnInput(val)                 // autocomplete da espécie
function atualizarIconeBio(classe)      // atualiza ícone e label do card bio
function aplicarStatusConservacao(s)    // guarda _iucn/_mma no acCurrentMatch
```

### Exportação XLSX
```javascript
function baixarXLSX()    // verifica XLSX disponível + null check + try-catch
function gerarXLSX()     // monta workbook — retorna null em caso de erro
function nomeArquivo()   // ex: "FAUNA_2026-05-10.xlsx"
```

**Estrutura da planilha (27 colunas):**
```
PROJETO | FASE | CAMPANHA | ÁREA | PONTO | ZONA | UTM(X) | UTM(Y) |
DATA | HORÁRIO | PERÍODO | ESTAÇÃO | MÉTODO | TIPO REG. | DESTINAÇÃO |
MICROHABITAT | CLASSE | ORDEM | FAMÍLIA | ESPÉCIE | Nº INDIV. | SEXO |
IUCN | MMA | NOTÁVEL | CONSULTOR | OBSERVAÇÕES
```

### MacKinnon
```javascript
function mckIniciar()              // inicializa sessão MacKinnon
function mckAdicionarEspecie()     // adiciona espécie à lista atual
function mckFinalizarLista()       // encerra lista e inicia nova
function mckRenderIFL()            // calcula e exibe IFL
function mckCarregar()             // carrega dados do IDB
```

### Dashboard
```javascript
function renderDashboard()                    // render completo do painel
function renderGraficoDashboard(allReg, fn)   // coordena donut + barras
function renderDonutChart(dadosGrupo)         // canvas puro — try-catch
function renderBarrasPonto(dadosPonto)        // barras horizontais
function atualizarDashInicio()                // saudação + contadores + ações
```

### Registros
```javascript
function renderRegistrosFiltrado()   // renderiza com filtro ativo (filtroAtual)
function filtrarDados(filtro, chip)  // atualiza filtroAtual e re-renderiza
async function toggleNotavel(idx)    // ativa/desativa marcador notável + IDB
```

---

## IndexedDB — Stores

| Store | Conteúdo |
|-------|----------|
| `registros` | Registros manuais de fauna |
| `biometria` | Registros de biometria |
| `campanhas` | Campanhas criadas |
| `pontos` | Pontos amostrais |
| `config` | Configurações da sessão |
| `perfil` | Dados do consultor |

---

## CSS — Variáveis (definidas no :root)

```css
--verde-escuro: #1a2e1a   --verde-medio: #2d5a27   --verde-claro: #4a8c3f
--verde-suave: #e8f2e6    --verde-borda: #b8d4b3   --borda: #b8d4b3
--areia: #f5f0e8          --areia-escura: #e8dfc8  --fundo: var(--areia)
--texto: #1a2e1a          --texto-suave: #4a6b47   --texto-muted: #7a9977
--branco: #ffffff         --sombra: 0 1px 3px ...  --radius: 10px
--radius-sm: 7px          --perigo: #c0392b        --sucesso: #2d5a27
--alerta: #8b6914
```

**⚠️ Não adicionar novos `var(--x)` sem definir no `:root` — causa campos sem borda.**

---

## Regras de Validação

Antes de qualquer commit, verificar:

```python
import re, subprocess

with open("index.html") as f:
    html = f.read()

# 1. JS Sintaxe
scripts = re.findall(r'<script[^>]*>(.*?)</script>', html, re.DOTALL)
js = '\n'.join(scripts)
with open("/tmp/wl.js","w") as f: f.write(js)
r = subprocess.run(['node','--check','/tmp/wl.js'], capture_output=True, text=True)
assert r.returncode == 0, "JS com erro de sintaxe!"

# 2. HTML depth
html_only = re.sub(r'<script[^>]*>.*?</script>', '', html, flags=re.DOTALL)
html_only = re.sub(r'<style[^>]*>.*?</style>', '', html_only, flags=re.DOTALL)
d = 0; i = 0
while i < len(html_only):
    if html_only[i:i+4] == '<div': d += 1; i += 4
    elif html_only[i:i+6] == '</div>': d -= 1; i += 6
    else: i += 1
assert d == 0, f"HTML com divs desbalanceados: depth={d}"

# 3. Nav no root
nav_b = html[:html.find('<nav class="nav-bar">')]
nav_b = re.sub(r'<script[^>]*>.*?</script>', '', nav_b, flags=re.DOTALL)
nav_b = re.sub(r'<style[^>]*>.*?</style>', '', nav_b, flags=re.DOTALL)
nd = nav_b.count('<div') - nav_b.count('</div>')
assert nd == 0, f"Nav não está no root! depth={nd}"

print("✅ Tudo OK")
```

---

## Bugs Históricos (não repetir)

| Bug | Causa | Fix |
|-----|-------|-----|
| Nav sumia | `<nav>` dentro de um `<div class="panel">` | Mover nav para root level |
| Exportação vazia | `r.horario` deletado por `normalizarRegistro` | Usar `r.hora \|\| r.horario` |
| Exportação travava | `XLSX.writeFile(null, ...)` sem null-check | Verificar `if (!wb) return` |
| MacKinnon não aparecia | `onMetodoChange` não chamado no carregamento | Chamar no `init()` e no `irPara()` |
| Campos sem borda | `var(--borda)` usada mas não definida no `:root` | Sempre definir vars no `:root` |
| `async async function salvarBio` | Fix de async aplicado 2x | Verificar duplicatas |
| Colunas XLSX desalinhadas | `cabecalho2` com 23 cols, `return` com 24 | Sempre contar colunas = campos |
| panel-manual não fechava | Remoção de bloco MacKinnon levou o `</div>` | Verificar depth após mover blocos |

---

## Features Implementadas (v2.0)

- ✅ Dashboard gráfico (donut chart canvas puro + barras por ponto)
- ✅ Entrada por voz (Web Speech API + getUserMedia para permissão Android)
- ✅ Histórico de espécies recentes (chips clicáveis)
- ✅ Filtros rápidos na aba Dados (7 chips: Todos/Aves/Mammalia/Répteis/Anfíb/Peixes/Notáveis)
- ✅ Marcador de espécie notável (⭐ persistido no IDB)
- ✅ Badge IUCN + MMA automático (lido do SPECIES_DB campos `iu` e `mm`)
- ✅ MacKinnon integrado como método inline no formulário Manual
- ✅ Ícone dinâmico por Classe nos Dados Biológicos
- ✅ Barra de contexto ativo (ctx-bar)
- ✅ Barra offline inteligente (conta registros pendentes)
- ✅ Cards de registro ricos (ícone por grupo, badges, notável, IUCN/MMA)
- ✅ SheetJS embutido inline (sem dependência de arquivo externo)
- ✅ GPS nativo Android (Capacitor) com fallback web
- ✅ Dark mode completo (todos os componentes)
- ✅ RECORD_AUDIO no AndroidManifest (permissão de microfone)

## Pendências

- [ ] Timer de esforço amostral (cronômetro por ponto)
- [ ] Compartilhar registro via Web Share API
- [ ] Modularizar o index.html (separar em arquivos por feature)
- [ ] Verificar "WildLog" no INPI (Classe 09 e 42)

---

## Build APK

O workflow `.github/workflows/build-apk.yml`:
1. Cria `www/index.html` a partir do `index.html` da raiz
2. Instala e copia SheetJS para `www/xlsx.full.min.js` (fallback)
3. Instala Capacitor e adiciona plataforma Android
4. Compila com Gradle e gera `WildLog.apk`
5. Publica como GitHub Release

Para gerar novo APK: fazer push na branch `main` ou disparar manualmente em **Actions → Run workflow**.

---

## Testes (Playwright)

```bash
# Instalar
pip install playwright
playwright install chromium

# Rodar
python3 tests/test_wildlog.py
```

Resultado esperado: **36 PASS, 0 FAIL**.

Blocos testados: carregamento, nav, formulário manual, MacKinnon inline, ícone dinâmico, dashboard gráfico, filtros, biometria, campanhas, exportação XLSX, dark mode, autocomplete.

---

## Contato

Desenvolvido por **Gabriel Almeida** — biólogo e consultor ambiental, Ceará/BR.
