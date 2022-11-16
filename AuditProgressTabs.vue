
<script setup>
import {watch,defineProps,defineEmits,reactive, onBeforeUpdate} from 'vue'    
const props = defineProps({
    statuses: {
        type: Object,
        default: {}
    },
    filter: { type: Array, default: [] }
})
const emit = defineEmits(['update:filter'])
const state= reactive({
    filter:props.filter
})
watch(state, (value) => {
    emit('update:filter', value.filter)
   
})
onBeforeUpdate(()=>{
    state.filter=props.filter
})
</script>

<template>
  
<div class="w-full ">
    <div  class="flex flex-row bg-white rounded-lg px-[2.5px]  text-sm justify-between w-full items-center flex-wrap ">
        <div v-for="(status,index) in props.statuses.data" :key="index" class="flex items-center justify-center relative  whitespace-nowrap h-[2.03rem] min-w-[6.2rem]  w-[calc(100%/7.165)] audit-wrap">
        <input  type="checkbox" v-model="state.filter" :id="status.id" :value="status.id" class="opacity-0 z-[100] w-full cursor-pointer" />
        <label :for="status.id" class="absolute left-0 top-0 p-1 mt-[2.5px] rounded-md w-full flex items-center justify-center select-none audit-button" :class="{'bg-[#E3EEFF]':state.filter.includes(String(status.id))||state.filter.includes(Number(status.id))}">{{status.name}}</label>
    </div>
    </div>
</div>
</template>
<style lang="scss" scoped>
    .audit-wrap{
        &:hover{
            .audit-button{
             background-color: #E3EEFF;   
            }
        }
    }
</style>