# Calculadora de Custo de Receita

Aplicacao web em Vue 3 para calcular o custo proporcional de ingredientes em uma receita.

## MVP atual

- Cadastro centralizado de ingredientes com:
	- nome
	- preco pago
	- quantidade comprada e unidade
- Reuso de ingredientes em multiplas receitas
- Atualizacao global de ingrediente:
	- ao editar preco/quantidade comprada em uma receita, o ingrediente e atualizado no catalogo central
	- essa alteracao reflete automaticamente em todas as receitas que usam o ingrediente
- Calculo proporcional por ingrediente:
	- formula: `(preco base / quantidade base convertida) * quantidade da receita`
	- conversao automatica entre `kg <-> g` e `l <-> ml`
- Custo total da receita (somatorio dos ingredientes)
- Suporte a multiplas receitas no navegador (selecionar, criar e excluir)
- Persistencia local no navegador com `localStorage`
- Migracao automatica do formato antigo para o novo formato de dados
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

- Custo por porcao/fatia
- Exportacao da ficha de custo (PDF ou impressao)
- Historico de reajuste de preco dos ingredientes
