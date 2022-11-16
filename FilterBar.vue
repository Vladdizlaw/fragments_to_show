
<script setup>
import { ref, reactive, onMounted, watch, computed, defineEmits, defineProps, onBeforeMount, onBeforeUpdate } from 'vue';
import { Popover, PopoverButton, PopoverPanel } from '@headlessui/vue'
import axios from 'axios';
const emit = defineEmits(['update:selectedFilters']);
const props = defineProps({
    selectedFilters: {
        type: Object, default: {}
    }
})
const addfilters = ref(null)//Кнопка добавить фильтры
const state = reactive({
    currentFilters: [
        { title: 'Дата начала', subtitle: ['Начало', 'Окончание'], type: 'datePicker', fieldName: 'start_date' },
        { title: 'Город', type: 'checkbox', fieldName: 'city_id', data: [] },
        { title: 'Куратор', type: 'checkbox', fieldName: 'created_id', data: [] },
        { title: 'Заказчик', type: 'checkbox', fieldName: 'customer_id', data: [] },
        {title: 'Статус у тайника', type: 'checkbox', fieldName: 'progress', data: [
                { id: 1, name: 'Приступил к проверке (физ.)' },
                { id: 2, name: 'Закончил проверку(физ.)' },
                { id: 3, name: 'Заполнил отчёт' },
                { id: 4, name: 'Проверка успешно выполнена' },
                { id: 5, name: 'Проверка аннулирована' },
                { id: 6, name: 'Отправлена заявка' },
                { id: 7, name: 'Назначен на проверку'},
                { id: 8, name: 'Проверка оплачена' },               
            ]
        },
    ],
    bufferChosenFilters: { currentFilters: [], allowedFilters: [], selectedFilters: { ...props.selectedFilters } },
    showCurrentFilters: true,
    showAllowedFilters: true,
    allowedFilters: [
        { title: 'Тип проверки', type: 'checkbox', fieldName: 'type', data: [{ id: 'visit', name: 'Визит' }, { id: 'complex', name: 'Комплекс' }, { id: 'call', name: 'Звонок' }] },
        { title: 'Чек-листы', type: 'checkbox', fieldName: 'checklist_id', data: [] },
        { title: 'Объекты', type: 'checkbox', fieldName: 'entity_id', data: [] },
        { title: 'Инструкция', type: 'checkbox', fieldName: 'instruction_id', data: [] },
        { title: 'Дата фактической проверки ', subtitle: ['Начало ', 'Окончание'], type: 'datePicker', fieldName: 'evaluated_at' },
        { title: 'Дата окончания ', subtitle: ['Начало ', 'Окончание'], type: 'datePicker', fieldName: 'end_date' },
        { title: 'Дата публикация', subtitle: ['Начало ', 'Окончание'], type: 'datePicker', fieldName: 'published_at' },
        { title: 'Дата создания ', subtitle: ['Начало ', 'Окончание'], type: 'datePicker', fieldName: 'created_at' },
        { title: 'Дата истечения', subtitle: ['Начало ', 'Окончание'], type: 'datePicker', fieldName: 'expiration_date' }
    ],
    searchFilters: '',
    city_id: [],
    customer_id: [],
    created_id: [],
    checklist_id: [],
    entity_id: [],
    instruction_id: [],

})
function handlePopupFilter(fieldName) {
    if (props.selectedFilters[fieldName]) {
        state.bufferChosenFilters.selectedFilters[fieldName] = [...props.selectedFilters[fieldName]]
    } else {
        state.bufferChosenFilters.selectedFilters[fieldName] = []
    }
    state.searchFilters = ''


}
async function getCitiesWithAuditsOnly() {
    try {
        const params = {
            for_all_audits: true
        }
        const cities = await axios
            .get('v2/entities/cities', { params })
            .then((res) => res.data)

        return cities.results
    } catch (e) { }
    return []
}
async function getCustomers() {
    try {
        const res = await axios.get('v2/users/roles/customer')
        return res.data
    } catch (e) { }
    return []
}
async function getChecklists() {
    try {
        const res = await axios.get('v3/moderator/checklists')
        return res.data.checklists
    } catch (e) { }
    return []
}
async function getCreators() {
    try {
        const res = await axios.get('v2/users/audit-creator')
        return res.data
    } catch (e) { }
    return []
}
async function getEntities() {
    try {
        const res = await axios.get('v3/moderator/entities')
        return res.data.entities
    } catch (e) { }
    return []
}
async function getInstructions() {
    try {
        const res = await axios.get('v3/moderator/instructions')
        return res.data.instructions
    } catch (e) { }
    return []
}
function isAddFiltersButtonInLeft() {
    const screenWidth = window.screen.width - 208
    const xCooordsButton = addfilters?.value?.el?.getBoundingClientRect().x
    if (xCooordsButton < screenWidth / 2) {
        return true
    }
    if (xCooordsButton > screenWidth / 2) {
        return false
    }
}
function resetAllFilters() {
    emit('update:selectedFilters', {})
    state.bufferChosenFilters.selectedFilters = {}
    const fieldsDefaultCurrentFilters = ['start_date', 'city_id', 'created_id', 'customer_id', 'progress']
    const currentFiltersInAllowedFilters = state.allowedFilters.filter(f => { return fieldsDefaultCurrentFilters.includes(f.fieldName) })
    const allowedFiltersInCurrentFilters = state.currentFilters.filter(f => { return !fieldsDefaultCurrentFilters.includes(f.fieldName) })
    state.currentFilters = state.currentFilters.filter(f => { return fieldsDefaultCurrentFilters.includes(f.fieldName) })
    state.allowedFilters = state.allowedFilters.filter(f => { return !fieldsDefaultCurrentFilters.includes(f.fieldName) })
    state.currentFilters = [...state.currentFilters, ...currentFiltersInAllowedFilters]
    state.allowedFilters = [...state.allowedFilters, ...allowedFiltersInCurrentFilters]

}
function addMoreFiters() {
    state.bufferChosenFilters.currentFilters = [...state.currentFilters]
    state.bufferChosenFilters.allowedFilters = [...state.allowedFilters]
    state.showAllowedFilters = true
    state.showCurrentFilters = true
    state.searchFilters = ''


}
function handleAddFilter(filter) {
    if (state.bufferChosenFilters.currentFilters.includes(filter)) {
        state.bufferChosenFilters.currentFilters = state.bufferChosenFilters.currentFilters.filter((el) => el.title !== filter.title)
        state.bufferChosenFilters.allowedFilters.push(filter)
    } else if (state.bufferChosenFilters.allowedFilters.includes(filter)) {
        state.bufferChosenFilters.allowedFilters = state.bufferChosenFilters.allowedFilters.filter((el) => el.title !== filter.title)
        state.bufferChosenFilters.currentFilters.push(filter)
    }

}
function resetFilter(fieldName) {
    state.bufferChosenFilters.selectedFilters[fieldName] = []
}
function applyChosenFilter(close, fieldName) {
    props.selectedFilters[fieldName] = state.bufferChosenFilters.selectedFilters[fieldName] ? [...state.bufferChosenFilters.selectedFilters[fieldName]] : undefined
    close()
}
function applyChosenFilters(close) {
    state.currentFilters = [...state.bufferChosenFilters.currentFilters]
    state.bufferChosenFilters.currentFilters = [...state.currentFilters]
    state.allowedFilters = [...state.bufferChosenFilters.allowedFilters]
    state.bufferChosenFilters.allowedFilters = [...state.allowedFilters]
    close()
}
watch(props.selectedFilters, (value) => {
    emit('update:selectedFilters', value)
})
function showPeriodFiltersFill(filter,value){
   //Отображение титла кнопки с периодами когда период выбран
    return `${filter.title}: От ${value[0].split('.')[0]}.${value[0].split('.')[1]} До ${value[1].split('.')[0]}.${value[1].split('.')[1]}`
}
function showedFilters(filter) {
    return filter.data.filter(el => {
        if (el.name) {
            return el['name'].toLowerCase().includes(state.searchFilters.toLowerCase()) || String(el['id']).toLowerCase().includes(state.searchFilters.toLowerCase())
        }
        if (el['last_name']) {
            return el['last_name'].toLowerCase().includes(state.searchFilters.toLowerCase()) || String(el['id']).toLowerCase().includes(state.searchFilters.toLowerCase())
        }
        if (el.title) {
            return el['title'].toLowerCase().includes(state.searchFilters.toLowerCase()) || String(el['id']).toLowerCase().includes(state.searchFilters.toLowerCase())
        }
    })
}
const showedCurrentFilters = computed(() => { 
    return state.bufferChosenFilters.currentFilters.filter(el => {
        if (el.title) {
            return el['title'].toLowerCase().includes(state.searchFilters?.toLowerCase())
        }
    })
})
const showedAllowedFilters = computed(() => {
    return state.bufferChosenFilters.allowedFilters.filter(el => {
        if (el.title) {
            return el['title'].toLowerCase().includes(state.searchFilters?.toLowerCase())
        }
    })
})


onBeforeUpdate(() => {
    sessionStorage.setItem('current_filters', JSON.stringify(state.currentFilters))
    sessionStorage.setItem('allowed_filters', JSON.stringify(state.allowedFilters))



})
onBeforeMount(() => {
    if (sessionStorage.getItem('current_filters') !== null && sessionStorage.getItem('allowed_filters') !== null) {
        state.currentFilters = JSON.parse(sessionStorage.getItem('current_filters'))
        state.allowedFilters = JSON.parse(sessionStorage.getItem('allowed_filters'))
    }


})
onMounted(async () => {
    const loadedFiltersName = ['city_id', 'customer_id', 'created_id', 'checklist_id', 'entity_id', 'instruction_id']
    state.city_id = await getCitiesWithAuditsOnly()
    state.customer_id = await getCustomers()
    state.created_id = await getCreators()
    state.checklist_id = await getChecklists()
    state.entity_id = await getEntities()
    state.instruction_id = await getInstructions()
    state.currentFilters.forEach(el => {
        if (loadedFiltersName.includes(el.fieldName)) {
            el.data = state[el.fieldName]//Запись в data filters
        }
    })
    const transferFilters = []
    state.allowedFilters.forEach((el) => {
        if (loadedFiltersName.includes(el.fieldName)) {
            el.data = state[el.fieldName]//Запись в data allowedFilters
        }
        if (props.selectedFilters[el.fieldName]?.length > 0) {
            //Перенос из  allowedFilter в filters
            transferFilters.push(el)
            state.currentFilters.push(el)
        }
    })
    state.allowedFilters = state.allowedFilters.filter(el => !transferFilters.includes(el))


})
</script>
<template>
    <div class="flex max-w-full ">
        <div class="max-w-full flex-wrap flex flex-row justify-start items-center gap-2">
            <Popover v-for="(filter, index) in state.currentFilters" :key="index"
                class="relative max-w-full flex-wrap flex flex-row justify-start items-center gap-2"
                v-slot="{close,open}">
                <PopoverButton @click="handlePopupFilter(filter.fieldName,close)"
                    class="bg-[#F1F6FF] rounded-md px-3 py-1 text-sm  ">
                    <div v-if="filter.type=='checkbox'||(!props.selectedFilters||!(props?.selectedFilters[filter?.fieldName]&&props?.selectedFilters[filter?.fieldName].length))" class="flex items-center gap-3 justify-center relative whitespace-nowrap" >
                    {{filter.title}} <svg class="mt-1 transition-transform" :class="{'rotate-180':open}" width="12"
                        height="12" viewBox="0 0 24 13" fill="none" xmlns="http://www.w3.org/2000/svg">
                        <path d="M22 1.99999L12 10.2609L2 1.99999" stroke="#A2A8B3" stroke-width="3"
                            stroke-linecap="round" />
                    </svg></div>
                    <div v-if="filter.type=='datePicker'&&props.selectedFilters[filter.fieldName]&&props.selectedFilters[filter.fieldName].length>0" class="flex items-center gap-3 justify-center relative whitespace-nowrap" >
                    {{showPeriodFiltersFill(filter,props.selectedFilters[filter.fieldName])}} <svg class="mt-1 transition-transform" :class="{'rotate-180':open}" width="12"
                        height="12" viewBox="0 0 24 13" fill="none" xmlns="http://www.w3.org/2000/svg">
                        <path d="M22 1.99999L12 10.2609L2 1.99999" stroke="#A2A8B3" stroke-width="3"
                            stroke-linecap="round" />
                    </svg></div>
                    <div v-if="filter.type=='checkbox'&&props.selectedFilters[filter.fieldName]&&props.selectedFilters[filter.fieldName].length > 0"
                        class="absolute top-[-5px] right-0 w-3 h-3 rounded-full bg-red-500 text-white text-xs flex justify-center items-center ">
                        {{props.selectedFilters[filter.fieldName].length}}</div>
                </PopoverButton>
                <PopoverPanel as="div"
                    class="absolute bg-white z-[100] left-0 top-[35px] rounded-md flex flex-col max-h-[31rem] h-[50vh] min-h-[1rem] min-w-[15rem] w-[20vw] max-w-[20rem] drop-shadow-lg ">
                    <div v-if="filter.type=='checkbox'">
                        <div class="flex flex-col items-center justify-center py-2 ">
                            <button class="flex items-center gap-1 text-base" @click="resetFilter(filter.fieldName)">
                                <svg width="20" height="20" viewBox="0 0 16 16" fill="none"
                                    xmlns="http://www.w3.org/2000/svg">
                                    <path d="M0.666626 2.66675V6.66675H4.66663" stroke="#1A1A1A" stroke-linecap="round"
                                        stroke-linejoin="round" />
                                    <path
                                        d="M2.33996 9.99989C2.77222 11.2268 3.59152 12.28 4.6744 13.0008C5.75728 13.7216 7.04508 14.0709 8.34377 13.9962C9.64246 13.9215 10.8817 13.4267 11.8747 12.5864C12.8677 11.7461 13.5608 10.6059 13.8494 9.33746C14.1381 8.06905 14.0067 6.74119 13.475 5.55395C12.9434 4.36671 12.0403 3.38441 10.9018 2.75506C9.76339 2.12571 8.45123 1.88339 7.16308 2.06463C5.87493 2.24586 4.68057 2.84083 3.75996 3.75988L0.666626 6.66655"
                                        stroke="#1A1A1A" stroke-linecap="round" stroke-linejoin="round" />
                                </svg>
                                Сбросить фильтры
                            </button>
                        </div>
                    </div>
                    <div v-if="filter.type =='datePicker'" class="flex p-2 gap-2 ">
                        <el-date-picker size="default" style="margin-bottom: 0.5rem; "
                            :start-placeholder="filter.subtitle[0]" :end-placeholder="filter.subtitle[1]"
                            value-format="DD.MM.YYYY" format="DD.MM.YYYY" :teleported="false" type="daterange"
                            v-model="state.bufferChosenFilters.selectedFilters[filter.fieldName]" />
                    </div>
                    <div class=" w-full px-2  opacity-100 z-[10]" v-if="filter.type=='checkbox'">
                        <el-input v-model="state.searchFilters" placeholder="Поиск" clearable />
                    </div>
                    <div class="h-full overflow-y-auto overflow-x-hidden  dialog pl-4 mr-2 mt-1">
                        <el-checkbox-group 
                            v-if="filter.type=='checkbox'"
                            v-model="state.bufferChosenFilters.selectedFilters[filter.fieldName]"
                            class="flex flex-wrap w-full " size="large">
                            <el-checkbox size="large" v-for="(item, i) in showedFilters(filter)" :key="i"
                                :label="`${item.id}`" class="w-1/2">
                                <p class="text-base "> {{
                                item.name||(item.last_name&&item.first_name?item.last_name+item.first_name:null)||item.title}}
                                </p>
                            </el-checkbox>
                        </el-checkbox-group>
                    </div>
                    <div class="w-full z-[100] px-2 pb-2 pt-1">
                        <button @click="applyChosenFilter(close,filter.fieldName)"
                            class="w-full bg-[#FF272B] rounded-md flex items-center justify-center">
                            <p class="text-white py-2 font-normal">Применить</p>
                        </button>
                    </div>
                </PopoverPanel>
            </Popover>
            <Popover class="relative " v-slot="{close}">
                <PopoverButton @click="addMoreFiters" class="bg-[#F1F6FF] rounded-md p-2  flex " ref="addfilters">
                    <svg width="14" height="14" viewBox="0 0 16 16" fill="none" xmlns="http://www.w3.org/2000/svg">
                        <path d="M8 3.33325V12.6666" stroke="#1A1A1A" stroke-linecap="round" stroke-linejoin="round" />
                        <path d="M3.33337 8H12.6667" stroke="#1A1A1A" stroke-linecap="round" stroke-linejoin="round" />
                    </svg>
                </PopoverButton>
                <PopoverPanel as="div"
                    class="absolute bg-white z-[100] right-0 top-[35px] rounded-md flex flex-col max-h-[31rem] h-[55vh] min-h-[1rem] w-[20rem] drop-shadow-lg overflow-hidden"
                    :class="{'left-0':isAddFiltersButtonInLeft(),'right-0':!(isAddFiltersButtonInLeft())}">
                    <div class="px-2 pt-2 pb-1 ">
                        <el-input v-model="state.searchFilters" placeholder="Поиск" clearable />
                    </div>
                    <div class="dialog overflow-auto h-full pl-4 mr-2 ">
                        <div @click="state.showCurrentFilters=!state.showCurrentFilters"
                            class="flex flex-row items-center justify-between w-full pr-2 cursor-pointer">
                            <p class="text-base font-normal text-[#6C6C6C]">Выбранные</p>
                            <svg class="mt-1 transition-transform" :class="{'rotate-180':state.showCurrentFilters}"
                                width="12" height="12" viewBox="0 0 24 13" fill="none"
                                xmlns="http://www.w3.org/2000/svg">
                                <path d="M22 1.99999L12 10.2609L2 1.99999" stroke="#A2A8B3" stroke-width="3"
                                    stroke-linecap="round" />
                            </svg>
                        </div>
                        <div class="flex flex-col items-start justify-start pb-4 border-b "
                            v-if="state.bufferChosenFilters.currentFilters.length > 0 && state.showCurrentFilters">
                            <el-checkbox-group class="flex flex-wrap w-full" size="large">
                                <el-checkbox size="large" v-for="(filter, i) in showedCurrentFilters" checked="true"
                                    :key="i" :label="`${filter.title}`" class="w-1/2" @change="handleAddFilter(filter)">
                                    <p class="text-base "> {{ filter.title }}</p>
                                </el-checkbox>

                            </el-checkbox-group>
                        </div>
                        <div @click="state.showAllowedFilters=!state.showAllowedFilters"
                            class="flex flex-row items-center justify-between w-full pr-2 cursor-pointer">
                            <p class="text-base font-normal text-[#6C6C6C]">Доступные</p>
                            <svg class="mt-1 transition-transform" :class="{'rotate-180':state.showAllowedFilters}"
                                width="12" height="12" viewBox="0 0 24 13" fill="none"
                                xmlns="http://www.w3.org/2000/svg">
                                <path d="M22 1.99999L12 10.2609L2 1.99999" stroke="#A2A8B3" stroke-width="3"
                                    stroke-linecap="round" />
                            </svg>
                        </div>
                        <div class="flex flex-col items-start justify-start pb-4 border-b "
                            v-if="state.bufferChosenFilters.allowedFilters?.length > 0 && state.showAllowedFilters">
                            <el-checkbox-group class="flex flex-wrap w-full" size="large">
                                <el-checkbox size="large" v-for="(filter, i) in showedAllowedFilters" :key="i"
                                    :label="`${filter.title}`" class="w-1/2" @change="handleAddFilter(filter)">
                                    <p class="text-base "> {{ filter.title }}</p>
                                </el-checkbox>

                            </el-checkbox-group>
                        </div>
                    </div>
                    <div class="w-full z-[100] px-2 pb-2 pt-1">
                        <button @click="applyChosenFilters(close)"
                            class="w-full bg-[#FF272B] rounded-md flex items-center justify-center">
                            <p class="text-white py-2 font-normal">Применить</p>
                        </button>
                    </div>
                </PopoverPanel>
            </Popover>

            <button class="flex items-center gap-1 text-sm pl-2" @click="resetAllFilters">
                Сбросить все <svg width="16" height="16" viewBox="0 0 16 16" fill="none"
                    xmlns="http://www.w3.org/2000/svg">
                    <path d="M0.666626 2.66675V6.66675H4.66663" stroke="#1A1A1A" stroke-linecap="round"
                        stroke-linejoin="round" />
                    <path
                        d="M2.33996 9.99989C2.77222 11.2268 3.59152 12.28 4.6744 13.0008C5.75728 13.7216 7.04508 14.0709 8.34377 13.9962C9.64246 13.9215 10.8817 13.4267 11.8747 12.5864C12.8677 11.7461 13.5608 10.6059 13.8494 9.33746C14.1381 8.06905 14.0067 6.74119 13.475 5.55395C12.9434 4.36671 12.0403 3.38441 10.9018 2.75506C9.76339 2.12571 8.45123 1.88339 7.16308 2.06463C5.87493 2.24586 4.68057 2.84083 3.75996 3.75988L0.666626 6.66655"
                        stroke="#1A1A1A" stroke-linecap="round" stroke-linejoin="round" />
                </svg>
            </button>
        </div>
    </div>
</template>
<style lang="scss" scoped>
.dialog::-webkit-scrollbar {
    width: 8px;
}

.dialog::-webkit-scrollbar-track {
    background-color: #F1F6FF;
    border-radius: 10px;
}

.dialog::-webkit-scrollbar-thumb {
    background: #D8D8D8;
    border-radius: 10px;
}
</style>