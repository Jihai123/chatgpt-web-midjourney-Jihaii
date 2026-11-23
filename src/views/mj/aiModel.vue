<script setup lang="ts">
import {NSelect, NInput,NSlider, NButton, useMessage,NTag,NPopselect ,NAvatar, NSwitch} from "naive-ui"
import type { SelectRenderLabel, SelectRenderTag } from 'naive-ui'
import { ref ,computed,watch, onMounted,h} from "vue";
import {gptConfigStore, homeStore,useChatStore} from '@/store'
import { mlog,chatSetting } from "@/api";
import { t } from "@/locales";

import AiModelServer from "./aiModelServer.vue";
 

const emit = defineEmits(['close']);
const chatStore = useChatStore();
const uuid = chatStore.active;
//mlog('uuid', uuid );
const chatSet = new chatSetting( uuid==null?1002:uuid);

const nGptStore = ref(  chatSet.getGptConfig() );

// 'claude-3-7-sonnet-20250219'
// ,'claude-3-5-sonnet-20241022','claude-3-sonnet-20240229','claude-3-opus-20240229','claude-3-haiku-20240307','claude-3-5-sonnet-20240620','suno-v3'

const curatedModelGroups = [
  {
    "type": "group",
    "key": "curated-0",
    "label": "热门主流 · Standard 优先",
    "children": [
      {
        "label": "GPT 6.2 Sol｜标准·综合编程",
        "value": "standard/gpt-6.2-sol"
      },
      {
        "label": "Claude Sonnet 5.5｜标准·写作编程",
        "value": "standard/claude-sonnet-5-5"
      },
      {
        "label": "Gemini 4 Argon｜标准·综合",
        "value": "standard/gemini-4-argon"
      },
      {
        "label": "Grok 4.7｜标准·主流新款",
        "value": "standard/grok-4.7"
      },
      {
        "label": "DeepSeek V4 Pro｜中文推理",
        "value": "deepseek-v4-pro"
      },
      {
        "label": "Gemini 3.8 Flash｜标准·轻量",
        "value": "standard/gemini-3.8-flash"
      },
      {
        "label": "Qwen Max｜中文应用",
        "value": "qwen-max"
      }
    ]
  },
  {
    "type": "group",
    "key": "curated-1",
    "label": "经济选择 · Basic / 低费用",
    "children": [
      {
        "label": "GPT 6.2 Sol｜Basic 经济版",
        "value": "basic/gpt-6.2-sol"
      },
      {
        "label": "Claude Sonnet 5.5｜Basic 经济版",
        "value": "basic/claude-sonnet-5-5"
      },
      {
        "label": "Gemini 3.8 Flash｜Basic 经济版",
        "value": "basic/gemini-3.8-flash"
      },
      {
        "label": "DeepSeek V4.1 Flash｜经济中文",
        "value": "deepseek-v4.1-flash-1017"
      },
      {
        "label": "GPT-4o mini｜轻量低费用",
        "value": "gpt-4o-mini"
      },
      {
        "label": "GPT 6.1 Sol｜Basic 经济备用",
        "value": "basic/gpt-6.1-sol"
      }
    ]
  },
  {
    "type": "group",
    "key": "curated-2",
    "label": "高阶旗舰 · 费用较高，按需选择",
    "children": [
      {
        "label": "GPT 6.2 Astra｜Standard 高阶",
        "value": "standard/gpt-6.2-astra"
      },
      {
        "label": "Claude Opus 5.5｜Standard 高阶",
        "value": "standard/claude-opus-5-5"
      },
      {
        "label": "Gemini 4 Carbon｜Standard 高阶",
        "value": "standard/gemini-4-carbon"
      },
      {
        "label": "Claude Fable 5.2｜Standard 高阶",
        "value": "standard/claude-fable-5-2"
      },
      {
        "label": "GLM 5.3｜中文进阶",
        "value": "glm-5.3"
      },
      {
        "label": "Kimi K3.1 Pro｜长文进阶",
        "value": "kimi-k3.1-pro"
      }
    ]
  },
  {
    "type": "group",
    "key": "curated-3",
    "label": "兼容备用",
    "children": [
      {
        "label": "GPT 6.1 Sol｜Standard",
        "value": "standard/gpt-6.1-sol"
      },
      {
        "label": "Gemini 4 Hydrogen｜Standard",
        "value": "standard/gemini-4-hydrogen"
      }
    ]
  }
];
const config = ref({maxToken:16384});
const st= ref({openMore:false,isShow:false ,server:'' });
const voiceList= computed(()=>{
    let rz=[];
    for(let o of "alloy,echo,fable,onyx,nova,shimmer".split(/[ ,]+/ig))rz.push({label:o,value:o}) 
    return rz;
});
const modellist = computed(() => {
    // 保留用户自定义、服务端 CUSTOM_MODELS 和历史对话模型。
    let groups: any[] = curatedModelGroups.map(g => ({
        ...g, children: g.children.map(m => ({ ...m }))
    }));
    const custom: string[] = [];
    const denied = new Set<string>();
    let allRemoved = false;
    (homeStore.myData.session.cmodels || '').split(/[ ,]+/ig).filter(Boolean).forEach((id: string) => {
        if (id === '-all') allRemoved = true;
        else if (id.startsWith('-')) denied.add(id.slice(1));
        else custom.push(id);
    });
    // 兼容已缓存的旧版前端：服务端可能下发 -all,model1,model2...
    // 对于新版界面，把 -all 当作允许清单；保留最新模型中文分组而非退化为裸 ID。
    const allowed = allRemoved ? new Set(custom) : null;
    groups = groups.map(g => ({
        ...g, children: g.children.filter((m: any) =>
            !denied.has(m.value) && (!allowed || allowed.has(m.value))
        )
    })).filter(g => g.children.length);
    if (gptConfigStore.myData.userModel) {
        custom.push(...gptConfigStore.myData.userModel.split(/[ ,]+/ig).filter(Boolean));
    }
    if (nGptStore.value?.model) custom.push(nGptStore.value.model);
    const seen = new Set(groups.flatMap(g => g.children.map((m: any) => m.value)));
    const extras = custom.filter(id => {
        if (!id || denied.has(id) || seen.has(id)) return false;
        seen.add(id);
        return true;
    }).map(id => ({ label: id + '（自定义/历史，注意核对价格）', value: id }));
    if (extras.length) groups.push({ type: 'group', key: 'custom', label: '自定义与历史模型', children: extras });
    if (!groups.length) groups.push({
        type: 'group', key: 'fallback', label: '备用模型',
        children: [{ label: 'GPT-4o mini', value: 'gpt-4o-mini' }]
    });
    return groups;
});
const isHistoricalModel = computed(() =>
    !curatedModelGroups.some(g => g.children.some(m => m.value === nGptStore.value?.model))
);
const ms= useMessage();
// const save = ()=>{ 
//     gptConfigStore.setMyData( nGptStore.value );
//     ms.success( t('common.saveSuccess')); //'保存成功'
//     emit('close');
// }
const saveChat=(type:string)=>{
     chatSet.save(  nGptStore.value );
     gptConfigStore.setMyData( nGptStore.value );
     homeStore.setMyData({act:'saveChat'}); 
     if(type!='hide')ms.success( t('common.saveSuccess'));
     emit('close');
}
 
watch(()=>nGptStore.value.model,(n)=>{
    nGptStore.value.gpts=undefined;
    let max=16384 *2;
    if( n.indexOf('gpt-3.5')>-1){
        max=4096*2;
    }else if(  n.indexOf('o1-mini')>-1){  
        max=65536 *2;
    }else if(  n.indexOf('o1-')>-1 || n=='o1' ){  
        max=65536 ;
    }else if( n=='gpt-4o-2024-08-06' || n=='chatgpt-4o-latest' || n.indexOf('gpt-4o')>-1 || n.indexOf('gpt-4.5')>-1){  
        max=16384 *2;
    }else if( n.indexOf('gpt-4')>-1 ||  n.indexOf('16k')>-1 ||  n.indexOf('o1-')>-1 ){ //['16k','8k','32k','gpt-4'].indexOf(n)>-1
        max=4096*2;
    }else if( n.toLowerCase().includes('claude-3-5')|| n.toLowerCase().includes('sonnet') 
        ||n.toLowerCase().includes('grok-3')
     ||  n.toLowerCase().includes('deepseek') ){ //deepseek
        max=4096*2*2;
    }else if( n.toLowerCase().includes('claude-3') ){
         max=4096*2;
    }

    config.value.maxToken=max/2;
    if(nGptStore.value.max_tokens> config.value.maxToken ) nGptStore.value.max_tokens= config.value.maxToken;
})

const reSet=()=>{
    gptConfigStore.setInit();
    nGptStore.value= gptConfigStore.myData;
}

onMounted(() => {
    //gptConfigStore.myData= chatSet.getGptConfig();
});

const serverSuccess=(s:any)=>{
    mlog('serverSuccess ', s  )
    nGptStore.value.model= s.model
}
//
//const f= ref({model:gptConfigStore.myData.model});
</script>
<template>
<section class="mb-4 flex justify-between items-center"  >
    <div class=" flex space-x-2 justify-between items-center">
     <div class="flex justify-start items-center">
        <span class="text-red-500">*</span>  
        {{ $t('mjset.model') }}
        
     </div>
     
    </div>
    <div  class="!w-[70%] flex justify-end items-center " >
       <div> 
        <n-select v-model:value="nGptStore.model" :options="modellist" size="small" filterable placeholder="选择最新模型或费用档位" />
        <div v-if="isHistoricalModel" class="text-[11px] text-amber-600 dark:text-amber-400">当前沿用历史模型；可在上方选择最新标准版或经济版。</div>
       </div>
       <div class=" pl-2" > 
        <!-- <NButton type="primary" @click="saveChat('no')" size="small" >{{ $t('mj.server_load') }}</NButton> -->
        <AiModelServer @success="serverSuccess"/>
        <div class="text-[11px] text-gray-500 dark:text-gray-400">从接口加载更多模型</div>
        <!-- 标准档优先；Basic 更省钱但偶有不稳定；无前缀模型请先核对费用。 -->
        <!-- <n-popselect
                v-model:value="st.server"
                :options="serverOptions"
                :render-label="renderLabel"
                size="medium"
                scrollable
            >
            <NTag  type="primary" round size="small" :bordered="false" class="!cursor-pointer">
            {{ $t('mj.server_load') }}
            
            </NTag>
        </n-popselect> -->
            

       </div>
    </div>
</section>
<section class="mb-4 flex justify-between items-center"  >
    <n-input   :placeholder="$t('mjchat.modlePlaceholder')" v-model:value="gptConfigStore.myData.userModel">
      <template #prefix>
        {{ $t('mjchat.myModle') }}
      </template>
    </n-input>
 </section>
 <section class=" flex justify-between items-center"  >
     <div> {{ $t('mjchat.historyCnt') }}
     </div>
     <div class=" flex justify-end items-center w-[80%] max-w-[240px]">
        <div class=" w-[200px]"><n-slider v-model:value="nGptStore.talkCount" :step="1" :max="50" /></div>
        <div  class="w-[50px] text-right">{{ nGptStore.talkCount }}</div>
    </div>
</section>
<div class="mb-4 text-[12px] text-gray-300 dark:text-gray-300/20">{{ $t('mjchat.historyToken') }}</div>

 <section class=" flex justify-between items-center"  >
     <div> {{ $t('mjchat.historyTCnt') }} 
     </div>
     <div class=" flex justify-end items-center w-[80%] max-w-[240px]">
        <div class=" w-[200px]"><n-slider v-model:value="nGptStore.max_tokens" :step="1" :max="config.maxToken" :min="1" /></div>
        <div  class="w-[50px] text-right">{{ nGptStore.max_tokens }}</div>
    </div>
</section>
<div class="mb-4 text-[12px] text-gray-300 dark:text-gray-300/20">{{ $t('mjchat.historyTCntInfo') }}  </div>

<!--禁用stream-->
<section  class=" flex justify-between items-center" >
     <div> {{ $t('禁用stream') }}</div>
     <div><n-switch v-model:value="nGptStore.forbidden_stream" size="small" /></div>
</section>
<div class="mb-4 text-[12px] text-gray-300 dark:text-gray-300/20">{{ $t('当禁用stream时,则使用非流式数据') }}  </div>

 <section class="mb-4"  >
    <div>{{ $t('mjchat.role') }}</div>
    <div>
     <n-input  type="textarea"  :placeholder=" $t('mjchat.rolePlaceholder') "   v-model:value="nGptStore.systemMessage" :autosize="{ minRows: 3 }"
    />
    </div>
 </section>

<template v-if="st.openMore">
    <section class=" flex justify-between items-center "  >
        <div>{{ $t('mj.temperature') }}</div>
        <div class=" flex justify-end items-center w-[80%] max-w-[240px]">
            <div class=" w-[200px]"><n-slider v-model:value="nGptStore.temperature" :step="0.01" :max="1" /></div>
            <div  class="w-[40px] text-right">{{ nGptStore.temperature }}</div>
        </div>
    </section>
    <div class="mb-4 text-[12px] text-gray-300 dark:text-gray-300/20"> {{ $t('mj.temperatureInfo') }}</div>


    <section class=" flex justify-between items-center "  >
        <div> {{ $t('mj.top_p') }}</div>
        <div class=" flex justify-end items-center w-[80%] max-w-[240px]">
            <div class=" w-[200px]"><n-slider v-model:value="nGptStore.top_p" :step="0.01" :max="1" /></div>
            <div  class="w-[40px] text-right">{{ nGptStore.top_p }}</div>
        </div>
    </section>
    <div class="mb-4 text-[12px] text-gray-300 dark:text-gray-300/20">{{ $t('mj.top_pInfo') }}</div>

    <section class=" flex justify-between items-center "  >
        <div> {{ $t('mj.presence_penalty') }}</div>
        <div class=" flex justify-end items-center w-[80%] max-w-[240px]">
            <div class=" w-[200px]"><n-slider v-model:value="nGptStore.presence_penalty" :step="0.01" :max="1" /></div>
            <div  class="w-[40px] text-right">{{ nGptStore.presence_penalty }}</div>
        </div>
    </section>
    <div class="mb-4 text-[12px] text-gray-300 dark:text-gray-300/20">{{ $t('mj.presence_penaltyInfo') }} </div>


    <section class=" flex justify-between items-center "  >
        <div>{{ $t('mj.frequency_penalty') }}</div>
        <div class=" flex justify-end items-center w-[80%] max-w-[240px]">
            <div class=" w-[200px]"><n-slider v-model:value="nGptStore.frequency_penalty" :step="0.01" :max="1" /></div>
            <div  class="w-[40px] text-right">{{ nGptStore.frequency_penalty }}</div>
        </div>
    </section>
    <div class="mb-4 text-[12px] text-gray-300 dark:text-gray-300/20">{{ $t('mj.frequency_penaltyInfo') }}</div>

    <section class="mb-4 flex justify-between items-center"  >
        <div >{{ $t('mj.tts_voice') }}</div>
        <n-select v-model:value="nGptStore.tts_voice" :options="voiceList" size="small"  class="!w-[50%]"   />
    </section>

    


</template>
<div v-else class="text-right cursor-pointer mb-4" @click="st.openMore=true">
    <NTag  type="primary" round size="small" :bordered="false" class="!cursor-pointer">More...</NTag>
</div>

 <section class=" text-right flex justify-end space-x-2"  >
    <NButton   @click="reSet()">{{ $t('mj.setBtBack') }}</NButton>
    <!-- <NButton type="primary" @click="saveChat">{{ $t('mj.setBtSaveChat') }}</NButton>
    <NButton type="primary" @click="save">{{ $t('mj.setBtSaveSys') }}</NButton> -->
    <NButton type="primary" @click="saveChat('no')">{{ $t('common.save') }}</NButton>
 </section>

 <!-- <NModal  v-model:show="st.isShow"  preset="card"  :title="$t('mjchat.modelChange')" class="!max-w-[820px]" @close="st.isShow=false" >
    Model内容
 </NModal> -->
</template>