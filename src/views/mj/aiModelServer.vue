<script setup lang="ts">
//NInfiniteScroll
import {NSelect, NInput,NSlider, NButton, useMessage,NTag,NEmpty,NModal,NDivider
} from "naive-ui"
//import type { SelectRenderLabel, SelectRenderTag } from 'naive-ui'
import { ref ,computed,watch, onMounted,h} from "vue";
import { SvgIcon } from '@/components/common'
import { gptFetch, mlog } from "@/api";

const st= ref({ server:'',isShow:false,isLoadData:0 ,"search":''});
const ms= useMessage();
  
const emit= defineEmits(['success']);
interface modelType{
    model:string
    //maxToken:Number
}
interface modelGroup{
    name:string
    key:string[]
    data:modelType[]
    class?:string[]
    icon?:string
    isClosed?:boolean
}

const mGroup= ref<modelGroup[]>([])

const isChatModel = (id: string) =>
    !/(?:image|video|embedding|tts|whisper|ocr|moderation|suno|music|flux|banana|veo|sora)/i.test(id);
const tierLabel = (id: string) =>
    id.startsWith('basic/') ? ' · 经济' :
    id.startsWith('standard/') ? ' · 标准' : ' · 核对价格';

const loadModel=async ()=>{
    st.value.isLoadData = 0;
    mGroup.value.forEach(group => group.data = []);
    try {
        const modelsData = await gptFetch('/v1/models');
        if (!Array.isArray(modelsData?.data)) throw new Error('Model catalog unavailable');
        for (const item of modelsData.data) {
            const id = typeof item?.id === 'string' ? item.id : '';
            if (!id || !isChatModel(id)) continue;
            const baseId = id.replace(/^(basic|standard)\//, '');
            const match = mGroup.value.find(group =>
                group.key.length && group.key.some(key => baseId.toLowerCase().startsWith(key))
            );
            (match ?? mGroup.value[mGroup.value.length - 1]).data.push({ model: id });
        }
        mGroup.value.forEach(group => group.data.sort((a, b) => a.model.localeCompare(b.model)));
        st.value.isLoadData = 1;
    } catch (error) {
        st.value.isLoadData = -1;
        ms.error('模型加载失败，请核对接口地址及 API Key');
    }
};
const openModelList=()=>{
    st.value.isShow = true;
    if (st.value.isLoadData !== 1) loadModel();
};

const initGroup=()=>{
    mGroup.value = [
        { name:'GPT', key:['gpt-', 'codex/'], data:[], icon:'ri:openai-fill' },
        { name:'Claude', key:['claude'], data:[], icon:'ri:claude-fill' },
        { name:'Gemini', key:['gemini'], data:[], icon:'cbi:gemini' },
        { name:'Grok', key:['grok'], data:[], icon:'token:xai' },
        { name:'DeepSeek', key:['deepseek'], data:[], icon:'arcticons:deepseek' },
        { name:'Qwen', key:['qwen'], data:[] },
        { name:'Kimi', key:['kimi'], data:[] },
        { name:'GLM', key:['glm'], data:[] },
        { name:'其他', key:[], data:[], isClosed:true }
    ];
};
onMounted(()=>{
 initGroup();
})

const successClick=(md:any)=> {
    emit('success', md)
    st.value.isShow=false
}

//const usageData = await gptFetch(urlUsage);
const modellist = computed(() => {
    let rz: any[]=[];
    for(let o of mGroup.value){
        if(o.data.length<=0) continue
        for(let v of o.data){
         rz.push({label:v.model,value:v.model})
        }
    }
    return rz;
});
const abc=()=>{
    //console.log('abc>> ',st.value.search)
    if(st.value.search=='') return;
    for(let o of mGroup.value){
        if(o.data.length<=0) continue
        for(let v of o.data){
         //rz.push({label:v.model,value:v.model})
         if(v.model==st.value.search) successClick( v)
        }
    }
}
</script>
<template>
<div @click="openModelList">
<NTag  type="primary" round size="small" :bordered="false" class="!cursor-pointer">
            {{ $t('mj.server_load') }}  </NTag>
 
</div>
<NModal  v-model:show="st.isShow"  preset="card"  :title="$t('mj.model_select')" class="!max-w-[620px]" @close="st.isShow=false" >
     <NEmpty v-if="st.isLoadData==0" description="正在读取模型列表..." />
     <div v-else-if="st.isLoadData==-1" class="p-4 text-center"><NEmpty description="加载失败，请检查 API Key 和接口地址" /><NButton class="mt-3" size="small" @click="loadModel">重试加载</NButton></div>
     <div   class=" overflow-y-auto max-h-[400px]" v-else >
        <div>
            
            <n-select v-model:value="st.search" @update:value="abc" clearable :options="modellist" size="small" placeholder="search and select your model"  filterable   />
       
        </div>
        <div v-for="mg in mGroup">
            <template v-if="mg.data.length>0">
            <div class="  relative"  >
                <n-divider title-placement="left">
                    <div class="flex justify-start items-center space-x-2" @click="mg.isClosed=!mg.isClosed">
                        <SvgIcon :icon="mg.icon" v-if="mg.icon"/>
                        <span>{{ mg.name }} </span>
                    </div>
                </n-divider>
                <div class="absolute right-3 top-[5px] rounded-full bg-white/10 cursor-pointer " @click="mg.isClosed=!mg.isClosed" :class="{'rotate-180':mg.isClosed}">
                   <SvgIcon icon="ri:arrow-up-s-line" />
                </div>
                <div class="absolute right-10 top-[2px]"  @click="mg.isClosed=!mg.isClosed">
                    <NTag  size="small" round   :bordered="false">
                        <span class=" cursor-pointer">{{ mg.data.length }}</span>
                    </NTag>
                </div>
            </div>
            <div v-if="mg.data.length>0 && !mg.isClosed" class="grid   gap-2 grid-cols-2">
                <div v-for="md in mg.data" >
                    <!-- <template v-if="st.search=='' || md.model.includes(st.search)"></template> -->
                   <NTag type="info" size="small" round>
                     <span class="cursor-pointer" @click="successClick(md)" >{{ md.model }}<span class="opacity-60">{{ tierLabel(md.model) }}</span></span>
                   </NTag>
                   
                </div>
            </div>
            </template>
        </div>
     </div>
 </NModal>
</template>