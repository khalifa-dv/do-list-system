<script setup>
import { ref, reactive, computed, watch } from 'vue'
import { useStore } from 'vuex'
import ConfirmationModal from '@/components/modals/ConfirmationModal.vue'
import Button from '@/components/base/Button.vue'
import MoveIcon from '@/components/icons/MoveIcon.vue'
import TrashIcon from '@/components/icons/TrashIcon.vue'
import PlusIcon from '@/components/icons/PlusIcon.vue'
import CloseIcon from '@/components/icons/CloseIcon.vue'
import SaveIcon from '@/components/icons/SaveIcon.vue'
import { container } from 'tailwindcss/defaultTheme'

const props = defineProps({
  payload: {
    type: Object,
    required: true,
    default: () => { },
  },
})

const emits = defineEmits(['addContainer', 'addCard'])

const store = useStore()
const state = reactive({
  isDraggable: false,
  isAddingContainer: false,
  isRemovingContainer: false,
  isRemovingCard: false,
  selectedContainerId: null,
  selectedCardId: null,
  tempCards: [],
})
const newContainerTitle = ref(null)
const newCardData = reactive({
  id: null,
  container_id: null,
  title: null,
  content: null,
  created_at: null,
})
const vuello = reactive({
  title: null,
  last_modified: null,
  containers: [],
  cards: [],
})

watch(
  () => props.payload,
  (newValue) => {
    console.log(newValue)
    vuello.title = newValue.title
    vuello.last_modified = newValue.last_modified
    vuello.containers = newValue.containers
    vuello.cards = newValue.cards
  },
  { immediate: true }
)
watch(
  () => state.isAddingContainer,
  (newValue) => {
    if (!newValue) {
      newContainerTitle.value = null
    }
  },
  { deep: true }
)

const cardList = (containerId) => {

  return vuello.cards.filter((card) => card.container_id === containerId)
}
const dragItem = (event, item) => {
  
  event.dataTransfer.dropEffect = 'move'
  event.dataTransfer.effectAllowed = 'move'
  event.dataTransfer.setData('id', item.id)
}
const dropItem = async (event, containerId) => {
  const id = event.dataTransfer.getData('id')
  const item = vuello.cards.find((card) => card.id == id)
  item.container_id = containerId

    try {
    await axios.patch(`/api/tasks/switchContainer/${id}`, {'container_id':containerId});
    // Success! Handle any additional logic here
  } catch (error) {
    // Error occurred. Handle or display the error message
    console.error(error);
  }
  vuello.last_modified = new Date().toLocaleString('en-GB')
  store.dispatch('vuello/setVuello', vuello)
}
const handleDeleteItem = async (type, id) => {
  if (type === 'container') {
    state.selectedContainerId = id
    state.isRemovingContainer = true
         try {
      await axios.delete(`/api/containers/${id}`);
      // Success! Handle any additional logic here
    } catch (error) {
      // Error occurred. Handle or display the error message
      console.error(error);
    }
  } else {
    state.selectedCardId = id
    state.isRemovingCard = true
    try {
      await axios.delete(`/api/tasks/${id}`);
      // Success! Handle any additional logic here
    } catch (error) {
      // Error occurred. Handle or display the error message
      console.error(error);
    }
  }
}
const deleteItem = (type) => {
  if (type === 'container') {
    vuello.containers = vuello.containers.filter(
      (container) => container.id !== state.selectedContainerId
    )
    vuello.cards = vuello.cards.filter(
      (card) => card.container_id !== state.selectedContainerId
    )
  } else {
    vuello.cards = vuello.cards.filter(
      (card) => card.id !== state.selectedCardId
    )
  }
  vuello.last_modified = new Date().toLocaleString('en-GB')
  store.dispatch('vuello/setVuello', vuello)
  state.isRemovingContainer = false
  state.isRemovingCard = false
}
const handleKanbanAction = async (mode, type, containerId, payload) => {
  if (mode === 'add') {
    if (type === 'container') {
      const newContainer = {
        id:
          vuello.containers.length > 0
            ? [...vuello.containers].pop().id + 1
            : 1,
        name: newContainerTitle.value,
      }
      vuello.containers.push(newContainer)
          try {
        await axios.post('/api/containers/', newContainer);
        // Success! Handle any additional logic here
      } catch (error) {
        // Error occurred. Handle or display the error message
        console.error(error);
      }
      state.isAddingContainer = false
    } else {
      newCardData.id =
        vuello.cards.length > 0 ? [...vuello.cards].pop().id + 1 : 1
      newCardData.container_id = containerId
      const newCard = {
        id: newCardData.id,
        container_id: newCardData.container_id,
        title: newCardData.title,
        content: newCardData.content,
      }
      vuello.cards.push(newCard)
      try {
        await axios.post('/api/tasks/', newCard);
        // Success! Handle any additional logic here
      } catch (error) {
        // Error occurred. Handle or display the error message
        console.error(error);
      }
      payload.is_adding_card = false
    }
  } else if (mode === 'delete') {
    if (type === 'container') {
      vuello.containers = vuello.containers.filter(
        (container) => container.id !== containerId
      )
      state.isRemovingContainer = false
        try {
        await axios.delete(`/api/containers/${id}`);
        // Success! Handle any additional logic here
      } catch (error) {
        // Error occurred. Handle or display the error message
        console.error(error);
      }
    }
    vuello.cards = vuello.cards.filter(
      (card) => card.container_id !== containerId
    )
  }
  if (payload) {
    payload.is_editing_container
      ? (payload.is_editing_container = false)
      : (payload.is_editing_card = false)
  }
  vuello.last_modified = new Date().toLocaleString('en-GB')
  store.dispatch('vuello/setVuello', vuello)
  newCardData.id = null
  newCardData.container_id = null
  newCardData.title = null
  newCardData.content = null
}
const handleEditCard = async (type, selectedCard) => {
  if (type === 'change') {
    selectedCard.is_editing_card = true
    state.tempCards = vuello.cards.map((card) => ({ ...card }))
  } else if (type === 'save') {
    handleKanbanAction(null, null, null, selectedCard)
    try {
      await axios.put(`/api/tasks/${selectedCard.id}`, selectedCard);
      // Success! Handle any additional logic here
    } catch (error) {
      // Error occurred. Handle or display the error message
      console.error(error);
    }
  } else {
    const data = state.tempCards.find((card) => card.id === selectedCard.id)
    selectedCard.title = data.title
    selectedCard.content = data.content
    selectedCard.is_editing_card = false
  }
}
</script>

<template>
  <div class="flex h-full w-full overflow-auto rounded-lg bg-[#F2F3F9] px-2 py-3">
    <TransitionGroup name="list">
      <div v-for="container in vuello.containers" :key="container.id" class="mx-1">
        <div class="min-h-[50px] min-w-[300px] max-w-[300px] rounded-lg bg-[#E4E5EC] p-1"
          @drop="dropItem($event, container.id)" @dragenter.prevent @dragover.prevent>
          <div class="flex h-full w-full place-items-center justify-between p-1">
            <Transition name="fade" mode="out-in">
              <input v-if="container.is_editing_container" v-model="container.name" type="text"
                class="block w-full rounded-lg border border-gray-300 bg-gray-50 p-2.5 text-sm text-gray-900 transition duration-300 ease-in-out focus:border-blue-500 focus:ring-blue-500"
                placeholder="Add Title" @keypress.enter="
                  handleKanbanAction(null, null, null, container)
                  " @blur="container.is_editing_container = false" />
              <div v-else class="text-md my-[0.30rem] w-full cursor-pointer p-1 font-semibold"
                @click="container.is_editing_container = true">
                {{ container.name }} ({{ cardList(container.id).length }})
              </div>
            </Transition>
            <TrashIcon height="25px"
              class="cursor-pointer rounded-full p-1 text-red-400 hover:bg-red-200 hover:text-red-700"
              @click="handleDeleteItem('container', container.id)" />
          </div>
          <div class="flex flex-col overflow-y-auto" style="max-height: calc(100vh - 165px)">
            <div v-for="card in cardList(container.id)" :key="card.id" class="m-1 cursor-pointer rounded-lg bg-white p-2"
              :draggable="state.isDraggable" @dragstart="dragItem($event, card)">
              <Transition name="fade" mode="out-in">
                <div v-if="card.is_editing_card" class="flex w-full flex-col rounded-md border-gray-400 bg-white">
                  <input v-model="card.title" type="text"
                    class="mb-2 block w-full rounded-lg border border-gray-300 bg-gray-50 p-2.5 text-sm text-gray-900 transition duration-300 ease-in-out focus:border-blue-500 focus:ring-blue-500"
                    placeholder="Add Card Title" />
                  <textarea v-model="card.content"
                    class="block w-full rounded-lg border border-gray-300 bg-gray-50 p-2.5 text-sm text-gray-900 transition duration-300 ease-in-out focus:border-blue-500 focus:ring-blue-500"
                    placeholder="Add Card Content" />
                         <input v-model="card.priority" type="number" max="5" min="0"
                        class="mb-2 mt-1 block w-full rounded-lg border border-gray-300 bg-gray-50 p-2.5 text-sm text-gray-900 transition duration-300 ease-in-out focus:border-blue-500 focus:ring-blue-500"
                        placeholder="Add Card priority" 
                           />
                
                  <div class="mt-2 flex w-full place-items-center justify-between">
                    <Button model="outline" size="sm" rounded="sm" @click="handleDeleteItem('card', card.id)">
                      Delete Card {{ index }}
                    </Button>
                    <div class="flex">
                      <SaveIcon height="30px"
                        class="mr-2 cursor-pointer rounded-full bg-blue-500 p-1 text-white hover:bg-blue-700"
                        @click="handleEditCard('save', card)" />
                      <CloseIcon height="30px"
                        class="cursor-pointer rounded-full p-1 text-red-500 hover:bg-red-600 hover:text-white"
                        @click="handleEditCard('cancel', card)" />
                    </div>
                  </div>
                </div>
                <div v-else>
                  <div class="flex flex-col" @click="handleEditCard('change', card)">
                    <div class="flex place-items-center justify-between">
                      <h1 class="text-sm font-semibold">
                        {{ card.title }}
                      </h1>
                      <MoveIcon height="25px"
                        class="cursor-grab rounded-full p-1 text-gray-400 hover:bg-gray-200 hover:text-gray-700"
                        @click.stop="" @mouseenter="state.isDraggable = true" @mouseleave="state.isDraggable = false" />
                    </div>
                    <pre class="mt-1 whitespace-pre-wrap font-sans text-sm">{{
                      card.content
                    }}</pre>
                  </div>
                </div>
              </Transition>
            </div>
            <div class="m-1 flex flex-col place-items-center justify-center">
              <Transition name="fade">
                <div v-if="container.is_adding_card" class="flex w-full flex-col rounded-md border-gray-400 bg-white p-2">
                  <input v-model="newCardData.title" type="text"
                    class="mb-2 block w-full rounded-lg border border-gray-300 bg-gray-50 p-2.5 text-sm text-gray-900 transition duration-300 ease-in-out focus:border-blue-500 focus:ring-blue-500"
                    placeholder="Add Card Title" @keypress.enter="
                      handleKanbanAction('add', 'card', container.id, container)
                      " />
                  <textarea v-model="newCardData.content"
                    class="block w-full rounded-lg border border-gray-300 bg-gray-50 p-2.5 text-sm text-gray-900 transition duration-300 ease-in-out focus:border-blue-500 focus:ring-blue-500"
                    placeholder="Add Card Content" />
                       <input v-model="newCardData.priority" type="number" max="5" min="0"
                      class="mb-2 block w-full rounded-lg border border-gray-300 bg-gray-50 p-2.5 text-sm text-gray-900 transition duration-300 ease-in-out focus:border-blue-500 focus:ring-blue-500"
                      placeholder="Add Card priority" 
                         />
                </div>
              </Transition>
              <Transition name="fade" mode="out-in">
                <div v-if="container.is_adding_card" class="mt-2 flex w-full place-items-center justify-end">
                  <SaveIcon height="30px"
                    class="mr-2 cursor-pointer rounded-full bg-blue-500 p-1 text-white hover:bg-blue-700" @click="
                      handleKanbanAction('add', 'card', container.id, container)
                      " />
                  <CloseIcon height="30px"
                    class="cursor-pointer rounded-full p-1 text-red-500 hover:bg-red-600 hover:text-white"
                    @click="container.is_adding_card = false" />
                </div>
                <Button v-else type="primary" model="outline" size="sm" rounded="sm" class="mt-1"
                  @click="container.is_adding_card = true">
                  <PlusIcon height="15px" />
                  Add More Card
                </Button>
              </Transition>
            </div>
          </div>
        </div>
      </div>
    </TransitionGroup>
    <div class="mx-1">
      <div class="flex min-h-[50px] min-w-[300px] flex-col place-items-center justify-center rounded-lg bg-[#E4E5EC] p-2">
        <Transition name="fade">
          <input v-if="state.isAddingContainer" v-model="newContainerTitle" type="text"
            class="mb-2 block w-full rounded-lg border border-gray-300 bg-gray-50 p-2.5 text-sm text-gray-900 transition duration-300 ease-in-out focus:border-blue-500 focus:ring-blue-500"
            placeholder="Add Title" @keypress.enter="handleKanbanAction('add', 'container')" />
        </Transition>
        <Transition name="fade" mode="out-in">
          <div v-if="state.isAddingContainer" class="mb-1 flex w-full place-items-center justify-end">
            <SaveIcon height="30px" class="mr-2 cursor-pointer rounded-full bg-blue-500 p-1 text-white hover:bg-blue-700"
              @click="handleKanbanAction('add', 'container')" />
            <CloseIcon height="30px"
              class="cursor-pointer rounded-full p-1 text-red-500 hover:bg-red-600 hover:text-white"
              @click="state.isAddingContainer = false" />
          </div>
          <Button v-else type="primary" model="outline" size="sm" rounded="sm" @click="state.isAddingContainer = true">
            <PlusIcon height="15px" />
            Add More List
          </Button>
        </Transition>
      </div>
    </div>
  </div>

  <!-- Confirmation Modal -->
  <ConfirmationModal :value="state.isRemovingContainer" @confirm="deleteItem('container')"
    @close="state.isRemovingContainer = false">
    All Contents inside this Container will also be deleted. Are you sure?
  </ConfirmationModal>

  <ConfirmationModal :value="state.isRemovingCard" @confirm="deleteItem('card')" @close="state.isRemovingCard = false">
    Are you sure want to delete this card?
  </ConfirmationModal>
</template>

<style>
.list-enter-active,
.list-leave-active {
  transition: all 0.5s ease-in-out;
}

.list-enter-from,
.list-leave-to {
  opacity: 0;
  transform: translateY(-30px);
}
</style>
