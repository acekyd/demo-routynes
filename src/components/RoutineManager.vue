<template>
  <div class="space-y-8">
    <!-- Completed Routines Counter -->
    <div class="text-center py-4">
      <h2 class="text-lg font-semibold">Completed Routines: {{ completedRoutines }}</h2>
    </div>

    <!-- Ongoing Routines -->
    <section v-if="ongoingRoutines.length">
      <h2 class="text-2xl font-bold section-title">Ongoing Routines</h2>
      <ul class="space-y-4">
        <li v-for="(ongoingRoutine, index) in ongoingRoutines" :key="index" class="bg-card list-item w-full">
          <div class="flex justify-between items-center">
            <h3 class="text-xl font-bold">{{ ongoingRoutine.name }}</h3>
          </div>
          <p class="steps-title">Steps</p>
          <ul class="mt-2 space-y-1">
            <li v-for="(step, sIndex) in ongoingRoutine.checklist" :key="sIndex" class="flex items-center">
              <input type="checkbox" v-model="step.completed" class="mr-2 checkbox" />
              <span class="text-left">{{ step.name }}</span>
            </li>
          </ul>
          <button 
            v-if="canCompleteRoutine(ongoingRoutine)"
            @click="completeRoutine(index)"
            class="w-full py-2 mt-2 bg-green-600 text-white font-semibold hover:bg-green-700 transition"
          >
            Complete Routine
          </button>
        </li>
      </ul>
    </section>

    <!-- Available Routines -->
    <section>
      <h2 class="text-2xl font-bold section-title">Available Routines</h2>
      <ul class="space-y-4">
        <li v-for="(routine, index) in routines" :key="index" class="bg-card p-4 list-item w-full">
          <div 
            class="flex justify-between items-center cursor-pointer" 
            @click="toggleSteps(index)"
          >
            <h3 class="text-xl font-bold">{{ routine.name }}</h3>
            <div class="flex space-x-2">
              <button 
                class="text-blue-600 hover:underline mr-2"
                @click.stop="handleStartRoutine(index)"
              >
                <i class="fas fa-play"></i>
              </button>
              <button 
                class="text-red-600 hover:underline"
                @click.stop="deleteRoutine(index)"
              >
                <i class="fas fa-trash"></i>
              </button>
            </div>
          </div>
          <p class="steps-title" v-if="routine.showSteps && routine.steps.length">Steps</p>
          <ul v-if="routine.showSteps" class="mt-2 space-y-1">
            <li v-for="(step, sIndex) in routine.steps" :key="sIndex" class="flex items-center">
              <span class="text-left w-full">{{ step }}</span>
            </li>
          </ul>
          <form v-if="routine.showSteps" @submit.prevent="addStep(index)" class="flex mt-2">
            <input 
              v-model="newStep[index]"
              type="text" 
              placeholder="Add a step" 
              class="flex-1 input border rounded focus:outline-none focus:ring-2 focus:ring-blue-600"
            />
            <button 
              type="submit" 
              class="ml-2 px-3 py-1 bg-green-600 text-white font-semibold rounded hover:bg-green-700 transition"
            >
              Add Step
            </button>
          </form>
        </li>
      </ul>
      <div class="flex justify-center">
        <button 
          @click="showAddRoutineForm = !showAddRoutineForm" 
          class="py-2 mt-4 bg-blue-600 text-white font-semibold rounded hover:bg-blue-700 transition"
        >
          Create New Routine
        </button>
      </div>
    </section>

    <form v-if="showAddRoutineForm" @submit.prevent="addRoutine" class="space-y-2 mt-4">
      <input 
        v-model="newRoutineName" 
        type="text" 
        placeholder="New routine name" 
        class="w-full input border rounded focus:outline-none focus:ring-2 focus:ring-blue-600"
      />
      <button 
        type="submit" 
        class="w-full py-2 bg-green-600 text-white font-semibold rounded hover:bg-green-700 transition"
      >
        Add Routine
      </button>
    </form>

    <!-- Add Steps Prompt Modal -->
    <div v-if="showAddStepsModal" class="fixed inset-0 bg-gray-800 bg-opacity-75 flex items-center justify-center">
      <div class="modal">
        <h3 class="text-lg font-semibold">Routine has no steps</h3>
        <p>Add steps to the routine before starting.</p>
        <button 
          @click="closeAddStepsModal"
          class="mt-4 px-3 py-1 bg-blue-600 text-white font-semibold rounded hover:bg-blue-700 transition"
        >
          OK
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, watch } from 'vue'

const LOCAL_STORAGE_KEY = 'routines'
const COMPLETED_ROUTINES_KEY = 'completedRoutines'

const newRoutineName = ref('')
const routines = ref([])
const newStep = ref({})
const showAddRoutineForm = ref(false)

const ongoingRoutines = ref([])
const completedRoutines = ref(0)
const showAddStepsModal = ref(false)

onMounted(() => {
  const storedRoutines = localStorage.getItem(LOCAL_STORAGE_KEY)
  if (storedRoutines) {
    routines.value = JSON.parse(storedRoutines)
  }
  const storedCompletedCount = localStorage.getItem(COMPLETED_ROUTINES_KEY)
  if (storedCompletedCount) {
    completedRoutines.value = JSON.parse(storedCompletedCount)
  }
})

watch(routines, (newRoutines) => {
  localStorage.setItem(LOCAL_STORAGE_KEY, JSON.stringify(newRoutines))
}, { deep: true })

watch(completedRoutines, (newCount) => {
  localStorage.setItem(COMPLETED_ROUTINES_KEY, JSON.stringify(newCount))
})

function addRoutine() {
  if (newRoutineName.value.trim() === '') return
  routines.value.push({ name: newRoutineName.value, steps: [], showSteps: false })
  newRoutineName.value = ''
  showAddRoutineForm.value = false
}

function toggleSteps(index) {
  routines.value[index].showSteps = !routines.value[index].showSteps
}

function handleStartRoutine(routineIndex) {
  const routine = routines.value[routineIndex]
  if (routine.steps.length > 0) {
    startRoutine(routineIndex)
  } else {
    showAddStepsModal.value = true
  }
}

function startRoutine(routineIndex) {
  const routineToStart = routines.value[routineIndex]
  ongoingRoutines.value.push({
    name: routineToStart.name,
    checklist: routineToStart.steps.map(step => ({ name: step, completed: false }))
  })
}

function closeAddStepsModal() {
  showAddStepsModal.value = false
}

function deleteRoutine(index) {
  routines.value.splice(index, 1)
}

function addStep(routineIndex) {
  if (!newStep.value[routineIndex] || newStep.value[routineIndex].trim() === '') return
  routines.value[routineIndex].steps.push(newStep.value[routineIndex])
  newStep.value[routineIndex] = ''
}

function canCompleteRoutine(ongoingRoutine) {
  return ongoingRoutine.checklist.length > 0 && ongoingRoutine.checklist.every(step => step.completed)
}

function completeRoutine(index) {
  ongoingRoutines.value.splice(index, 1)
  completedRoutines.value += 1
}
</script>

<style scoped>
@import 'https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0-beta3/css/all.min.css';
</style>
