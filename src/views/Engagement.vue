<script setup lang="ts">
import { ref, computed } from 'vue'
import { useFeedbackStore } from '../stores/feedback'
import { startOfMonth, endOfMonth, subMonths, parseISO, format } from 'date-fns'
import { ptBR } from 'date-fns/locale'
import AIInsightsPanel from '../components/AIInsightsPanel.vue'
import { Line } from 'vue-chartjs'
import { Chart as ChartJS, CategoryScale, LinearScale, PointElement, LineElement, Title, Tooltip, Legend } from 'chart.js'

ChartJS.register(CategoryScale, LinearScale, PointElement, LineElement, Title, Tooltip, Legend)

const feedbackStore = useFeedbackStore()

// Dados dos últimos 4 meses
const lastFourMonthsData = computed(() => {
  const fourMonthsAgo = subMonths(new Date(), 4)
  return feedbackStore.feedbackData.filter(f => {
    const date = parseISO(f.created_at)
    return date >= fourMonthsAgo
  })
})

// Função para calcular estatísticas por papel
const getRoleStats = (data: any[], role: string) => {
  const roleData = data.filter(f => f.role === role)
  return {
    total: roleData.length,
    promoters: roleData.filter(f => f.score >= 9).length,
    neutrals: roleData.filter(f => f.score >= 7 && f.score <= 8).length,
    detractors: roleData.filter(f => f.score <= 6).length,
    nps: roleData.length ? Math.round(((roleData.filter(f => f.score >= 9).length - 
          roleData.filter(f => f.score <= 6).length) / roleData.length) * 100) : 0
  }
}

// Estatísticas dos últimos 4 meses
const currentStats = computed(() => ({
  gestor: getRoleStats(lastFourMonthsData.value, 'manager'),
  supervisor: getRoleStats(lastFourMonthsData.value, 'supervisor'),
  agente: getRoleStats(lastFourMonthsData.value, 'agent')
}))

// Estatísticas do período anterior (4 meses anteriores para comparação)
const lastStats = computed(() => {
  const eightMonthsAgo = subMonths(new Date(), 8)
  const fourMonthsAgo = subMonths(new Date(), 4)
  
  const previousPeriodData = feedbackStore.feedbackData.filter(f => {
    const date = parseISO(f.created_at)
    return date >= eightMonthsAgo && date < fourMonthsAgo
  })

  return {
    gestor: getRoleStats(previousPeriodData, 'manager'),
    supervisor: getRoleStats(previousPeriodData, 'supervisor'),
    agente: getRoleStats(previousPeriodData, 'agent')
  }
})

// Função para capitalizar a primeira letra
const capitalize = (s: string) => s.charAt(0).toUpperCase() + s.slice(1)

// Função para calcular estatísticas por palavra-chave
const getKeywordStats = (keywords: string[]) => {
  const months = Array.from({ length: 4 }, (_, i) => {
    const date = subMonths(new Date(), i)
    const start = startOfMonth(date)
    const end = endOfMonth(date)
    const monthData = feedbackStore.feedbackData.filter(f => {
      const fDate = parseISO(f.created_at)
      return fDate >= start && fDate <= end && 
             keywords.some(keyword => f.reason?.toLowerCase().includes(keyword.toLowerCase()))
    })
    
    return {
      month: format(date, 'MMMM', { locale: ptBR }),
      count: monthData.length,
      comments: monthData.map(f => f.reason)
    }
  }).reverse()

  return months
}

const instabilityStats = computed(() => getKeywordStats(['instabilidade']))
const supportStats = computed(() => getKeywordStats(['suporte']))
const noCallStats = computed(() => getKeywordStats(['não disca', 'não liga', 'não chama', 'demora', 'ligações desconhecidas']))

const chartData = computed(() => {
  const labels = instabilityStats.value.map(stat => stat.month)
  return {
    labels,
    datasets: [
      {
        label: 'Reclamações de instabilidade',
        data: instabilityStats.value.map(stat => stat.count),
        borderColor: '#FF6384',
        backgroundColor: '#FF6384',
        fill: false,
        tension: 0.4,
        pointStyle: 'circle',
        pointRadius: 5,
        pointHoverRadius: 5
      },
      {
        label: 'Reclamações de suporte',
        data: supportStats.value.map(stat => stat.count),
        borderColor: '#36A2EB',
        backgroundColor: '#36A2EB',
        fill: false,
        tension: 0.4,
        pointStyle: 'circle',
        pointRadius: 5,
        pointHoverRadius: 5
      },
      {
        label: 'Problemas com mailing',
        data: noCallStats.value.map(stat => stat.count),
        borderColor: '#FFCE56',
        backgroundColor: '#FFCE56',
        fill: false,
        tension: 0.4,
        pointStyle: 'circle',
        pointRadius: 5,
        pointHoverRadius: 5
      }
    ]
  }
})

const chartOptions = {
  responsive: true,
  maintainAspectRatio: false,
  plugins: {
    legend: {
      position: 'bottom' as const,
      labels: {
        usePointStyle: true,
        padding: 20,
      },
    },
    tooltip: {
      mode: 'index' as const,
      intersect: false
    }
  },
  scales: {
    x: {
      display: true,
      title: {
        display: true,
        text: 'Mês'
      }
    },
    y: {
      display: true,
      title: {
        display: true,
        text: 'NPS'
      }
    }
  }
}

const activeFilter = ref('Suporte')

// Definindo a propriedade insights
const insights = ref({ recommendations: [] })
</script>

<template>
  <div class="space-y-6">
    <!-- Seção de Insights -->
    <div>
      <!-- Remover o título -->
      <!-- <h1 class="text-2xl font-semibold text-gray-900">Insights do Mês</h1> -->

      <!-- Cards de Insights -->
      <div class="grid grid-cols-1 md:grid-cols-3 gap-6">
        <!-- Card do Gestor -->
        <div class="bg-white rounded-[10px] border border-[#E1E9F4] shadow-[0_12px_24px_0_rgba(18,38,63,0.03)] px-6 py-6 card-padding">
          <div class="flex items-center justify-between mb-4">
            <h3 class="text-lg font-medium text-[#373753]">Gestor</h3>
            <span class="text-sm font-medium bg-blue-100 text-blue-800 px-2 py-1 rounded-lg">
              {{ currentStats.gestor.nps }}% NPS
            </span>
          </div>

          <!-- Adicionando o divisor -->
          <hr class="border-t border-gray-200 mb-6 -mx-6">

          <div class="space-y-3">
            <div class="flex justify-between text-sm">
              <span class="text-gray-500 flex items-center gap-2">
                <span class="text-lg">😀</span>
                Promotores
              </span>
              <span class="font-medium">{{ currentStats.gestor.promoters }}</span>
            </div>
            <div class="flex justify-between text-sm">
              <span class="text-gray-500 flex items-center gap-2">
                <span class="text-lg">😐</span>
                Neutros
              </span>
              <span class="font-medium">{{ currentStats.gestor.neutrals }}</span>
            </div>
            <div class="flex justify-between text-sm">
              <span class="text-gray-500 flex items-center gap-2">
                <span class="text-lg">😡</span>
                Detratores
              </span>
              <span class="font-medium">{{ currentStats.gestor.detractors }}</span>
            </div>
            <div class="pt-6 border-t">
              <p class="text-sm text-gray-500 mb-2">Comparação trimestre passado</p>
              <div class="flex items-center">
                <span class="font-medium" :class="{
                  'text-green-600': currentStats.gestor.nps > lastStats.gestor.nps,
                  'text-red-600': currentStats.gestor.nps < lastStats.gestor.nps,
                  'text-gray-600': currentStats.gestor.nps === lastStats.gestor.nps
                }">
                  {{ currentStats.gestor.nps - lastStats.gestor.nps }}%
                </span>
                <span class="ml-2 text-xs font-medium px-2 py-0.5 rounded"
                      :class="{
                        'bg-green-100 text-green-800': currentStats.gestor.nps > lastStats.gestor.nps,
                        'bg-red-100 text-red-800': currentStats.gestor.nps < lastStats.gestor.nps,
                        'bg-gray-100 text-gray-800': currentStats.gestor.nps === lastStats.gestor.nps
                      }">
                  {{ currentStats.gestor.nps > lastStats.gestor.nps ? 'Subiu' : currentStats.gestor.nps < lastStats.gestor.nps ? 'Baixou' : 'Estável' }}
                </span>
              </div>
            </div>
          </div>
        </div>

        <!-- Card do Supervisor -->
        <div class="bg-white rounded-[10px] border border-[#E1E9F4] shadow-[0_12px_24px_0_rgba(18,38,63,0.03)] px-6 py-6 card-padding">
          <div class="flex items-center justify-between mb-4">
            <h3 class="text-lg font-medium text-[#373753]">Supervisor</h3>
            <span class="text-sm font-medium bg-blue-100 text-blue-800 px-2 py-1 rounded-lg">
              {{ currentStats.supervisor.nps }}% NPS
            </span>
          </div>

          <!-- Adicionando o divisor -->
          <hr class="border-t border-gray-200 mb-6 -mx-6">

          <div class="space-y-3">
            <div class="flex justify-between text-sm">
              <span class="text-gray-500 flex items-center gap-2">
                <span class="text-lg">😀</span>
                Promotores
              </span>
              <span class="font-medium">{{ currentStats.supervisor.promoters }}</span>
            </div>
            <div class="flex justify-between text-sm">
              <span class="text-gray-500 flex items-center gap-2">
                <span class="text-lg">😐</span>
                Neutros
              </span>
              <span class="font-medium">{{ currentStats.supervisor.neutrals }}</span>
            </div>
            <div class="flex justify-between text-sm">
              <span class="text-gray-500 flex items-center gap-2">
                <span class="text-lg">😡</span>
                Detratores
              </span>
              <span class="font-medium">{{ currentStats.supervisor.detractors }}</span>
            </div>
            <div class="pt-6 border-t">
              <p class="text-sm text-gray-500 mb-2">Comparação trimestre passado</p>
              <div class="flex items-center">
                <span class="font-medium" :class="{
                  'text-green-600': currentStats.supervisor.nps > lastStats.supervisor.nps,
                  'text-red-600': currentStats.supervisor.nps < lastStats.supervisor.nps,
                  'text-gray-600': currentStats.supervisor.nps === lastStats.supervisor.nps
                }">
                  {{ currentStats.supervisor.nps - lastStats.supervisor.nps }}%
                </span>
                <span class="ml-2 text-xs font-medium px-2 py-0.5 rounded"
                      :class="{
                        'bg-green-100 text-green-800': currentStats.supervisor.nps > lastStats.supervisor.nps,
                        'bg-red-100 text-red-800': currentStats.supervisor.nps < lastStats.supervisor.nps,
                        'bg-gray-100 text-gray-800': currentStats.supervisor.nps === lastStats.supervisor.nps
                      }">
                  {{ currentStats.supervisor.nps > lastStats.supervisor.nps ? 'Subiu' : currentStats.supervisor.nps < lastStats.supervisor.nps ? 'Baixou' : 'Estável' }}
                </span>
              </div>
            </div>
          </div>
        </div>

        <!-- Card do Agente -->
        <div class="bg-white rounded-[10px] border border-[#E1E9F4] shadow-[0_12px_24px_0_rgba(18,38,63,0.03)] px-6 py-6 card-padding">
          <div class="flex items-center justify-between mb-4">
            <h3 class="text-lg font-medium text-[#373753]">Agente</h3>
            <span class="text-sm font-medium bg-blue-100 text-blue-800 px-2 py-1 rounded-lg">
              {{ currentStats.agente.nps }}% NPS
            </span>
          </div>

          <!-- Adicionando o divisor -->
          <hr class="border-t border-gray-200 mb-6 -mx-6">

          <div class="space-y-3">
            <div class="flex justify-between text-sm">
              <span class="text-gray-500 flex items-center gap-2">
                <span class="text-lg">😀</span>
                Promotores
              </span>
              <span class="font-medium">{{ currentStats.agente.promoters }}</span>
            </div>
            <div class="flex justify-between text-sm">
              <span class="text-gray-500 flex items-center gap-2">
                <span class="text-lg">😐</span>
                Neutros
              </span>
              <span class="font-medium">{{ currentStats.agente.neutrals }}</span>
            </div>
            <div class="flex justify-between text-sm">
              <span class="text-gray-500 flex items-center gap-2">
                <span class="text-lg">😡</span>
                Detratores
              </span>
              <span class="font-medium">{{ currentStats.agente.detractors }}</span>
            </div>
            <div class="pt-6 border-t">
              <p class="text-sm text-gray-500 mb-2">Comparação trimestre passado</p>
              <div class="flex items-center">
                <span class="font-medium" :class="{
                  'text-green-600': currentStats.agente.nps > lastStats.agente.nps,
                  'text-red-600': currentStats.agente.nps < lastStats.agente.nps,
                  'text-gray-600': currentStats.agente.nps === lastStats.agente.nps
                }">
                  {{ currentStats.agente.nps - lastStats.agente.nps }}%
                </span>
                <span class="ml-2 text-xs font-medium px-2 py-0.5 rounded"
                      :class="{
                        'bg-green-100 text-green-800': currentStats.agente.nps > lastStats.agente.nps,
                        'bg-red-100 text-red-800': currentStats.agente.nps < lastStats.agente.nps,
                        'bg-gray-100 text-gray-800': currentStats.agente.nps === lastStats.agente.nps
                      }">
                  {{ currentStats.agente.nps > lastStats.agente.nps ? 'Subiu' : currentStats.agente.nps < lastStats.agente.nps ? 'Baixou' : 'Estável' }}
                </span>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- Recomendações gerais -->
      <div v-if="insights?.recommendations?.length" class="col-span-full mt-6 bg-white rounded-lg shadow p-6">
        <h3 class="text-lg font-semibold mb-4">Recomendações Gerais</h3>
        <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
          <div v-for="(rec, i) in insights.recommendations" 
               :key="i" 
               class="bg-gray-50 p-4 rounded-lg">
            <p class="text-gray-700">{{ rec }}</p>
          </div>
        </div>
      </div>
    </div>

    <!-- Espaçamento entre seções -->
    <!-- Removido o espaçamento adicional -->

    <!-- Gráfico de Comparação de Tópicos -->
    <div class="mt-6 bg-white rounded-[10px] border border-[#E1E9F4] shadow-[0_12px_24px_0_rgba(18,38,63,0.03)]">
      <div class="px-6 py-4">
        <h3 class="text-lg font-medium text-[#373753]">Tópicos dos últimos quatro meses</h3>
      </div>
      <hr class="border-t border-gray-200">
      <div class="p-6 h-[400px]">
        <Line :data="chartData" :options="chartOptions" />
      </div>
    </div>

    <!-- Seção de Análise de Reclamações -->
    <div class="analysis-section">
      <div class="analysis-card">
        <div class="flex justify-between items-center mb-4">
          <h2 class="text-lg font-medium text-[#373753]">Análise de Reclamações</h2>
          <div class="flex space-x-4">
            <button class="filter-button" :class="{ 'active-filter': activeFilter === 'Suporte' }" @click="activeFilter = 'Suporte'">Suporte</button>
            <button class="filter-button" :class="{ 'active-filter': activeFilter === 'Instabilidade' }" @click="activeFilter = 'Instabilidade'">Instabilidade</button>
            <button class="filter-button" :class="{ 'active-filter': activeFilter === 'Problemas com mailing' }" @click="activeFilter = 'Problemas com mailing'">Problemas com mailing</button>
          </div>
        </div>
        <hr class="border-t border-gray-200 mb-6 -mx-6">
        <div v-if="activeFilter === 'Suporte'" class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
          <div v-for="(month, index) in supportStats" :key="index" class="bg-white rounded-lg border border-[#E1E9F4] shadow-[0_12px_24px_0_rgba(18,38,63,0.03)] h-80">
            <div class="p-4">
              <div class="flex justify-between items-center">
                <h4 class="font-medium text-[#373753]">{{ capitalize(month.month) }}</h4>
                <div class="flex items-center">
                  <p class="text-base font-bold text-[#373753]">{{ month.count }}</p>
                  <span class="ml-2 text-sm text-[#677C92]">{{ month.count === 1 ? 'Relato' : 'Relatos' }}</span>
                </div>
              </div>
            </div>
            <hr class="border-t border-gray-200">
            <div class="p-4 mt-2 space-y-2 overflow-y-auto max-h-56 custom-scroll">
              <div v-for="(comment, i) in month.comments" :key="i" class="text-sm text-gray-600 bg-red-50 p-2 rounded">
                {{ comment }}
              </div>
            </div>
          </div>
        </div>
        <div v-if="activeFilter === 'Instabilidade'" class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
          <div v-for="(month, index) in instabilityStats" :key="index" class="bg-white rounded-lg border border-[#E1E9F4] shadow-[0_12px_24px_0_rgba(18,38,63,0.03)] h-80">
            <div class="p-4">
              <div class="flex justify-between items-center">
                <h4 class="font-medium text-[#373753]">{{ capitalize(month.month) }}</h4>
                <div class="flex items-center">
                  <p class="text-base font-bold text-[#373753]">{{ month.count }}</p>
                  <span class="ml-2 text-sm text-[#677C92]">{{ month.count === 1 ? 'Relato' : 'Relatos' }}</span>
                </div>
              </div>
            </div>
            <hr class="border-t border-gray-200">
            <div class="p-4 mt-2 space-y-2 overflow-y-auto max-h-56 custom-scroll">
              <div v-for="(comment, i) in month.comments" :key="i" class="text-sm text-gray-600 bg-red-50 p-2 rounded">
                {{ comment }}
              </div>
            </div>
          </div>
        </div>
        <div v-if="activeFilter === 'Problemas com mailing'" class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
          <div v-for="(month, index) in noCallStats" :key="index" class="bg-white rounded-lg border border-[#E1E9F4] shadow-[0_12px_24px_0_rgba(18,38,63,0.03)] h-80">
            <div class="p-4">
              <div class="flex justify-between items-center">
                <h4 class="font-medium text-[#373753]">{{ capitalize(month.month) }}</h4>
                <div class="flex items-center">
                  <p class="text-base font-bold text-[#373753]">{{ month.count }}</p>
                  <span class="ml-2 text-sm text-[#677C92]">{{ month.count === 1 ? 'Relato' : 'Relatos' }}</span>
                </div>
              </div>
            </div>
            <hr class="border-t border-gray-200">
            <div class="p-4 mt-2 space-y-2 overflow-y-auto max-h-56 custom-scroll">
              <div v-for="(comment, i) in month.comments" :key="i" class="text-sm text-gray-600 bg-red-50 p-2 rounded">
                {{ comment }}
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.tab-button {
  padding: 0.5rem 1rem;
  border: 1px solid #ccc;
  border-radius: 0.375rem;
  background-color: #f9f9f9;
  cursor: pointer;
  transition: background-color 0.3s;
}

.tab-button.active {
  background-color: #e2e8f0;
  border-color: #cbd5e0;
}

.tab-button:hover {
  background-color: #edf2f7;
}

.custom-scroll {
  scrollbar-width: thin;
  scrollbar-color: #cbd5e0 transparent;
}

.custom-scroll::-webkit-scrollbar {
  width: 4px;
}

.custom-scroll::-webkit-scrollbar-track {
  background: transparent;
}

.custom-scroll::-webkit-scrollbar-thumb {
  background-color: #cbd5e0;
  border-radius: 10px;
}

html, body {
  height: 100%;
  overflow-y: auto;
}

/* Ajustando espaçamentos globais */
.space-y-6 > * + * {
  margin-top: 24px;
}

.gap-6 {
  gap: 24px;
}

.mb-6 {
  margin-bottom: 24px;
}

.p-6 {
  padding: 24px;
}

.pt-6 {
  padding-top: 24px;
}

.mt-6 {
  margin-top: 24px;
}

.py-6 {
  padding-top: 24px;
  padding-bottom: 24px;
}

.px-6 {
  padding-left: 24px;
  padding-right: 24px;
}

/* Ajustes de espaçamento para os cards */
.card-padding {
  padding: 16px 24px 24px 24px; /* Top, Right, Bottom, Left */
}

.analysis-card {
  background-color: #fff;
  border: 1px solid #E1E9F4;
  border-radius: 10px;
  box-shadow: 0 12px 24px 0 rgba(18, 38, 63, 0.03);
  padding: 16px 24px 24px 24px; /* Top, Right, Bottom, Left */
  margin-bottom: 24px;
}

.filter-button {
  background-color: #E1E9F4;
  border: none;
  padding: 0 16px;
  margin-left: 8px;
  border-radius: 8px;
  cursor: pointer;
  font-size: 14px;
  color: #333;
  height: 28px;
  display: flex;
  align-items: center;
  transition: background-color 0.2s ease;
}

.filter-button:hover {
  background-color: #d1ddef;
}

.active-filter {
  background-color: #7E9CF7;
  color: white;
}

.active-filter:hover {
  background-color: #7E9CF7;
}

.analysis-title {
  font-size: 18px;
  font-weight: bold;
  margin-bottom: 0;
}

.title-divider {
  height: 1px;
  background-color: #E1E9F4;
  margin: 0 -24px 20px -24px;  /* Margens negativas para compensar o padding */
}
</style>