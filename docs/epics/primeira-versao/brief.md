# Primeira versão do Grifo

## Problema
O Bruno usa há muito tempo um tradutor de texto selecionado (Crow Translate): seleciona um
texto em qualquer programa, aperta um atalho e vê a tradução. Ele quer o mesmo jeito de
trabalhar, mas com várias ações sobre o texto selecionado, não só traduzir.

Este repositório é um fork do Pot (pot-app/pot-desktop, GPL-3.0, Tauri 1 + React). O Pot já
captura o texto selecionado, registra atalhos globais e traduz. O repositório original está
arquivado, então não haverá atualizações vindas de lá: este fork segue por conta própria.

## Resultado esperado
O Bruno instala o app no Windows e ele aparece como **Grifo** (não como "pot"). Ao selecionar
um texto em qualquer programa e apertar **Ctrl+Alt+Espaço**, abre uma paleta com uma busca e
uma lista de ações. Ele escolhe uma ação e vê o resultado na hora:

- **Traduzir**: abre a janela de tradução que o Pot já tem, com o texto selecionado.
- **Maiúsculas** / **Minúsculas**: mostra o texto convertido, com botão Copiar.
- **Remover quebras de linha**: junta as linhas num parágrafo só, com botão Copiar.
- **Pesquisar no Google**: abre o navegador padrão com a busca.

Tudo o que o Pot já fazia (tradução por atalho, OCR, configurações) continua funcionando.

## Escopo
- Incluído:
  - Renomear o app de Pot para Grifo (nome, identificador, títulos) e desligar o
    atualizador automático do Pot.
  - Estrutura de testes automáticos do front-end (vitest).
  - Um módulo de ações em Rust com as 5 ações acima.
  - A janela "Paleta de comandos", aberta por atalho global configurável
    (padrão Ctrl+Alt+Espaço), com busca e navegação por teclado.
  - A exibição do resultado na própria paleta, com botão Copiar.
- Fora de escopo:
  - Barra de ícones flutuante sobre a seleção.
  - Tela de configuração das ações (ligar/desligar, reordenar, criar ações próprias).
  - Ações com IA (corrigir, explicar, resumir) e qualquer chave de API nova.
  - Botão "Substituir" (colar o resultado no lugar da seleção).
  - Builds para macOS e Linux, instalador, assinatura de código e qualquer mudança em
    `.github/` (feitos fora do pipeline, com o Bruno).
  - Migração para Tauri 2.

## Critérios de aceite do épico
- Dado o app instalado, quando ele é aberto, então o nome exibido na bandeja, nas janelas e
  no instalador é "Grifo" e ele não tenta se atualizar a partir dos servidores do Pot.
- Dado um texto selecionado em outro programa, quando o Bruno aperta Ctrl+Alt+Espaço, então
  abre a janela da paleta mostrando o início do texto selecionado e a lista de ações.
- Dado a paleta aberta, quando ele digita "maiu", então só "Maiúsculas" aparece na lista
  (a busca ignora acentos e maiúsculas/minúsculas).
- Dado a paleta aberta com o texto "ação rápida", quando ele escolhe "Maiúsculas", então a
  paleta mostra "AÇÃO RÁPIDA" e um botão Copiar que copia esse texto.
- Dado a paleta aberta, quando ele escolhe "Traduzir", então a paleta fecha e a janela de
  tradução do Pot abre com o texto selecionado.
- Dado a paleta aberta, quando ele escolhe "Pesquisar no Google", então o navegador padrão
  abre em `https://www.google.com/search?q=<texto>` e a paleta fecha.
- Dado a paleta aberta, quando ele aperta Esc, então a paleta fecha.

## Como testar no fim
1. Baixar e instalar o instalador do Windows gerado a partir deste repositório (o build do
   instalador é configurado com o Bruno, fora do pipeline).
2. Abrir o Grifo e conferir que o ícone da bandeja e a janela de configurações dizem "Grifo".
3. Abrir o Bloco de Notas, escrever "ação rápida em português" e selecionar o texto.
4. Apertar Ctrl+Alt+Espaço: a paleta abre mostrando o texto selecionado.
5. Digitar "maiu", apertar Enter: aparece "AÇÃO RÁPIDA EM PORTUGUÊS". Clicar em Copiar e
   colar no Bloco de Notas.
6. Selecionar de novo, Ctrl+Alt+Espaço, escolher "Traduzir": abre a janela de tradução.
7. Selecionar de novo, Ctrl+Alt+Espaço, escolher "Pesquisar no Google": abre o navegador.
8. Abrir a paleta e apertar Esc: ela fecha.

## Restrições
- Manter Tauri 1 e a estrutura atual do Pot; mudar o mínimo de arquivos existentes.
- Não renomear o crate Rust (`pot`) nem pastas: só nomes exibidos e o identificador.
- Não mexer em `.github/`, assinatura, segredos ou serviços pagos.
- Manter o arquivo `LICENSE` (GPL-3.0) e o crédito ao Pot no README.
- Textos novos da interface em português do Brasil (a i18n completa fica para depois).
