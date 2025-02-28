<script setup lang="ts">
import { ref, computed, onMounted, watch } from 'vue'
import { useFeedbackStore } from '../stores/feedback'
import { subDays } from 'date-fns'
import type { FeedbackData } from '../stores/feedback'

const feedbackStore = useFeedbackStore()
const isTableExpanded = ref(false)
const isTableMounted = ref(false)

// Estado para controle da ordenação
const sortField = ref<keyof FeedbackData>('user_id') // Exemplo de inicialização
const sortDirection = ref('desc') // Direção inicial (decrescente)

// Watch para dados do feedbackStore
watch(() => feedbackStore.feedbackData, (newData) => {
  if (newData.length > 0) {
    // Quando novos dados são carregados, filtra automaticamente para 30 dias
    const today = new Date()
    const thirtyDaysAgo = subDays(today, 30)
    feedbackStore.filterFeedback('month', [], [], null, null)
    isTableExpanded.value = true
    isTableMounted.value = true
  }
}, { immediate: true })

const tableData = computed(() => 
  isTableMounted.value ? feedbackStore.filteredData : []
)

const getNPSZone = (score: number) => {
  if (score >= 75) return { zone: 'Excelência', color: 'bg-emerald-100 text-emerald-800', textColor: 'text-emerald-800' }
  if (score >= 50) return { zone: 'Qualidade', color: 'bg-green-100 text-green-800', textColor: 'text-green-800' }
  if (score >= 0) return { zone: 'Melhoria', color: 'bg-yellow-100 text-yellow-800', textColor: 'text-yellow-800' }
  return { zone: 'Crítica', color: 'bg-red-100 text-red-800', textColor: 'text-red-800' }
}

const toggleTable = () => {
  isTableExpanded.value = !isTableExpanded.value
  if (isTableExpanded.value && !isTableMounted.value) {
    isTableMounted.value = true
  }
}

// Função para alternar ordenação
const toggleSort = (field: string) => {
  if (sortField.value === field) {
    // Se já está ordenando por este campo, inverte a direção
    sortDirection.value = sortDirection.value === 'asc' ? 'desc' : 'asc'
  } else {
    // Se é um novo campo, define ele como padrão e direção descendente
    sortField.value = field as keyof FeedbackData
    sortDirection.value = 'desc'
  }
}

// Dados ordenados
const sortedData = computed(() => {
  return [...tableData.value].sort((a, b) => {
    const aValue = Number(a[sortField.value as keyof FeedbackData])
    const bValue = Number(b[sortField.value as keyof FeedbackData])
    
    if (sortField.value === 'score') {
      return sortDirection.value === 'asc' 
        ? aValue - bValue 
        : bValue - aValue
    }
    
    if (sortField.value === 'created_at') {
      return sortDirection.value === 'asc'
        ? new Date(aValue).getTime() - new Date(bValue).getTime()
        : new Date(bValue).getTime() - new Date(aValue).getTime()
    }
    
    return 0
  })
})
</script>

<template>
  <div class="flex-1 bg-[#F9FAFC]">
    <div class="bg-white rounded-[10px] border border-[#E1E9F4] shadow-[0_12px_24px_0_rgba(23,18,63,0.03)]">
      <div 
        @click="toggleTable"
        class="flex items-center justify-between p-6 cursor-pointer hover:bg-gray-50"
      >
        <h3 class="text-lg font-medium text-[#373753]">Detalhes do NPS</h3>
        <svg 
          class="w-6 h-6 text-gray-500 transition-transform duration-200"
          :class="{ 'transform rotate-180': isTableExpanded }"
          xmlns="http://www.w3.org/2000/svg" 
          fill="none" 
          viewBox="0 0 24 24" 
          stroke="currentColor"
        >
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7" />
        </svg>
      </div>
      
      <div v-show="isTableExpanded" class="border-t border-[#E1E9F4]">
        <div v-if="isTableMounted" class="overflow-x-auto">
          <table class="min-w-full divide-y divide-gray-200">
            <thead class="bg-[#F0F4FA]">
              <tr>
                <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">ID</th>
                <th 
                  @click="toggleSort('score')"
                  class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider cursor-pointer hover:bg-gray-100"
                >
                  <div class="flex items-center gap-2">
                    Nota
                    <svg 
                      class="w-4 h-4 transition-transform"
                      :class="{ 'transform rotate-180': sortField === 'score' && sortDirection === 'desc' }"
                      viewBox="0 0 24 24"
                      fill="none"
                      stroke="currentColor"
                    >
                      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 15l7-7 7 7" />
                    </svg>
                  </div>
                </th>
                <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Status</th>
                <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Comentário</th>
                <th 
                  @click="toggleSort('created_at')"
                  class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider cursor-pointer hover:bg-gray-100"
                >
                  <div class="flex items-center gap-2">
                    Data
                    <svg 
                      class="w-4 h-4 transition-transform"
                      :class="{ 'transform rotate-180': sortField === 'created_at' && sortDirection === 'desc' }"
                      viewBox="0 0 24 24"
                      fill="none"
                      stroke="currentColor"
                    >
                      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 15l7-7 7 7" />
                    </svg>
                  </div>
                </th>
                <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Empresa</th>
                <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Função</th>
              </tr>
            </thead>
            <tbody class="bg-white divide-y divide-gray-200">
              <tr v-for="item in sortedData" :key="item.user_id">
                <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-900">{{ item.user_id }}</td>
                <td class="px-6 py-4 whitespace-nowrap text-sm" :class="getNPSZone(item.score).textColor">{{ item.score }}</td>
                <td class="px-6 py-4 whitespace-nowrap text-sm">
                  <span :class="[getNPSZone(item.score).color, 'text-xs font-medium px-2.5 py-0.5 rounded-full']">
                    {{ getNPSZone(item.score).zone }}
                  </span>
                </td>
                <td class="px-6 py-4 text-sm text-gray-900">{{ item.reason }}</td>
                <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-900">{{ item.created_at }}</td>
                <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-900">{{ item.company_id }}</td>
                <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-900">{{ item.role }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>