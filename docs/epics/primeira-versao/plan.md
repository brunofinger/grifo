# Plano técnico — Primeira versão do Grifo

## Desenho

### Ponto de partida (código do Pot, já existente)
- Tauri **1.8**, back-end em `src-tauri/src/` (Rust), front-end React + Vite em `src/` (JSX).
- Cada janela é criada por `build_window(label, title)` em `src-tauri/src/window.rs` e
  carrega `index.html`; `src/App.jsx` escolhe o componente pelo rótulo da janela
  (`windowMap[appWindow.label]`).
- `src/App.jsx` já fecha qualquer janela com **Esc** (listener global de `keydown`).
- Atalhos globais: `src-tauri/src/hotkey.rs` (`register`, `register_shortcut("all")`,
  `register_shortcut_by_frontend`), com o valor salvo na config pelo nome
  (ex.: `hotkey_selection_translate`), lido por `config::get`/`config::set`.
- Texto selecionado: crate `selection` (`selection::get_text()`), usado em
  `selection_translate()` em `window.rs`.
- Tradução de um texto arbitrário: `text_translate(text: String)` em `window.rs`
  (é o que `server.rs` usa em `POST /translate`).
- Comandos Tauri são registrados em `tauri::generate_handler![...]` em `src-tauri/src/main.rs`.
- Página de atalhos das configurações: `src/window/Config/pages/Hotkey/`.

### Identidade do app (T1)
`src-tauri/tauri.conf.json`:
- `package.productName`: `"Grifo"`
- `tauri.bundle.identifier`: `"com.brunofinger.grifo"`
- `tauri.bundle.shortDescription`: `"Grifo"`; `longDescription`: `"Ações rápidas sobre o texto selecionado"`
- `tauri.updater.active`: `false` (o atualizador apontava para os releases do Pot)

Também trocar textos visíveis "Pot"/"pot" por "Grifo" no tooltip da bandeja e na notificação
de instância única em `main.rs` (`"The program is already running..."` pode ficar em inglês).
Não mexer no nome do crate, nos ícones nem nas traduções i18n existentes.

### Testes do front-end (T1)
- devDependencies: `vitest`, `@testing-library/react`, `@testing-library/jest-dom`, `jsdom`.
- `package.json`: script `"test": "vitest run"`.
- `vite.config.js`: bloco `test: { environment: 'jsdom', globals: true, setupFiles: ['./src/test/setup.js'] }`.
- `src/test/setup.js`: importa `@testing-library/jest-dom`.
- `src/test/__mocks__/tauri.js` (ou `vi.mock` por teste) para simular `@tauri-apps/api`
  (`invoke`, `appWindow.close`, `clipboard.writeText`) — o jsdom não tem o runtime do Tauri.
- Primeiro teste: `src/test/appIdentity.test.js` lê `src-tauri/tauri.conf.json` e verifica os
  quatro campos da identidade.

### Módulo de ações (T2) — `src-tauri/src/actions.rs`
```rust
#[derive(Serialize, Clone, Debug, PartialEq)]
#[serde(rename_all = "snake_case")]
pub enum ActionKind { Translate, Uppercase, Lowercase, RemoveLineBreaks, WebSearch }

#[derive(Serialize, Clone, Debug)]
pub struct Action { pub key: String, pub name: String, pub description: String, pub kind: ActionKind }

#[derive(Serialize, Debug, PartialEq)]
#[serde(tag = "type", content = "value", rename_all = "snake_case")]
pub enum ActionOutput { Text(String), OpenUrl(String), Translate(String) }

pub fn default_actions() -> Vec<Action>;
pub fn run_action(key: &str, text: &str) -> Result<ActionOutput, String>; // função pura, sem efeitos
pub fn remove_line_breaks(text: &str) -> String;
pub fn search_url(text: &str) -> String;

#[tauri::command] pub fn list_actions() -> Vec<Action>;
#[tauri::command] pub fn execute_action(key: String, text: String) -> Result<ActionOutput, String>;
```
Ações padrão, nesta ordem (chave · nome · descrição):
1. `translate` · Traduzir · Traduz com os serviços configurados
2. `uppercase` · Maiúsculas · Converte para MAIÚSCULAS
3. `lowercase` · Minúsculas · Converte para minúsculas
4. `remove_line_breaks` · Remover quebras de linha · Junta as linhas num parágrafo
5. `web_search` · Pesquisar no Google · Abre a busca no navegador

Regras:
- `uppercase`/`lowercase`: `str::to_uppercase`/`to_lowercase` (Unicode: "ação" → "AÇÃO").
- `remove_line_breaks`: troca `\r\n`, `\n` e `\r` por espaço, colapsa espaços repetidos em um,
  `trim()` nas pontas.
- `search_url`: `https://www.google.com/search?q=` + texto codificado, usando
  `reqwest::Url::parse_with_params` (já é dependência; não adicionar crate de URL).
- Texto vazio ou só espaços → `Err("Nenhum texto selecionado")`.
- Chave desconhecida → `Err("Ação desconhecida: <chave>")`.
- `execute_action` chama `run_action` e aplica os efeitos:
  `OpenUrl(url)` → `tauri::api::shell::open(&app.shell_scope(), url, None)`;
  `Translate(text)` → `text_translate(text)`; depois devolve o mesmo `ActionOutput` ao front.
- Registrar `mod actions;` e os dois comandos em `main.rs`.

### Janela da paleta (T3)
Back-end:
- `palette_window()` em `window.rs`: lê `selection::get_text()` **antes** de criar/mostrar a
  janela, guarda o texto num estado gerenciado `PaletteText(Mutex<String>)` (registrado com
  `app.manage` no `setup`), cria a janela via `build_window("palette", "Paleta")`,
  tamanho lógico 560×440, centralizada no monitor do mouse, `always_on_top(true)`,
  `skip_taskbar(true)`, e emite o evento `palette_text` com o texto para a janela.
- Comando `get_palette_text() -> String` (o front chama ao montar, para não depender do evento).
- Atalho: nome de config `hotkey_palette`. Na primeira execução, se não existir, gravar
  `"CommandOrControl+Alt+Space"`. Registrar em `register_shortcut("all")`, aceitar o nome em
  `register_shortcut_by_frontend` e tratar o caso `"hotkey_palette"` em `register_shortcut`.
- Na página `src/window/Config/pages/Hotkey/`, adicionar um campo "Paleta de comandos"
  idêntico aos existentes, ligado a `hotkey_palette`.

Front-end — `src/window/Palette/`:
- `filter.js`: `normalize(s)` (minúsculas + remove acentos via `NFD` e
  `/[̀-ͯ]/g`) e `filterActions(actions, query)` — query vazia devolve todas na
  ordem original; caso contrário, mantém as ações cujo `normalize(name + ' ' + description)`
  contém `normalize(query.trim())`.
- `index.jsx`: componente `Palette`, registrado em `windowMap` de `src/App.jsx` como `palette`.

Tela da paleta (estado "lista"):
- Campo de busca no topo, com foco automático, placeholder "Buscar ação…".
- Faixa com o texto selecionado: primeiros 60 caracteres + "…" quando maior, e
  "· N caracteres".
- Lista de ações filtradas: nome em destaque e descrição abaixo; o primeiro item já vem
  destacado.
- Seta para baixo/cima move o destaque (sem passar das pontas); Enter executa o destacado;
  clique executa o clicado.
- Estado vazio da busca: "Nenhuma ação encontrada para “<busca>”".
- Sem texto selecionado: faixa mostra "Nenhum texto selecionado — selecione um texto e
  aperte o atalho de novo." e Enter/clique não executam nada.
- Esc fecha (já garantido pelo listener do `App.jsx`).

### Resultado na paleta (T4)
Ao executar, o front chama `invoke('execute_action', { key, text })`:
- `{ type: 'text', value }` → estado "resultado": título com o nome da ação, o texto em
  bloco com `white-space: pre-wrap`, botões **Copiar** (usa `writeText` de
  `@tauri-apps/api/clipboard`; o rótulo vira "Copiado" por 1,5 s) e **Voltar** (volta à
  lista com a busca limpa).
- `{ type: 'open_url' }` ou `{ type: 'translate' }` → `appWindow.close()`.
- Erro (promise rejeitada) → mensagem "Não foi possível executar: <erro>" acima da lista, e
  a paleta continua aberta.
- Enquanto espera a resposta, o item clicado mostra "Executando…" e novos cliques são
  ignorados.

## Decisões
| Decisão | Alternativas | Motivo |
|---|---|---|
| Fork do Pot | Crow Translate (C++/Qt); app do zero | Pot já é Tauri, roda nos 3 sistemas e já traduz e captura seleção |
| Manter Tauri 1 | Migrar para Tauri 2 | Migração é grande e não traz nada para esta versão |
| Ações definidas em Rust (lista fixa) | Arquivos de configuração editáveis | Configuração pelo usuário fica para o épico da tela de ações |
| `run_action` puro + efeitos em `execute_action` | Efeitos dentro de cada ação | Permite testar todas as ações com `cargo test` sem abrir janelas |
| Resultado mostrado dentro da paleta | Janela de resultado separada | Menos janelas para criar agora; dá para separar depois |
| vitest + Testing Library | Jest | Integra direto com o Vite que o projeto já usa |
| Identificador novo `com.brunofinger.grifo` | Manter `com.pot-app.desktop` | Permite instalar junto com o Pot original e não mistura configurações |

## Riscos
- Compilar o crate Tauri na VPS exige bibliotecas do sistema (webkit2gtk, libayatana,
  libxdo etc.) → o ambiente da VPS precisa delas antes do T1; `cargo test` cobre só a
  lógica pura, não abre janelas.
- O front do Pot não tem testes e importa APIs do Tauri que não existem no jsdom → mockar
  `@tauri-apps/api` e `tauri-plugin-*-api` nos testes; testar componentes isolados, não o
  `App.jsx` inteiro.
- `selection::get_text()` pode demorar ou falhar em alguns programas → tratar erro como texto
  vazio (estado "Nenhum texto selecionado"), nunca travar a abertura da paleta.
- Ctrl+Alt+Espaço pode estar ocupado por outro programa → falha de registro já gera
  notificação no Pot; o atalho é editável na página de atalhos.
- O workflow `.github/workflows/package.yml` herdado do Pot depende de segredos que não
  existem → não mexer nele neste épico; o build do instalador é ajustado com o Bruno.

## Ordem das tarefas
T1 → T2 → T3 → T4
