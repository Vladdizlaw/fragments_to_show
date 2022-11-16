<script setup>
import draggable from 'vuedraggable'
import { Popover, PopoverButton, PopoverPanel } from '@headlessui/vue'
import { nextTick, ref, reactive, computed, watch, defineEmits, defineProps, onBeforeMount, onBeforeUpdate, onBeforeUnmount, onMounted } from 'vue';
const props = defineProps({
    audits: {
        type: Array, default: []  //Список итемов
    },
    currentSettings: {
        type: Array, default: [   //Список текущих настроек таблицы 
            //  пример данных:
            // { title: 'ID',  field: 'id', sorted: false, width: '3rem' },
            // { title: 'Объекты', field: 'entity', sorted: false, width: '15rem' },
            // { title: 'Статус',  field: 'status', sorted: false, width: '10rem' },
            // { title: 'Тайник',  field: 'doer', sorted: false, width: '10rem' },
            // { title: 'Отклики',  field: 'bids_count', sorted: false, width: '5rem' },
            // { title: 'Создана',  field: 'created_at', sorted: false, width: '10rem' },
            // { title: 'Период',  field: 'start_date', sorted: false, width: '10rem' },
        ]
    },
    allowedSettings: {
        type: Array, default: [    //Список доступных настроек таблицы   
            //  пример данных:
            // { title: 'Тип проверки',  field: 'type', sorted: false, width: '10rem' },
            // { title: 'Период публикации проверки',field: 'published_at', sorted: false, width: '15rem' },
            // { title: 'Дата истечения проверки ', field: 'expiration_date', sorted: false, width: '15rem' },
            // { title: 'Вознаграждение', field: 'reward_amount', sorted: false, width: '10rem' },
            // { title: 'Возмещение',field: 'refund_amount', sorted: false, width: '10rem' },
            // { title: 'Средневзвешенный балл',  field: 'weighted_avg_score', sorted: false, width: '15rem' },
            // { title: 'Суммарный балл',  field: 'report_score', sorted: false, width: '10rem' },
        ]

    },
    routeTo: { //Объект куда роутить и какое поле отсылать при нажатие на итем 
        type: Object,
        default: {
            name: '',
            field: ''
        }
    },
    scrollup: {
        type: Boolean,

    }

})
const emit = defineEmits(['update:scrollUp'])
const itemslist = ref(null) //ref на шаблон в таблице для сейва скролла
const state = reactive({
    scroll: 0,
    showCurrentSettings: true,
    showAllowedSettings: true,
    searchFilter: '',
    currentSettings: [...props.currentSettings],
    allowedSettings: [...props.allowedSettings],
    bufferChosenSettings: { currentSettings: [...props.currentSettings], allowedSettings: [...props.allowedSettings] }
})
const typeTranslate = { complex: 'Комплекс', visit: 'Визит', call: 'Звонок' }
function applyChosenSettings(close) {
    state.currentSettings = [...state.bufferChosenSettings.currentSettings]
    state.bufferChosenSettings.currentSettings = [...state.currentSettings]
    state.allowedSettings = [...state.bufferChosenSettings.allowedSettings]
    state.bufferChosenSettings.allowedSettings = [...state.allowedSettings]
    close()
}

function sortAudits(item) {
    const index = state.currentSettings.findIndex(el => el.field == item.field)
    let field = item.field
    let field2 //Для доступа во вложенные свойства

    switch (item.field) {
        case 'entity': {
            field2 = ['name']
            break
        }
        case 'doer': {
            if (state.currentSettings[index]['field']) {
                field2 = ['rating']
            } else {
                field2 = undefined
            }
            break
        }
        case 'status': {
            field2 = ['title']
            break
        }
    }
    state.currentSettings[index].sorted = !state.currentSettings[index].sorted
    if (item.sorted) {
        props.audits = props.audits.sort((a, b) => {
            if (!b[field] && a[field])
                return -1

            if (a[field] && b[field] && field2 && a[field][field2] > b[field][field2])
                return 1
            if (a[field] && b[field] && field2 && a[field][field2] < b[field][field2])
                return -1
            if (!field2 && a[field] && b[field] && a[field] > b[field])
                return 1
            if (!field2 && a[field] && b[field] && a[field] < b[field])
                return -1

            return 0
        })
    }
    if (!item.sorted) {
        props.audits = props.audits.sort((a, b) => {
            if (!a[field] && b[field])
                return -1

            if (a[field] && b[field] && field2 && a[field][field2] < b[field][field2])
                return 1
            if (a[field] && b[field] && field2 && a[field][field2] > b[field][field2])
                return -1
            if (!field2 && a[field] && b[field] && a[field] < b[field])
                return 1
            if (!field2 && a[field] && b[field] && a[field] > b[field])
                return -1
            return 0
        })
    }
}

function addMoreSettings() {
    state.bufferChosenSettings.currentSettings = [...state.currentSettings]
    state.bufferChosenSettings.allowedSettings = [...state.allowedSettings]
    state.searchFilter = ''
    state.showAllowedSettings = true
    state.showCurrentSettings = true
}
function handleAddSetting(item) {
    if (state.bufferChosenSettings.currentSettings.includes(item)) {
        state.bufferChosenSettings.currentSettings = state.bufferChosenSettings.currentSettings.filter((el) => el.title !== item.title)
        state.bufferChosenSettings.allowedSettings.push(item)
    } else if (state.bufferChosenSettings.allowedSettings.includes(item)) {
        state.bufferChosenSettings.allowedSettings = state.bufferChosenSettings.allowedSettings.filter((el) => el.title !== item.title)
        state.bufferChosenSettings.currentSettings.push(item)
    }

}
function formatPhoneNumber(str) {
    let cleaned = ('' + str).replace(/\D/g, '')

    let match = cleaned.match(/^(7|8)?(\d{3})(\d{3})(\d{2})(\d{2})$/)

    if (match) {
        let intlCode = match[1] ? '+7' : ''
        return [
            intlCode,
            ' (',
            match[2],
            ') ',
            match[3],
            '-',
            match[4],
            '-',
            match[5]
        ].join('')
    }

    return null
}
function moveCurrentSettings(el) {
    //Что б работал драг на computed массиве
    const movedElement = { ...state.bufferChosenSettings.currentSettings[el.oldIndex] }
    state.bufferChosenSettings.currentSettings.splice(el.oldIndex, 1)
    state.bufferChosenSettings.currentSettings.splice(el.newIndex, 0, movedElement)
}
watch(async () => {
    //Скролл вверх при выборе страницы
    if (props.scrollup === true) {
        await nextTick()
        itemslist.value.scrollTo(0, 0)
        emit('update:scrollup', false)

    }
})
const showedCurrentSettings = computed(() => {
    return state.bufferChosenSettings.currentSettings.filter(el => {
        if (el.title) {
            return el['title'].toLowerCase().includes(state.searchFilter.toLowerCase())
        }
    })


})

const showedAllowedSettings = computed(() => {
    return state.bufferChosenSettings.allowedSettings.filter(el => {
        if (el.title) {
            return el['title'].toLowerCase().includes(state.searchFilter.toLowerCase())
        }
    })
})

onBeforeUpdate(() => {
    sessionStorage.setItem('current_settings', JSON.stringify(state.currentSettings))
    sessionStorage.setItem('allowed_settings', JSON.stringify(state.allowedSettings))
    state.scroll = itemslist.value.scrollTop
    sessionStorage.setItem('scroll_table', state.scroll)

})
onBeforeMount(() => {
    if (sessionStorage.getItem('current_settings') !== null && sessionStorage.getItem('allowed_settings') !== null) {
        state.currentSettings = JSON.parse(sessionStorage.getItem('current_settings'))
        state.allowedSettings = JSON.parse(sessionStorage.getItem('allowed_settings'))
    }
    if (sessionStorage.getItem('scroll_table') !== null) {
        state.scroll = sessionStorage.getItem('scroll_table')
    }
    onBeforeUnmount(() => {
        state.scroll = itemslist.value.scrollTop
        sessionStorage.setItem('scroll_table', state.scroll)
    })
    onMounted(async () => {
        await nextTick()
        itemslist.value.scrollTo(0, state.scroll)
    })
})

</script>
<template>
    <div
        class="dialog overflow-x-auto  h-full max-h-fit overflow-y-hidden box-border min-w-full max-w-fit flex  rounded-md bg-[#FFFFFF]">
        <div class="relative  min-w-full w-fit h-full  flex flex-col">
            <section
                class=" border-b rounded-md bg-white flex flex-row gap-4  items-center justify-between min-w-full p-4  pr-6 w-fit h-10 bg-[#F1F6FF]">
                <div v-for="(setting,index) in state.currentSettings" :key="index" :style="`width:${setting.width}`"
                    class=" text-base font-bold flex  items-center gap-2 shrink-0 cursor-pointer h-full  "
                    @click="sortAudits(setting)">
                    {{setting.title}}
                    <div>
                        <svg :class="{'rotate-180':setting.sorted}" width="8" height="4" viewBox="0 0 8 4" fill="none"
                            xmlns="http://www.w3.org/2000/svg">
                            <path d="M0 0L4 4L8 0" fill="black" />
                        </svg>
                    </div>
                </div>
                <div>
                    <Popover class="relative " v-slot="{close}">
                        <PopoverButton
                            class="text-base font-bold  flex justify-end items-center bg-[#F1F6FF] h-8 w-12  rounded-md  "
                            @click="addMoreSettings">
                            <svg class="transition-all active:rotate-180" width="20" height="20" viewBox="0 0 16 16"
                                fill="none" xmlns="http://www.w3.org/2000/svg">
                                <g clip-path="url(#clip0_628_1909)">
                                    <path
                                        d="M8 10C9.10457 10 10 9.10457 10 8C10 6.89543 9.10457 6 8 6C6.89543 6 6 6.89543 6 8C6 9.10457 6.89543 10 8 10Z"
                                        stroke="#1A1A1A" stroke-linecap="round" stroke-linejoin="round" />
                                    <path
                                        d="M12.9333 9.99996C12.8446 10.201 12.8181 10.4241 12.8573 10.6404C12.8965 10.8566 12.9996 11.0562 13.1533 11.2133L13.1933 11.2533C13.3173 11.3771 13.4156 11.5242 13.4827 11.686C13.5498 11.8479 13.5844 12.0214 13.5844 12.1966C13.5844 12.3718 13.5498 12.5453 13.4827 12.7072C13.4156 12.8691 13.3173 13.0161 13.1933 13.14C13.0695 13.2639 12.9225 13.3623 12.7606 13.4294C12.5987 13.4965 12.4252 13.531 12.25 13.531C12.0748 13.531 11.9013 13.4965 11.7394 13.4294C11.5775 13.3623 11.4305 13.2639 11.3067 13.14L11.2667 13.1C11.1095 12.9463 10.91 12.8432 10.6937 12.804C10.4775 12.7647 10.2544 12.7912 10.0533 12.88C9.85615 12.9645 9.68799 13.1048 9.56954 13.2836C9.45109 13.4625 9.38752 13.6721 9.38667 13.8866V14C9.38667 14.3536 9.24619 14.6927 8.99614 14.9428C8.74609 15.1928 8.40696 15.3333 8.05333 15.3333C7.69971 15.3333 7.36057 15.1928 7.11053 14.9428C6.86048 14.6927 6.72 14.3536 6.72 14V13.94C6.71484 13.7193 6.64341 13.5053 6.51501 13.3258C6.3866 13.1462 6.20716 13.0095 6 12.9333C5.79892 12.8445 5.57587 12.8181 5.35961 12.8573C5.14335 12.8965 4.94379 12.9996 4.78667 13.1533L4.74667 13.1933C4.62284 13.3173 4.47579 13.4156 4.31392 13.4827C4.15206 13.5498 3.97856 13.5843 3.80333 13.5843C3.62811 13.5843 3.45461 13.5498 3.29275 13.4827C3.13088 13.4156 2.98383 13.3173 2.86 13.1933C2.73603 13.0695 2.63769 12.9224 2.57059 12.7605C2.50349 12.5987 2.46895 12.4252 2.46895 12.25C2.46895 12.0747 2.50349 11.9012 2.57059 11.7394C2.63769 11.5775 2.73603 11.4305 2.86 11.3066L2.9 11.2666C3.05369 11.1095 3.15679 10.9099 3.196 10.6937C3.23522 10.4774 3.20874 10.2544 3.12 10.0533C3.03549 9.85611 2.89517 9.68795 2.71631 9.5695C2.53745 9.45105 2.32786 9.38748 2.11333 9.38663H2C1.64638 9.38663 1.30724 9.24615 1.05719 8.9961C0.807144 8.74605 0.666668 8.40691 0.666668 8.05329C0.666668 7.69967 0.807144 7.36053 1.05719 7.11048C1.30724 6.86043 1.64638 6.71996 2 6.71996H2.06C2.28066 6.7148 2.49467 6.64337 2.6742 6.51497C2.85373 6.38656 2.99048 6.20712 3.06667 5.99996C3.15541 5.79888 3.18188 5.57583 3.14267 5.35957C3.10346 5.1433 3.00036 4.94375 2.84667 4.78663L2.80667 4.74663C2.6827 4.62279 2.58435 4.47574 2.51725 4.31388C2.45016 4.15202 2.41562 3.97851 2.41562 3.80329C2.41562 3.62807 2.45016 3.45457 2.51725 3.29271C2.58435 3.13084 2.6827 2.98379 2.80667 2.85996C2.9305 2.73599 3.07755 2.63765 3.23941 2.57055C3.40128 2.50345 3.57478 2.46891 3.75 2.46891C3.92522 2.46891 4.09872 2.50345 4.26059 2.57055C4.42245 2.63765 4.5695 2.73599 4.69333 2.85996L4.73333 2.89996C4.89045 3.05365 5.09001 3.15675 5.30628 3.19596C5.52254 3.23517 5.74559 3.2087 5.94667 3.11996H6C6.19718 3.03545 6.36535 2.89513 6.4838 2.71627C6.60225 2.53741 6.66581 2.32782 6.66667 2.11329V1.99996C6.66667 1.64634 6.80714 1.3072 7.05719 1.05715C7.30724 0.807102 7.64638 0.666626 8 0.666626C8.35362 0.666626 8.69276 0.807102 8.94281 1.05715C9.19286 1.3072 9.33333 1.64634 9.33333 1.99996V2.05996C9.33419 2.27448 9.39776 2.48408 9.51621 2.66294C9.63466 2.8418 9.80282 2.98212 10 3.06663C10.2011 3.15537 10.4241 3.18184 10.6404 3.14263C10.8567 3.10342 11.0562 3.00032 11.2133 2.84663L11.2533 2.80663C11.3772 2.68266 11.5242 2.58431 11.6861 2.51721C11.8479 2.45011 12.0214 2.41558 12.1967 2.41558C12.3719 2.41558 12.5454 2.45011 12.7073 2.51721C12.8691 2.58431 13.0162 2.68266 13.14 2.80663C13.264 2.93046 13.3623 3.07751 13.4294 3.23937C13.4965 3.40124 13.531 3.57474 13.531 3.74996C13.531 3.92518 13.4965 4.09868 13.4294 4.26055C13.3623 4.42241 13.264 4.56946 13.14 4.69329L13.1 4.73329C12.9463 4.89041 12.8432 5.08997 12.804 5.30623C12.7648 5.5225 12.7913 5.74555 12.88 5.94663V5.99996C12.9645 6.19714 13.1048 6.3653 13.2837 6.48375C13.4626 6.6022 13.6721 6.66577 13.8867 6.66663H14C14.3536 6.66663 14.6928 6.8071 14.9428 7.05715C15.1929 7.3072 15.3333 7.64634 15.3333 7.99996C15.3333 8.35358 15.1929 8.69272 14.9428 8.94277C14.6928 9.19282 14.3536 9.33329 14 9.33329H13.94C13.7255 9.33415 13.5159 9.39771 13.337 9.51616C13.1582 9.63461 13.0178 9.80278 12.9333 9.99996V9.99996Z"
                                        stroke="black" stroke-linecap="round" stroke-linejoin="round" />
                                </g>
                                <defs>
                                    <clipPath id="clip0_628_1909">
                                        <rect width="20" height="20" fill="white" />
                                    </clipPath>
                                </defs>
                            </svg>
                        </PopoverButton>
                        <PopoverPanel as="div"
                            class="pop_settings absolute bg-white z-[100] flex flex-col right-[-10px] top-[37px] rounded-md  max-h-[50vh] min-h-[1rem] w-[20rem] drop-shadow-lg">
                            <div class="px-2 pt-2 pb-1">
                                <el-input v-model="state.searchFilter" placeholder="Поиск" />
                            </div>
                            <div
                                class="relative w-full dialog  flex flex-col w-fit rounded-md overflow-y-auto pl-4 mr-2 h-[25rem]  w-[19.5rem]">

                                <div class="flex flex-col items-start justify-start  border-b  "
                                    v-if="state.bufferChosenSettings.currentSettings.length > 0">

                                    <div @click="state.showCurrentSettings=!state.showCurrentSettings"
                                        class="flex flex-row items-center justify-between w-full pr-2 cursor-pointer">
                                        <p class="text-base font-normal text-[#6C6C6C]">Выбранные</p>
                                        <svg class="mt-1 transition-transform"
                                            :class="{'rotate-180':state.showCurrentSettings}" width="12" height="12"
                                            viewBox="0 0 24 13" fill="none" xmlns="http://www.w3.org/2000/svg">
                                            <path d="M22 1.99999L12 10.2609L2 1.99999" stroke="#A2A8B3" stroke-width="3"
                                                stroke-linecap="round" />
                                        </svg>
                                    </div>
                                    <draggable :list="showedCurrentSettings" @end="moveCurrentSettings"
                                        :swapThreshold="0.9" animation="150" class="w-full pr-1">
                                        <template #item="{element}">
                                            <div class="relative w-full">

                                                <div class="flex flex-wrap flex-row w-full justify-between items-center"
                                                    v-if="state.showCurrentSettings">
                                                    <el-checkbox size="large" :key="element.field" checked="true"
                                                        :label="`${element.title}`" class="w-[calc(100% - 1em)]"
                                                        @change="handleAddSetting(element)">
                                                        <p class="text-base "> {{ element.title }} </p>
                                                    </el-checkbox>
                                                    <svg width="24" height="24" viewBox="0 0 24 24" fill="none"
                                                        xmlns="http://www.w3.org/2000/svg" class="w-[1em] pr-1 cursor-grab active:cursor-grabbing" >
                                                        <path
                                                            d="M17 13C17.5523 13 18 12.5523 18 12C18 11.4477 17.5523 11 17 11C16.4477 11 16 11.4477 16 12C16 12.5523 16.4477 13 17 13Z"
                                                            stroke="#D8D8D8" stroke-width="2" stroke-linecap="round"
                                                            stroke-linejoin="round" />
                                                        <path
                                                            d="M7 13C7.55228 13 8 12.5523 8 12C8 11.4477 7.55228 11 7 11C6.44772 11 6 11.4477 6 12C6 12.5523 6.44772 13 7 13Z"
                                                            stroke="#D8D8D8" stroke-width="2" stroke-linecap="round"
                                                            stroke-linejoin="round" />
                                                        <path
                                                            d="M17 6C17.5523 6 18 5.55228 18 5C18 4.44772 17.5523 4 17 4C16.4477 4 16 4.44772 16 5C16 5.55228 16.4477 6 17 6Z"
                                                            stroke="#D8D8D8" stroke-width="2" stroke-linecap="round"
                                                            stroke-linejoin="round" />
                                                        <path
                                                            d="M7 6C7.55228 6 8 5.55228 8 5C8 4.44772 7.55228 4 7 4C6.44772 4 6 4.44772 6 5C6 5.55228 6.44772 6 7 6Z"
                                                            stroke="#D8D8D8" stroke-width="2" stroke-linecap="round"
                                                            stroke-linejoin="round" />
                                                        <path
                                                            d="M17 20C17.5523 20 18 19.5523 18 19C18 18.4477 17.5523 18 17 18C16.4477 18 16 18.4477 16 19C16 19.5523 16.4477 20 17 20Z"
                                                            stroke="#D8D8D8" stroke-width="2" stroke-linecap="round"
                                                            stroke-linejoin="round" />
                                                        <path
                                                            d="M7 20C7.55228 20 8 19.5523 8 19C8 18.4477 7.55228 18 7 18C6.44772 18 6 18.4477 6 19C6 19.5523 6.44772 20 7 20Z"
                                                            stroke="#D8D8D8" stroke-width="2" stroke-linecap="round"
                                                            stroke-linejoin="round" />
                                                    </svg>

                                                </div>
                                            </div>
                                        </template>
                                    </draggable>
                                </div>
                                <div class="flex flex-col items-start justify-start"
                                    v-if="state.bufferChosenSettings.allowedSettings.length > 0">
                                    <div @click="state.showAllowedSettings=!state.showAllowedSettings"
                                        class="flex flex-row items-center justify-between w-full pr-2 cursor-pointer">
                                        <p class="pt-2 text-base font-normal text-[#6C6C6C]">Доступные</p>
                                        <svg class="mt-1 transition-transform"
                                            :class="{'rotate-180':state.showAllowedSettings}" width="12" height="12"
                                            viewBox="0 0 24 13" fill="none" xmlns="http://www.w3.org/2000/svg">
                                            <path d="M22 1.99999L12 10.2609L2 1.99999" stroke="#A2A8B3" stroke-width="3"
                                                stroke-linecap="round" />
                                        </svg>
                                    </div>
                                    <el-checkbox-group class="flex flex-wrap w-full" size="large"
                                        v-if="state.showAllowedSettings">
                                        <el-checkbox size="large" v-for="(item, i) in showedAllowedSettings" :key="i"
                                            :label="`${item.title}`" class="w-1/2" @change="handleAddSetting(item)">
                                            <p class="text-base "> {{ item.title }}</p>
                                        </el-checkbox>

                                    </el-checkbox-group>

                                </div>

                            </div>
                            <div class="w-full z-[100] px-2 pb-2 pt-1">
                                <button @click="applyChosenSettings(close)"
                                    class="w-full bg-[#FF272B] rounded-md flex items-center justify-center">
                                    <p class="text-white py-2 font-normal">Применить</p>
                                </button>
                            </div>
                        </PopoverPanel>
                    </Popover>
                </div>
            </section>
            <section ref="itemslist"
                class="dialog flex flex-col overflow-y-auto min-w-full w-fit  gap-1 h-full max-h-fit rounded-b-md  text-sm">
                <router-link
                    :to="{name: props.routeTo.name, params: {[props.routeTo.field]: audit[props.routeTo.field]}}"
                    class=" w-full cursor-pointer flex flex-row gap-4 px-4 justify-between hover:bg-[#E3EEFF]"
                    v-for="(audit,index) in props.audits" :key="index">
                    <div class="flex flex-row   items-start justify-start py-2 gap-4 "
                        v-for="(col,ind) in state.currentSettings" :key="ind">
                        <div v-if="col.field=='id'" :style="`width:${col.width};`"
                            class="flex flex-col  gap-1 shrink-0">
                            {{audit.id}}
                        </div>
                        <div v-if="col.field=='entity'" :style="`width:${col.width};`"
                            class="gap-1 flex flex-col shrink-0 items-between">
                            <p class="font-bold">{{audit.entity.name}}<br /></p>
                            <span v-if="audit.entity.address">{{audit.entity.address}} </span><span v-else>Адресс не
                                указан</span>
                            <div v-if="!!audit.comment"
                                class="mt-1 p-2 mb-1 flex  w-fit items-start rounded-lg bg-yellow-100">
                                <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor"
                                    class="flex-shrink-0 h-5 w-5 text-yellow-500">
                                    <path fill-rule="evenodd"
                                        d="M18 10a8 8 0 11-16 0 8 8 0 0116 0zm-7-4a1 1 0 11-2 0 1 1 0 012 0zM9 9a1 1 0 000 2v3a1 1 0 001 1h1a1 1 0 100-2v-3a1 1 0 00-1-1H9z"
                                        clip-rule="evenodd"></path>
                                </svg>
                                <div class="ml-2">
                                    <p class="text-xs font-medium capitalize-first text-gray-900">
                                        {{ audit.comment }}
                                    </p>
                                </div>
                            </div>
                        </div>

                        <div v-if="col.field=='status'" class="flex flex-col  gap-1 shrink-0 mt-1"
                            :style="`width:${col.width};`">
                            <div v-if="audit.status.name === 'audit-created'"
                                class=" rounded-full px-2 bg-[#71A8FF] text-white h-[1.4rem] flex items-center justify-center">
                                <div class="text-sm whitespace-nowrap lowercase">
                                    {{ audit.status.title }}
                                </div>
                            </div>

                            <div v-if="audit.status.name === 'audit-finished'">
                                <div class="">
                                    <icon-cursor class="status-message__icon" />

                                    <div class="status-message__title ">
                                        {{ audit.status.title }}
                                    </div>
                                </div>
                            </div>

                            <div v-if="audit.status.name === 'audit-started'"
                                class="inline-block rounded-full px-1.5 bg-[#FFBF41] text-white h-[1.4rem] justify-center flex items-center">
                                <div class="text-sm whitespace-nowrap lowercase">
                                    {{ audit.status.title }}
                                </div>
                            </div>

                            <div v-if="audit.status.name === 'audit-completed'"
                                class="inline-block rounded-full px-1.5 bg-[#44C863] text-white h-[1.4rem] flex items-center justify-center">
                                <div class="text-sm whitespace-nowrap lowercase">
                                    {{ audit.status.title }}
                                </div>
                            </div>

                            <div v-if="audit.status.name === 'pending'"
                                class="inline-block rounded-full px-1.5 bg-pink-500 text-white h-[1.4rem] flex items-center justify-center">
                                <div class="text-sm whitespace-nowrap lowercase">
                                    на модерации
                                </div>
                            </div>


                            <div v-if="audit.status.name === 'audit-closed'"
                                class="inline-block rounded-full px-1.5 h-[1.4rem] flex items-center bg-gray-400 text-white justify-center">
                                <div class="text-sm whitespace-nowrap lowercase">Закрыта</div>
                            </div>

                            <div v-if="audit.status.name === 'return'"
                                class="inline-block rounded-full px-1.5 bg-[#FF272B] text-white h-[1.4rem] flex items-center justify-center">
                                <div class="text-sm whitespace-nowrap lowercase">
                                    На доработке
                                </div>
                            </div>

                            <div v-if="audit.status.name === 'no-status'"
                                class="status-message inline-block rounded-full py-0.5 h-[1.4rem] flex items-center justify-center">
                                <div class="flex items-center">
                                    <icon-cancel class="mr-1" />
                                    <div class="text-sm whitespace-nowrap">
                                        {{ audit.status.title }}
                                    </div>
                                </div>

                            </div>
                            <span v-if="audit.doer" class="text-sm flex w-fit">{{
                            $t(`moderator.secretStatus[${audit.assigned.status_id}]`) }}</span>
                            <span v-else>Тайник не назначен</span>
                        </div>

                        <div v-if="col.field=='doer'" class="flex flex-col gap-1 shrink-0"
                            :style="`width:${col.width};`">

                            <div v-if="!!audit.doer" class="flex flex-col items-start justify-center">
                                <div>
                                    {{ audit.doer.last_name }} {{ audit.doer.first_name }}
                                </div>
                                <div v-if="audit.doer.rating" class="ml-[-2px] flex gap-2 items-center">
                                    <svg width="16" height="16" viewBox="0 0 14 13" fill="none"
                                        xmlns="http://www.w3.org/2000/svg">
                                        <path
                                            d="M7 0L8.5716 4.83688H13.6574L9.5429 7.82624L11.1145 12.6631L7 9.67376L2.8855 12.6631L4.4571 7.82624L0.342604 4.83688H5.4284L7 0Z"
                                            fill="#FFC90B" />
                                    </svg>
                                    <span :class="{'text-[#FF272B]':audit.doer.rating<3}">{{audit.doer.rating}}</span>
                                </div>
                                <div v-if="0">{{ formatPhoneNumber(audit.doer.phone) }}</div>
                            </div>
                            <div v-else>Не назначен</div>
                        </div>

                        <div v-if="col.field=='bids_count'" class="flex flex-col  gap-1 shrink-0"
                            :style="`width:${col.width};`">
                            <span
                                class="flex items-center justify-center px-2 py-1  text-xs font-semibold leading-none text-white bg-[#B4B4B4] rounded-full w-5"
                                :class="{'bg-[#7CD992]': audit.bids_count>0}">{{
                                audit.bids_count }}</span>
                        </div>

                        <div v-if="col.field=='start_date'" class="flex flex-col gap-1 shrink-0"
                            :style="`width:${col.width};`">
                            <div>{{ moment(audit.start_date).format('DD.MM.YYYY') }} -</div>

                            <div>{{ moment(audit.end_date).format('DD.MM.YYYY') }}</div>
                        </div>

                        <div v-if="col.field=='created_at'" class="flex flex-col  gap-1 shrink-0 text-sm"
                            :style="`width:${col.width};`">
                            <span>{{ moment(audit.created_at).format('DD.MM.YYYY, HH:mm') }}</span>
                            <div v-if="audit.creator">
                                {{ audit.creator.last_name }}
                                {{ audit.creator.first_name[0] }}.
                                {{ audit.creator.middle_name[0] }} ({{ audit.creator.id }})
                            </div>
                        </div>

                        <div v-if="col.field=='evaluated_at'" class="flex flex-col  gap-1 shrink-0"
                            :style="`width:${col.width};`">
                            <div v-if="audit.evaluated_at">{{ moment(audit.evaluated_at).format('DD.MM.YYYY') }} </div>
                            <div v-else>Нет данных</div>
                        </div>

                        <div v-if="col.field=='type'" class="flex flex-col gap-1 shrink-0"
                            :style="`width:${col.width};`">
                            {{typeTranslate[audit.type]}}
                        </div>

                        <div v-if="col.field=='published_at'" class="flex flex-col  gap-1 shrink-0"
                            :style="`width:${col.width};`">
                            <div v-if="audit.published_at">{{ moment(audit.published_at).format('DD.MM.YYYY') }} </div>
                            <div v-else>Нет данных</div>
                        </div>

                        <div v-if="col.field=='expiration_date'" class="flex flex-col  gap-1 shrink-0"
                            :style="`width:${col.width};`">
                            <div v-if="audit.expiration_date">{{ moment(audit.expiration_date).format('DD.MM.YYYY') }}
                            </div>
                            <div v-else>Нет данных</div>
                        </div>

                        <div v-if="col.field=='reward_amount'" class="flex flex-col  gap-1 shrink-0"
                            :style="`width:${col.width};`">
                            <div v-if="audit.reward_amount">{{ audit.reward_amount }} </div>
                            <div v-else>Нет данных</div>
                        </div>

                        <div v-if="col.field=='refund_amount'" class="flex flex-col  gap-1 shrink-0"
                            :style="`width:${col.width};`">
                            <div v-if="audit.refund_amount">{{ audit.refund_amount }} </div>
                            <div v-else>Нет данных</div>
                        </div>

                        <div v-if="col.field=='weighted_avg_score'" class="flex flex-col  gap-1 shrink-0"
                            :style="`width:${col.width};`">
                            <div v-if="audit.weighted_avg_score">{{ audit.weighted_avg_score }}% </div>
                            <div v-else>Нет данных</div>
                        </div>

                        <div v-if="col.field=='report_score'" class="flex flex-col  gap-1 shrink-0"
                            :style="`width:${col.width};`">
                            <div v-if="audit.report_score">{{ audit.report_score }} </div>
                            <div v-else>Нет данных</div>
                        </div>
                    </div>
                    <div class="w-11 mr-[1.5px]"></div>
                </router-link>
            </section>

        </div>
    </div>
</template>
<style lang="scss" scoped>
.dialog::-webkit-scrollbar {
    width: 10px;
    height: 10px;
    opacity: 0.5;

}

.dialog::-webkit-scrollbar-track {
    background-color: #F1F6FF;
    border-radius: 10px;
}

.dialog::-webkit-scrollbar-thumb {
    background: #D8D8D8;
    opacity: 0.6;
    border-radius: 10px;

}

.pop_settings {
    box-shadow: 0px 0px 10px rgba(26, 26, 26, 0.1);
}
</style>

