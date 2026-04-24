<script setup>
import { computed, reactive, ref, watch } from 'vue'

const STORAGE_KEY = 'recipe-cost-calculator-v1'
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

const ingredienteForm = reactive({
  nome: '',
  precoBase: '',
  quantidadeBase: '',
  unidadeBase: 'kg',
  quantidadeReceita: '',
  unidadeReceita: 'kg',
})

const mensagemErro = ref('')
const ingredienteEmEdicaoId = ref(null)
const ingredienteEdicaoForm = reactive({
  nome: '',
  precoBase: '',
  quantidadeBase: '',
  unidadeBase: 'kg',
  quantidadeReceita: '',
  unidadeReceita: 'kg',
})

const criarReceita = (nome = 'Nova receita') => ({
  id: crypto.randomUUID(),
  nome,
  ingredientes: [],
})

const receitaAtiva = computed(() =>
  receitas.value.find((receita) => receita.id === receitaAtivaId.value),
)

const ingredientesAtivos = computed(() => receitaAtiva.value?.ingredientes || [])

const nomeReceitaAtual = computed({
  get: () => receitaAtiva.value?.nome || '',
  set: (novoNome) => {
    if (!receitaAtiva.value) return
    receitaAtiva.value.nome = novoNome
  },
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

const calcularCusto = (item) => {
  const precoBase = Number(item.precoBase)
  const quantidadeBase = Number(item.quantidadeBase)
  const quantidadeReceita = Number(item.quantidadeReceita)

  if (precoBase <= 0 || quantidadeBase <= 0 || quantidadeReceita < 0) {
    return 0
  }

  const quantidadeBaseConvertida = converterQuantidade(
    quantidadeBase,
    item.unidadeBase,
    item.unidadeReceita,
  )

  if (!quantidadeBaseConvertida || quantidadeBaseConvertida <= 0) {
    return 0
  }

  return (precoBase / quantidadeBaseConvertida) * quantidadeReceita
}

const totalReceita = computed(() =>
  ingredientesAtivos.value.reduce((acc, item) => acc + calcularCusto(item), 0),
)

const limparFormulario = () => {
  ingredienteForm.nome = ''
  ingredienteForm.precoBase = ''
  ingredienteForm.quantidadeBase = ''
  ingredienteForm.unidadeBase = 'kg'
  ingredienteForm.quantidadeReceita = ''
  ingredienteForm.unidadeReceita = 'kg'
  mensagemErro.value = ''
}

const adicionarIngrediente = () => {
  if (!receitaAtiva.value) return

  const nome = ingredienteForm.nome.trim()
  const precoBase = Number(ingredienteForm.precoBase)
  const quantidadeBase = Number(ingredienteForm.quantidadeBase)
  const quantidadeReceita = Number(ingredienteForm.quantidadeReceita)

  if (!nome) {
    mensagemErro.value = 'Informe o nome do ingrediente.'
    return
  }

  if (precoBase <= 0 || quantidadeBase <= 0 || quantidadeReceita < 0) {
    mensagemErro.value =
      'Revise os valores: preco base e quantidade base devem ser maiores que zero.'
    return
  }

  if (
    !unidadesCompativeis(
      ingredienteForm.unidadeBase,
      ingredienteForm.unidadeReceita,
    )
  ) {
    mensagemErro.value =
      'As unidades precisam ser compativeis: kg/g, l/ml ou un com un.'
    return
  }

  receitaAtiva.value.ingredientes.push({
    id: crypto.randomUUID(),
    nome,
    precoBase,
    quantidadeBase,
    unidadeBase: ingredienteForm.unidadeBase,
    quantidadeReceita,
    unidadeReceita: ingredienteForm.unidadeReceita,
  })

  limparFormulario()
}

const removerIngrediente = (id) => {
  if (!receitaAtiva.value) return

  receitaAtiva.value.ingredientes = receitaAtiva.value.ingredientes.filter(
    (item) => item.id !== id,
  )

  if (ingredienteEmEdicaoId.value === id) {
    ingredienteEmEdicaoId.value = null
  }
}

const iniciarEdicaoIngrediente = (item) => {
  ingredienteEmEdicaoId.value = item.id
  ingredienteEdicaoForm.nome = item.nome
  ingredienteEdicaoForm.precoBase = String(item.precoBase)
  ingredienteEdicaoForm.quantidadeBase = String(item.quantidadeBase)
  ingredienteEdicaoForm.unidadeBase = item.unidadeBase
  ingredienteEdicaoForm.quantidadeReceita = String(item.quantidadeReceita)
  ingredienteEdicaoForm.unidadeReceita = item.unidadeReceita
  mensagemErro.value = ''
}

const cancelarEdicaoIngrediente = () => {
  ingredienteEmEdicaoId.value = null
  mensagemErro.value = ''
}

const salvarEdicaoIngrediente = (id) => {
  if (!receitaAtiva.value) return

  const nome = ingredienteEdicaoForm.nome.trim()
  const precoBase = Number(ingredienteEdicaoForm.precoBase)
  const quantidadeBase = Number(ingredienteEdicaoForm.quantidadeBase)
  const quantidadeReceita = Number(ingredienteEdicaoForm.quantidadeReceita)

  if (!nome) {
    mensagemErro.value = 'Informe o nome do ingrediente.'
    return
  }

  if (precoBase <= 0 || quantidadeBase <= 0 || quantidadeReceita < 0) {
    mensagemErro.value =
      'Revise os valores: preco base e quantidade base devem ser maiores que zero.'
    return
  }

  if (
    !unidadesCompativeis(
      ingredienteEdicaoForm.unidadeBase,
      ingredienteEdicaoForm.unidadeReceita,
    )
  ) {
    mensagemErro.value =
      'As unidades precisam ser compativeis: kg/g, l/ml ou un com un.'
    return
  }

  receitaAtiva.value.ingredientes = receitaAtiva.value.ingredientes.map((item) => {
    if (item.id !== id) return item

    return {
      ...item,
      nome,
      precoBase,
      quantidadeBase,
      unidadeBase: ingredienteEdicaoForm.unidadeBase,
      quantidadeReceita,
      unidadeReceita: ingredienteEdicaoForm.unidadeReceita,
    }
  })

  ingredienteEmEdicaoId.value = null
  mensagemErro.value = ''
}

const limparReceita = () => {
  if (!receitaAtiva.value) return

  receitaAtiva.value.ingredientes = []
  cancelarEdicaoIngrediente()
}

const adicionarReceita = () => {
  const novoNome = `Receita ${receitas.value.length + 1}`
  const novaReceita = criarReceita(novoNome)

  receitas.value.push(novaReceita)
  receitaAtivaId.value = novaReceita.id
  limparFormulario()
  cancelarEdicaoIngrediente()
}

const excluirReceitaAtiva = () => {
  if (!receitaAtiva.value) return

  receitas.value = receitas.value.filter(
    (receita) => receita.id !== receitaAtivaId.value,
  )

  if (receitas.value.length === 0) {
    const receitaInicial = criarReceita('Receita 1')
    receitas.value.push(receitaInicial)
  }

  receitaAtivaId.value = receitas.value[0].id
  limparFormulario()
  cancelarEdicaoIngrediente()
}

const salvarNoNavegador = () => {
  const payload = {
    receitas: receitas.value,
    receitaAtivaId: receitaAtivaId.value,
  }

  localStorage.setItem(STORAGE_KEY, JSON.stringify(payload))
}

const carregarDoNavegador = () => {
  const raw = localStorage.getItem(STORAGE_KEY)
  if (!raw) return

  try {
    const parsed = JSON.parse(raw)

    if (Array.isArray(parsed.receitas) && parsed.receitas.length > 0) {
      receitas.value = parsed.receitas
      receitaAtivaId.value = parsed.receitaAtivaId || parsed.receitas[0].id
      return
    }

    if (parsed.receitaNome || Array.isArray(parsed.ingredientes)) {
      const receitaMigrada = criarReceita(parsed.receitaNome || 'Receita 1')
      receitaMigrada.ingredientes = Array.isArray(parsed.ingredientes)
        ? parsed.ingredientes
        : []

      receitas.value = [receitaMigrada]
      receitaAtivaId.value = receitaMigrada.id
      return
    }
  } catch {
    receitas.value = []
  }

  if (receitas.value.length === 0) {
    const receitaInicial = criarReceita('Receita 1')
    receitas.value = [receitaInicial]
    receitaAtivaId.value = receitaInicial.id
  }
}

carregarDoNavegador()

if (receitas.value.length === 0) {
  const receitaInicial = criarReceita('Receita 1')
  receitas.value = [receitaInicial]
  receitaAtivaId.value = receitaInicial.id
}

if (!receitas.value.some((receita) => receita.id === receitaAtivaId.value)) {
  receitaAtivaId.value = receitas.value[0].id
}

watch([receitas, receitaAtivaId], salvarNoNavegador, { deep: true })
</script>

<template>
  <main class="pagina">
    <section class="hero">
      <p class="selo">Cravo e canela</p>
      <h1>Calculadora de Custo de Receita</h1>
      <p>
        Monte a receita, informe preco e quantidade de cada ingrediente, e veja o
        custo total do preparo.
      </p>
    </section>

    <section class="card">
      <div class="linha receitas-controle">
        <div class="campo">
          <label for="receitaAtiva">Receita ativa</label>
          <select id="receitaAtiva" v-model="receitaAtivaId" @change="cancelarEdicaoIngrediente">
            <option v-for="receita in receitas" :key="receita.id" :value="receita.id">
              {{ receita.nome }}
            </option>
          </select>
        </div>

        <div class="acoes-receita">
          <button type="button" class="btn btn-primario" @click="adicionarReceita">
            Nova receita
          </button>
          <button type="button" class="btn btn-secundario" @click="excluirReceitaAtiva">
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

      <form class="form-grid" @submit.prevent="adicionarIngrediente">
        <div class="campo campo-largo">
          <label for="nome">Ingrediente</label>
          <input
            id="nome"
            v-model="ingredienteForm.nome"
            type="text"
            placeholder="Ex.: Farinha de trigo"
          />
        </div>

        <div class="campo">
          <label for="precoBase">Preco pago (R$)</label>
          <input
            id="precoBase"
            v-model="ingredienteForm.precoBase"
            type="number"
            min="0"
            step="0.01"
            placeholder="0,00"
          />
        </div>

        <div class="campo">
          <label for="quantidadeBase">Quantidade comprada</label>
          <input
            id="quantidadeBase"
            v-model="ingredienteForm.quantidadeBase"
            type="number"
            min="0"
            step="0.01"
            placeholder="1"
          />
        </div>

        <div class="campo">
          <label for="unidadeBase">Unidade comprada</label>
          <select id="unidadeBase" v-model="ingredienteForm.unidadeBase">
            <option v-for="unidade in unidadesPadrao" :key="unidade" :value="unidade">
              {{ unidade }}
            </option>
          </select>
        </div>

        <div class="campo">
          <label for="quantidadeReceita">Quantidade na receita</label>
          <input
            id="quantidadeReceita"
            v-model="ingredienteForm.quantidadeReceita"
            type="number"
            min="0"
            step="0.01"
            placeholder="0"
          />
        </div>

        <div class="campo">
          <label for="unidadeReceita">Unidade na receita</label>
          <select id="unidadeReceita" v-model="ingredienteForm.unidadeReceita">
            <option v-for="unidade in unidadesPadrao" :key="`${unidade}-receita`" :value="unidade">
              {{ unidade }}
            </option>
          </select>
        </div>

        <div class="acoes-form">
          <button type="submit" class="btn btn-primario">Adicionar ingrediente</button>
          <button type="button" class="btn btn-secundario" @click="limparFormulario">
            Limpar campos
          </button>
        </div>
      </form>

      <p v-if="mensagemErro" class="erro">{{ mensagemErro }}</p>
    </section>

    <section class="card">
      <div class="cabecalho-tabela">
        <h2>Ingredientes da receita</h2>
        <button
          class="btn btn-secundario"
          type="button"
          @click="limparReceita"
          :disabled="ingredientesAtivos.length === 0"
        >
          Limpar receita
        </button>
      </div>

      <div v-if="ingredientesAtivos.length === 0" class="vazio">
        Nenhum ingrediente adicionado ainda.
      </div>

      <div v-else class="tabela-wrapper">
        <table>
          <thead>
            <tr>
              <th>Ingrediente</th>
              <th>Preco base</th>
              <th>Qtd. base</th>
              <th>Qtd. receita</th>
              <th>Custo na receita</th>
              <th></th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="item in ingredientesAtivos" :key="item.id">
              <td v-if="ingredienteEmEdicaoId === item.id">
                <input v-model="ingredienteEdicaoForm.nome" type="text" class="input-tabela" />
              </td>
              <td v-else>{{ item.nome }}</td>

              <td v-if="ingredienteEmEdicaoId === item.id">
                <input
                  v-model="ingredienteEdicaoForm.precoBase"
                  type="number"
                  min="0"
                  step="0.01"
                  class="input-tabela"
                />
              </td>
              <td v-else>{{ formatarMoeda(Number(item.precoBase)) }}</td>

              <td v-if="ingredienteEmEdicaoId === item.id" class="celula-dupla">
                <input
                  v-model="ingredienteEdicaoForm.quantidadeBase"
                  type="number"
                  min="0"
                  step="0.01"
                  class="input-tabela"
                />
                <select v-model="ingredienteEdicaoForm.unidadeBase" class="input-tabela">
                  <option v-for="unidade in unidadesPadrao" :key="`ed-base-${unidade}`" :value="unidade">
                    {{ unidade }}
                  </option>
                </select>
              </td>
              <td v-else>{{ Number(item.quantidadeBase) }} {{ item.unidadeBase }}</td>

              <td v-if="ingredienteEmEdicaoId === item.id" class="celula-dupla">
                <input
                  v-model="ingredienteEdicaoForm.quantidadeReceita"
                  type="number"
                  min="0"
                  step="0.01"
                  class="input-tabela"
                />
                <select v-model="ingredienteEdicaoForm.unidadeReceita" class="input-tabela">
                  <option v-for="unidade in unidadesPadrao" :key="`ed-rec-${unidade}`" :value="unidade">
                    {{ unidade }}
                  </option>
                </select>
              </td>
              <td v-else>{{ Number(item.quantidadeReceita) }} {{ item.unidadeReceita }}</td>

              <td class="custo-item">{{ formatarMoeda(calcularCusto(item)) }}</td>
              <td>
                <div class="acoes-linha" v-if="ingredienteEmEdicaoId === item.id">
                  <button class="btn-salvar" type="button" @click="salvarEdicaoIngrediente(item.id)">
                    Salvar
                  </button>
                  <button class="btn-cancelar" type="button" @click="cancelarEdicaoIngrediente">
                    Cancelar
                  </button>
                </div>
                <div class="acoes-linha" v-else>
                  <button class="btn-editar" type="button" @click="iniciarEdicaoIngrediente(item)">
                    Editar
                  </button>
                  <button class="btn-remover" type="button" @click="removerIngrediente(item.id)">
                    Remover
                  </button>
                </div>
              </td>
            </tr>
          </tbody>
        </table>
      </div>

      <div class="resumo">
        <span>Total da receita:</span>
        <strong>{{ formatarMoeda(totalReceita) }}</strong>
      </div>
    </section>
  </main>
</template>
