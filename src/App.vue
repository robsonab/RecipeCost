<script setup>
import { computed, reactive, ref, watch } from 'vue'

const STORAGE_KEY = 'recipe-cost-calculator-v2'
const moeda = 'BRL'

const unidadesPadrao = ['g', 'kg', 'ml', 'l', 'un']

const grupoUnidade = {
  g: 'massa',
  kg: 'massa',
  ml: 'volume',
  l: 'volume',
  un: 'unidade',
}

const fatorConversao = {
  g: 1,
  kg: 1000,
  ml: 1,
  l: 1000,
  un: 1,
}

const receitas = ref([])
const receitaAtivaId = ref('')
const ingredientesCatalogo = ref([])
const abaAtiva = ref('receita')

const novoIngredienteForm = reactive({
  nome: '',
  precoBase: '',
  quantidadeBase: '',
  unidadeBase: 'kg',
})

const itemForm = reactive({
  ingredienteId: '',
  quantidadeReceita: '',
  unidadeReceita: 'kg',
})

const mensagemErro = ref('')
const itemEmEdicaoId = ref(null)
const confirmacaoExclusao = reactive({
  aberta: false,
  tipo: '',
  alvoId: '',
  titulo: '',
  mensagem: '',
})

const itemEdicaoForm = reactive({
  quantidadeReceita: '',
  unidadeReceita: 'kg',
  nomeIngrediente: '',
  precoBase: '',
  quantidadeBase: '',
  unidadeBase: 'kg',
})

const criarReceita = (nome = 'Receita 1') => ({
  id: crypto.randomUUID(),
  nome,
  itens: [],
})

const formatarMoeda = (valor) =>
  new Intl.NumberFormat('pt-BR', {
    style: 'currency',
    currency: moeda,
    minimumFractionDigits: 2,
  }).format(valor)

const unidadesCompativeis = (unidadeBase, unidadeReceita) =>
  grupoUnidade[unidadeBase] === grupoUnidade[unidadeReceita]

const converterQuantidade = (quantidade, unidadeOrigem, unidadeDestino) => {
  if (!unidadesCompativeis(unidadeOrigem, unidadeDestino)) return null

  const quantidadeNormalizada = quantidade * fatorConversao[unidadeOrigem]
  return quantidadeNormalizada / fatorConversao[unidadeDestino]
}

const receitaAtiva = computed(() =>
  receitas.value.find((receita) => receita.id === receitaAtivaId.value),
)

const itensAtivos = computed(() => receitaAtiva.value?.itens || [])

const nomeReceitaAtual = computed({
  get: () => receitaAtiva.value?.nome || '',
  set: (novoNome) => {
    if (!receitaAtiva.value) return
    receitaAtiva.value.nome = novoNome
  },
})

const ingredienteSelecionado = computed(() =>
  ingredientesCatalogo.value.find(
    (ingrediente) => ingrediente.id === itemForm.ingredienteId,
  ),
)

const obterIngredienteDoItem = (item) =>
  ingredientesCatalogo.value.find(
    (catalogo) => catalogo.id === item.ingredienteId,
  )

const custoDoItem = (item) => {
  const ingrediente = obterIngredienteDoItem(item)
  if (!ingrediente) return 0

  const precoBase = Number(ingrediente.precoBase)
  const quantidadeBase = Number(ingrediente.quantidadeBase)
  const quantidadeReceita = Number(item.quantidadeReceita)

  if (precoBase <= 0 || quantidadeBase <= 0 || quantidadeReceita < 0) {
    return 0
  }

  const quantidadeBaseConvertida = converterQuantidade(
    quantidadeBase,
    ingrediente.unidadeBase,
    item.unidadeReceita,
  )

  if (!quantidadeBaseConvertida || quantidadeBaseConvertida <= 0) {
    return 0
  }

  return (precoBase / quantidadeBaseConvertida) * quantidadeReceita
}

const totalReceita = computed(() =>
  itensAtivos.value.reduce((acc, item) => acc + custoDoItem(item), 0),
)

const limparNovoIngredienteForm = () => {
  novoIngredienteForm.nome = ''
  novoIngredienteForm.precoBase = ''
  novoIngredienteForm.quantidadeBase = ''
  novoIngredienteForm.unidadeBase = 'kg'
}

const limparItemForm = () => {
  itemForm.ingredienteId = ''
  itemForm.quantidadeReceita = ''
  itemForm.unidadeReceita = 'kg'
}

const adicionarIngredienteCatalogo = () => {
  const nome = novoIngredienteForm.nome.trim()
  const precoBase = Number(novoIngredienteForm.precoBase)
  const quantidadeBase = Number(novoIngredienteForm.quantidadeBase)

  if (!nome) {
    mensagemErro.value = 'Informe o nome do ingrediente.'
    return
  }

  if (precoBase <= 0 || quantidadeBase <= 0) {
    mensagemErro.value =
      'Preco base e quantidade comprada devem ser maiores que zero.'
    return
  }

  ingredientesCatalogo.value.push({
    id: crypto.randomUUID(),
    nome,
    precoBase,
    quantidadeBase,
    unidadeBase: novoIngredienteForm.unidadeBase,
  })

  if (!itemForm.ingredienteId) {
    itemForm.ingredienteId = ingredientesCatalogo.value.at(-1)?.id || ''
  }

  limparNovoIngredienteForm()
  mensagemErro.value = ''
}

const removerIngredienteCatalogo = (id) => {
  const estaEmUso = receitas.value.some((receita) =>
    receita.itens.some((item) => item.ingredienteId === id),
  )

  if (estaEmUso) {
    mensagemErro.value =
      'Nao e possivel excluir: o ingrediente esta sendo usado em pelo menos uma receita.'
    return
  }

  ingredientesCatalogo.value = ingredientesCatalogo.value.filter(
    (ingrediente) => ingrediente.id !== id,
  )

  if (itemForm.ingredienteId === id) {
    itemForm.ingredienteId = ''
  }

  mensagemErro.value = ''
}

const abrirConfirmacaoIngrediente = (id) => {
  const ingrediente = ingredientesCatalogo.value.find((item) => item.id === id)
  if (!ingrediente) return

  confirmacaoExclusao.aberta = true
  confirmacaoExclusao.tipo = 'ingrediente'
  confirmacaoExclusao.alvoId = id
  confirmacaoExclusao.titulo = 'Excluir ingrediente'
  confirmacaoExclusao.mensagem = `Deseja realmente excluir "${ingrediente.nome}" do cadastro central?`
}

const adicionarItemReceita = () => {
  if (!receitaAtiva.value) return

  const ingrediente = ingredienteSelecionado.value
  const quantidadeReceita = Number(itemForm.quantidadeReceita)

  if (!ingrediente) {
    mensagemErro.value = 'Selecione um ingrediente do cadastro central.'
    return
  }

  if (quantidadeReceita < 0) {
    mensagemErro.value = 'Quantidade da receita deve ser maior ou igual a zero.'
    return
  }

  if (!unidadesCompativeis(ingrediente.unidadeBase, itemForm.unidadeReceita)) {
    mensagemErro.value =
      'As unidades precisam ser compativeis: kg/g, l/ml ou un com un.'
    return
  }

  const itemJaExiste = receitaAtiva.value.itens.some(
    (item) => item.ingredienteId === ingrediente.id,
  )

  if (itemJaExiste) {
    mensagemErro.value =
      'Esse ingrediente ja foi adicionado na receita ativa. Edite o item existente.'
    return
  }

  receitaAtiva.value.itens.push({
    id: crypto.randomUUID(),
    ingredienteId: ingrediente.id,
    quantidadeReceita,
    unidadeReceita: itemForm.unidadeReceita,
  })

  limparItemForm()
  mensagemErro.value = ''
}

const limparReceita = () => {
  if (!receitaAtiva.value) return

  receitaAtiva.value.itens = []
  itemEmEdicaoId.value = null
}

const adicionarReceita = () => {
  const novaReceita = criarReceita(`Receita ${receitas.value.length + 1}`)
  receitas.value.push(novaReceita)
  receitaAtivaId.value = novaReceita.id
  itemEmEdicaoId.value = null
  mensagemErro.value = ''
}

const excluirReceita = (id) => {
  receitas.value = receitas.value.filter((receita) => receita.id !== id)

  if (receitas.value.length === 0) {
    const receitaInicial = criarReceita('Receita 1')
    receitas.value.push(receitaInicial)
  }

  receitaAtivaId.value = receitas.value[0].id
  itemEmEdicaoId.value = null
}

const abrirConfirmacaoReceita = () => {
  if (!receitaAtiva.value) return

  confirmacaoExclusao.aberta = true
  confirmacaoExclusao.tipo = 'receita'
  confirmacaoExclusao.alvoId = receitaAtiva.value.id
  confirmacaoExclusao.titulo = 'Excluir receita'
  confirmacaoExclusao.mensagem = `Deseja realmente excluir a receita "${receitaAtiva.value.nome}"?`
}

const cancelarConfirmacaoExclusao = () => {
  confirmacaoExclusao.aberta = false
  confirmacaoExclusao.tipo = ''
  confirmacaoExclusao.alvoId = ''
  confirmacaoExclusao.titulo = ''
  confirmacaoExclusao.mensagem = ''
}

const confirmarExclusao = () => {
  if (confirmacaoExclusao.tipo === 'ingrediente') {
    removerIngredienteCatalogo(confirmacaoExclusao.alvoId)
  }

  if (confirmacaoExclusao.tipo === 'receita') {
    excluirReceita(confirmacaoExclusao.alvoId)
  }

  cancelarConfirmacaoExclusao()
}

const iniciarEdicaoItem = (item) => {
  const ingrediente = obterIngredienteDoItem(item)
  if (!ingrediente) return

  itemEmEdicaoId.value = item.id
  itemEdicaoForm.quantidadeReceita = String(item.quantidadeReceita)
  itemEdicaoForm.unidadeReceita = item.unidadeReceita
  itemEdicaoForm.nomeIngrediente = ingrediente.nome
  itemEdicaoForm.precoBase = String(ingrediente.precoBase)
  itemEdicaoForm.quantidadeBase = String(ingrediente.quantidadeBase)
  itemEdicaoForm.unidadeBase = ingrediente.unidadeBase
  mensagemErro.value = ''
}

const cancelarEdicaoItem = () => {
  itemEmEdicaoId.value = null
  mensagemErro.value = ''
}

const salvarEdicaoItem = (itemId) => {
  if (!receitaAtiva.value) return

  const item = receitaAtiva.value.itens.find((itemAtual) => itemAtual.id === itemId)
  if (!item) return

  const ingrediente = obterIngredienteDoItem(item)
  if (!ingrediente) return

  const nomeIngrediente = itemEdicaoForm.nomeIngrediente.trim()
  const precoBase = Number(itemEdicaoForm.precoBase)
  const quantidadeBase = Number(itemEdicaoForm.quantidadeBase)
  const quantidadeReceita = Number(itemEdicaoForm.quantidadeReceita)

  if (!nomeIngrediente) {
    mensagemErro.value = 'Informe o nome do ingrediente.'
    return
  }

  if (precoBase <= 0 || quantidadeBase <= 0 || quantidadeReceita < 0) {
    mensagemErro.value =
      'Revise os valores: preco base e quantidade base devem ser maiores que zero.'
    return
  }

  if (!unidadesCompativeis(itemEdicaoForm.unidadeBase, itemEdicaoForm.unidadeReceita)) {
    mensagemErro.value =
      'As unidades precisam ser compativeis: kg/g, l/ml ou un com un.'
    return
  }

  ingrediente.nome = nomeIngrediente
  ingrediente.precoBase = precoBase
  ingrediente.quantidadeBase = quantidadeBase
  ingrediente.unidadeBase = itemEdicaoForm.unidadeBase

  item.quantidadeReceita = quantidadeReceita
  item.unidadeReceita = itemEdicaoForm.unidadeReceita

  itemEmEdicaoId.value = null
  mensagemErro.value = ''
}

const removerItemReceita = (itemId) => {
  if (!receitaAtiva.value) return

  receitaAtiva.value.itens = receitaAtiva.value.itens.filter(
    (item) => item.id !== itemId,
  )

  if (itemEmEdicaoId.value === itemId) {
    itemEmEdicaoId.value = null
  }
}

const migrarFormatoAntigo = (parsed) => {
  const catalogo = []
  const indicePorNome = new Map()

  const obterOuCriarIngrediente = (itemAntigo) => {
    const chave = `${(itemAntigo.nome || '').trim().toLowerCase()}-${itemAntigo.unidadeBase || 'kg'}`

    if (indicePorNome.has(chave)) {
      return indicePorNome.get(chave)
    }

    const novo = {
      id: crypto.randomUUID(),
      nome: itemAntigo.nome || 'Ingrediente',
      precoBase: Number(itemAntigo.precoBase) || 0,
      quantidadeBase: Number(itemAntigo.quantidadeBase) || 0,
      unidadeBase: itemAntigo.unidadeBase || 'kg',
    }

    catalogo.push(novo)
    indicePorNome.set(chave, novo)
    return novo
  }

  if (Array.isArray(parsed.receitas)) {
    const receitasMigradas = parsed.receitas.map((receita) => ({
      id: receita.id || crypto.randomUUID(),
      nome: receita.nome || 'Receita',
      itens: Array.isArray(receita.ingredientes)
        ? receita.ingredientes.map((item) => {
            const ingrediente = obterOuCriarIngrediente(item)
            return {
              id: item.id || crypto.randomUUID(),
              ingredienteId: ingrediente.id,
              quantidadeReceita: Number(item.quantidadeReceita) || 0,
              unidadeReceita: item.unidadeReceita || ingrediente.unidadeBase,
            }
          })
        : [],
    }))

    return {
      receitas: receitasMigradas,
      ingredientesCatalogo: catalogo,
      receitaAtivaId: receitasMigradas[0]?.id || '',
    }
  }

  if (parsed.receitaNome || Array.isArray(parsed.ingredientes)) {
    const itens = Array.isArray(parsed.ingredientes)
      ? parsed.ingredientes.map((item) => {
          const ingrediente = obterOuCriarIngrediente(item)
          return {
            id: item.id || crypto.randomUUID(),
            ingredienteId: ingrediente.id,
            quantidadeReceita: Number(item.quantidadeReceita) || 0,
            unidadeReceita: item.unidadeReceita || ingrediente.unidadeBase,
          }
        })
      : []

    const receita = {
      id: crypto.randomUUID(),
      nome: parsed.receitaNome || 'Receita 1',
      itens,
    }

    return {
      receitas: [receita],
      ingredientesCatalogo: catalogo,
      receitaAtivaId: receita.id,
    }
  }

  return null
}

const salvarNoNavegador = () => {
  const payload = {
    receitas: receitas.value,
    receitaAtivaId: receitaAtivaId.value,
    ingredientesCatalogo: ingredientesCatalogo.value,
  }

  localStorage.setItem(STORAGE_KEY, JSON.stringify(payload))
}

const inicializarEstadoPadrao = () => {
  if (receitas.value.length === 0) {
    const receitaInicial = criarReceita('Receita 1')
    receitas.value = [receitaInicial]
    receitaAtivaId.value = receitaInicial.id
  }

  if (!receitas.value.some((receita) => receita.id === receitaAtivaId.value)) {
    receitaAtivaId.value = receitas.value[0].id
  }
}

const carregarDoNavegador = () => {
  const rawNovo = localStorage.getItem(STORAGE_KEY)

  if (rawNovo) {
    try {
      const parsed = JSON.parse(rawNovo)
      receitas.value = Array.isArray(parsed.receitas) ? parsed.receitas : []
      receitaAtivaId.value = parsed.receitaAtivaId || ''
      ingredientesCatalogo.value = Array.isArray(parsed.ingredientesCatalogo)
        ? parsed.ingredientesCatalogo
        : []
      inicializarEstadoPadrao()
      return
    } catch {
      receitas.value = []
      ingredientesCatalogo.value = []
    }
  }

  const rawAntigo = localStorage.getItem('recipe-cost-calculator-v1')
  if (!rawAntigo) {
    inicializarEstadoPadrao()
    return
  }

  try {
    const parsed = JSON.parse(rawAntigo)
    const migrado = migrarFormatoAntigo(parsed)

    if (migrado) {
      receitas.value = migrado.receitas
      receitaAtivaId.value = migrado.receitaAtivaId
      ingredientesCatalogo.value = migrado.ingredientesCatalogo
    }
  } catch {
    receitas.value = []
    ingredientesCatalogo.value = []
  }

  inicializarEstadoPadrao()
}

carregarDoNavegador()

watch([receitas, receitaAtivaId, ingredientesCatalogo], salvarNoNavegador, {
  deep: true,
})
</script>

<template>
  <main class="pagina">
    <section class="hero">
      <p class="selo">Pão de mel Cravo e Canela e Cakes</p>
      <h1>Calculadora de Custo de Receita</h1>
      <p>
        Cadastro centralizado de ingredientes: voce reutiliza o mesmo ingrediente em
        varias receitas e qualquer atualizacao de preco/compra reflete em todas.
      </p>
    </section>

    <section class="abas" aria-label="Navegacao por abas">
      <button
        type="button"
        class="aba-btn"
        :class="{ ativo: abaAtiva === 'receita' }"
        @click="abaAtiva = 'receita'"
      >
        Receitas
      </button>
      <button
        type="button"
        class="aba-btn"
        :class="{ ativo: abaAtiva === 'catalogo' }"
        @click="abaAtiva = 'catalogo'"
      >
        Lista de Ingredientes
      </button>
    </section>

    <p v-if="mensagemErro" class="erro erro-global">{{ mensagemErro }}</p>

    <section class="card" v-if="abaAtiva === 'receita'">
      <div class="linha receitas-controle">
        <div class="campo">
          <label for="receitaAtiva">Receita ativa</label>
          <select id="receitaAtiva" v-model="receitaAtivaId" @change="cancelarEdicaoItem">
            <option v-for="receita in receitas" :key="receita.id" :value="receita.id">
              {{ receita.nome }}
            </option>
          </select>
        </div>

        <div class="acoes-receita">
          <button type="button" class="btn btn-primario" @click="adicionarReceita">
            Nova receita
          </button>
          <button type="button" class="btn btn-secundario" @click="abrirConfirmacaoReceita">
            Excluir receita
          </button>
        </div>
      </div>

      <div class="linha receita-nome">
        <label for="receitaNome">Nome da receita</label>
        <input
          id="receitaNome"
          v-model="nomeReceitaAtual"
          type="text"
          placeholder="Ex.: Bolo de cenoura"
        />
      </div>
    </section>

    <section class="card" v-if="abaAtiva === 'catalogo'">
      <h2>Cadastro central de ingredientes</h2>

      <form class="form-grid" @submit.prevent="adicionarIngredienteCatalogo">
        <div class="campo campo-largo">
          <label for="nomeIngredienteCatalogo">Ingrediente</label>
          <input
            id="nomeIngredienteCatalogo"
            v-model="novoIngredienteForm.nome"
            type="text"
            placeholder="Ex.: Farinha de trigo"
          />
        </div>

        <div class="campo">
          <label for="precoBaseCatalogo">Preco pago (R$)</label>
          <input
            id="precoBaseCatalogo"
            v-model="novoIngredienteForm.precoBase"
            type="number"
            min="0"
            step="0.01"
            placeholder="0,00"
          />
        </div>

        <div class="campo">
          <label for="quantidadeBaseCatalogo">Quantidade comprada</label>
          <input
            id="quantidadeBaseCatalogo"
            v-model="novoIngredienteForm.quantidadeBase"
            type="number"
            min="0"
            step="0.01"
            placeholder="1"
          />
        </div>

        <div class="campo">
          <label for="unidadeBaseCatalogo">Unidade comprada</label>
          <select id="unidadeBaseCatalogo" v-model="novoIngredienteForm.unidadeBase">
            <option
              v-for="unidade in unidadesPadrao"
              :key="`catalogo-${unidade}`"
              :value="unidade"
            >
              {{ unidade }}
            </option>
          </select>
        </div>

        <div class="acoes-form">
          <button type="submit" class="btn btn-primario">Adicionar no catalogo</button>
          <button type="button" class="btn btn-secundario" @click="limparNovoIngredienteForm">
            Limpar campos
          </button>
        </div>
      </form>

      <div class="tabela-wrapper" v-if="ingredientesCatalogo.length > 0">
        <table>
          <thead>
            <tr>
              <th>Ingrediente</th>
              <th>Preco base</th>
              <th>Qtd. comprada</th>
              <th></th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="ingrediente in ingredientesCatalogo" :key="ingrediente.id">
              <td>{{ ingrediente.nome }}</td>
              <td>{{ formatarMoeda(Number(ingrediente.precoBase)) }}</td>
              <td>{{ Number(ingrediente.quantidadeBase) }} {{ ingrediente.unidadeBase }}</td>
              <td>
                <button
                  type="button"
                  class="btn-remover"
                  @click="abrirConfirmacaoIngrediente(ingrediente.id)"
                >
                  Excluir
                </button>
              </td>
            </tr>
          </tbody>
        </table>
      </div>

      <p v-else class="vazio">Nenhum ingrediente no catalogo central.</p>
    </section>

    <section class="card" v-if="abaAtiva === 'receita'">
      <h2>Adicionar ingrediente na receita ativa</h2>

      <form class="form-grid" @submit.prevent="adicionarItemReceita">
        <div class="campo campo-largo">
          <label for="ingredienteReceita">Ingrediente do catalogo</label>
          <select
            id="ingredienteReceita"
            v-model="itemForm.ingredienteId"
            :disabled="ingredientesCatalogo.length === 0"
          >
            <option value="">Selecione</option>
            <option
              v-for="ingrediente in ingredientesCatalogo"
              :key="ingrediente.id"
              :value="ingrediente.id"
            >
              {{ ingrediente.nome }}
            </option>
          </select>
        </div>

        <div class="campo">
          <label for="quantidadeReceita">Quantidade na receita</label>
          <input
            id="quantidadeReceita"
            v-model="itemForm.quantidadeReceita"
            type="number"
            min="0"
            step="0.01"
            placeholder="0"
          />
        </div>

        <div class="campo">
          <label for="unidadeReceita">Unidade na receita</label>
          <select id="unidadeReceita" v-model="itemForm.unidadeReceita">
            <option v-for="unidade in unidadesPadrao" :key="`item-${unidade}`" :value="unidade">
              {{ unidade }}
            </option>
          </select>
        </div>

        <div class="acoes-form">
          <button type="submit" class="btn btn-primario">Adicionar ingrediente</button>
          <button type="button" class="btn btn-secundario" @click="limparItemForm">
            Limpar campos
          </button>
        </div>
      </form>
    </section>

    <section class="card" v-if="abaAtiva === 'receita'">
      <div class="cabecalho-tabela">
        <h2>Itens da receita</h2>
        <button
          class="btn btn-secundario"
          type="button"
          @click="limparReceita"
          :disabled="itensAtivos.length === 0"
        >
          Limpar receita
        </button>
      </div>

      <div v-if="itensAtivos.length === 0" class="vazio">
        Nenhum ingrediente adicionado ainda.
      </div>

      <div v-else class="tabela-wrapper">
        <table>
          <thead>
            <tr>
              <th>Ingrediente</th>
              <th>Preco base global</th>
              <th>Qtd. comprada global</th>
              <th>Qtd. na receita</th>
              <th>Custo na receita</th>
              <th></th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="item in itensAtivos" :key="item.id">
              <template v-if="obterIngredienteDoItem(item)">
                <td v-if="itemEmEdicaoId === item.id">
                  <input
                    v-model="itemEdicaoForm.nomeIngrediente"
                    type="text"
                    class="input-tabela"
                  />
                </td>
                <td v-else>{{ obterIngredienteDoItem(item)?.nome }}</td>

                <td v-if="itemEmEdicaoId === item.id">
                  <input
                    v-model="itemEdicaoForm.precoBase"
                    type="number"
                    min="0"
                    step="0.01"
                    class="input-tabela"
                  />
                </td>
                <td v-else>
                  {{ formatarMoeda(Number(obterIngredienteDoItem(item)?.precoBase || 0)) }}
                </td>

                <td v-if="itemEmEdicaoId === item.id" class="celula-dupla">
                  <input
                    v-model="itemEdicaoForm.quantidadeBase"
                    type="number"
                    min="0"
                    step="0.01"
                    class="input-tabela"
                  />
                  <select v-model="itemEdicaoForm.unidadeBase" class="input-tabela">
                    <option
                      v-for="unidade in unidadesPadrao"
                      :key="`ed-base-${unidade}`"
                      :value="unidade"
                    >
                      {{ unidade }}
                    </option>
                  </select>
                </td>
                <td v-else>
                  {{ Number(obterIngredienteDoItem(item)?.quantidadeBase || 0) }}
                  {{ obterIngredienteDoItem(item)?.unidadeBase }}
                </td>

                <td v-if="itemEmEdicaoId === item.id" class="celula-dupla">
                  <input
                    v-model="itemEdicaoForm.quantidadeReceita"
                    type="number"
                    min="0"
                    step="0.01"
                    class="input-tabela"
                  />
                  <select v-model="itemEdicaoForm.unidadeReceita" class="input-tabela">
                    <option
                      v-for="unidade in unidadesPadrao"
                      :key="`ed-rec-${unidade}`"
                      :value="unidade"
                    >
                      {{ unidade }}
                    </option>
                  </select>
                </td>
                <td v-else>{{ Number(item.quantidadeReceita) }} {{ item.unidadeReceita }}</td>

                <td class="custo-item">{{ formatarMoeda(custoDoItem(item)) }}</td>

                <td>
                  <div class="acoes-linha" v-if="itemEmEdicaoId === item.id">
                    <button class="btn-salvar" type="button" @click="salvarEdicaoItem(item.id)">
                      Salvar
                    </button>
                    <button class="btn-cancelar" type="button" @click="cancelarEdicaoItem">
                      Cancelar
                    </button>
                  </div>
                  <div class="acoes-linha" v-else>
                    <button class="btn-editar" type="button" @click="iniciarEdicaoItem(item)">
                      Editar
                    </button>
                    <button class="btn-remover" type="button" @click="removerItemReceita(item.id)">
                      Remover
                    </button>
                  </div>
                </td>
              </template>
            </tr>
          </tbody>
        </table>
      </div>

      <div class="resumo">
        <span>Total da receita:</span>
        <strong>{{ formatarMoeda(totalReceita) }}</strong>
      </div>
    </section>

    <div
      v-if="confirmacaoExclusao.aberta"
      class="modal-overlay"
      role="dialog"
      aria-modal="true"
      :aria-label="confirmacaoExclusao.titulo"
    >
      <div class="modal-confirmacao">
        <h3>{{ confirmacaoExclusao.titulo }}</h3>
        <p>{{ confirmacaoExclusao.mensagem }}</p>

        <div class="modal-acoes">
          <button type="button" class="btn btn-secundario" @click="cancelarConfirmacaoExclusao">
            Cancelar
          </button>
          <button type="button" class="btn btn-perigo" @click="confirmarExclusao">
            Confirmar exclusao
          </button>
        </div>
      </div>
    </div>
  </main>
</template>
