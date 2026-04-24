# Calculadora de Custo de Receita

Aplicacao web em Vue 3 para calcular o custo proporcional de ingredientes em uma receita.

## MVP atual

- Cadastro rapido de ingredientes com:
	- nome
	- preco pago
	- quantidade comprada e unidade
	- quantidade usada na receita e unidade
- Calculo proporcional por ingrediente:
	- formula: `(preco base / quantidade base) * quantidade da receita`
- Custo total da receita (somatorio dos ingredientes)
- Suporte a multiplas receitas no navegador (selecionar, criar e excluir)
- Persistencia local no navegador com `localStorage`
- Interface em portugues e responsiva para desktop e mobile

## Tecnologias

- Vue 3
- Vite
- JavaScript

## Como executar

1. Instale as dependencias:

```bash
npm install
```

2. Rode em modo desenvolvimento:

```bash
npm run dev
```

3. Para gerar build de producao:

```bash
npm run build
```

4. Para visualizar a build local:

```bash
npm run preview
```

## Observacao sobre Node.js

Durante o scaffold foram exibidos avisos de engine para a versao local do Node.js.
Recomendado usar Node.js `20.19+` (ou `22.12+`) para evitar warnings do Vite mais recente.

## Proximos passos sugeridos

- Cadastro de ingredientes base reutilizaveis
- Lista de receitas salvas
- Custo por porcao/fatia
