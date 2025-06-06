<template>
  <div class="panel-tab__content">
    <el-radio-group
      v-if="type === 'UserTask'"
      v-model="approveMethod"
      @change="onApproveMethodChange"
    >
      <div class="flex-col">
        <div v-for="(item, index) in APPROVE_METHODS" :key="index">
          <el-radio :value="item.value" :label="item.value">
            {{ item.label }}
          </el-radio>
          <el-form-item prop="approveRatio">
            <el-input-number
              v-model="approveRatio"
              :min="10"
              :max="100"
              :step="10"
              size="small"
              v-if="
                item.value === ApproveMethodType.APPROVE_BY_RATIO &&
                approveMethod === ApproveMethodType.APPROVE_BY_RATIO
              "
              @change="onApproveRatioChange"
            />
          </el-form-item>
        </div>
      </div>
    </el-radio-group>
    <div v-else>
      除了UserTask以外节点的多实例待实现
    </div>
    <!-- 与Simple设计器配置合并，保留以前的代码 -->
    <el-form label-width="90px" style="display: none">
      <el-form-item label="快捷配置">
        <el-button size="small" @click="changeConfig('依次审批')">依次审批</el-button>
        <el-button size="small" @click="changeConfig('会签')">会签</el-button>
        <el-button size="small" @click="changeConfig('或签')">或签</el-button>
      </el-form-item>
      <el-form-item label="会签类型">
        <el-select v-model="loopCharacteristics" @change="changeLoopCharacteristicsType">
          <el-option label="并行多重事件" value="ParallelMultiInstance" />
          <el-option label="时序多重事件" value="SequentialMultiInstance" />
          <el-option label="无" value="Null" />
        </el-select>
      </el-form-item>
      <template
        v-if="
          loopCharacteristics === 'ParallelMultiInstance' ||
          loopCharacteristics === 'SequentialMultiInstance'
        "
      >
        <el-form-item label="循环数量" key="loopCardinality">
          <el-input
            v-model="loopInstanceForm.loopCardinality"
            clearable
            @change="updateLoopCardinality"
          />
        </el-form-item>
        <el-form-item label="集合" key="collection" v-show="false">
          <el-input v-model="loopInstanceForm.collection" clearable @change="updateLoopBase" />
        </el-form-item>
        <!-- add by 芋艿：由于「元素变量」暂时用不到，所以这里 display 为 none -->
        <el-form-item label="元素变量" key="elementVariable" style="display: none">
          <el-input v-model="loopInstanceForm.elementVariable" clearable @change="updateLoopBase" />
        </el-form-item>
        <el-form-item label="完成条件" key="completionCondition">
          <el-input
            v-model="loopInstanceForm.completionCondition"
            clearable
            @change="updateLoopCondition"
          />
        </el-form-item>
        <!-- add by 芋艿：由于「异步状态」暂时用不到，所以这里 display 为 none -->
        <el-form-item label="异步状态" key="async" style="display: none">
          <el-checkbox
            v-model="loopInstanceForm.asyncBefore"
            label="异步前"
            value="异步前"
            @change="updateLoopAsync('asyncBefore')"
          />
          <el-checkbox
            v-model="loopInstanceForm.asyncAfter"
            label="异步后"
            value="异步后"
            @change="updateLoopAsync('asyncAfter')"
          />
          <el-checkbox
            v-model="loopInstanceForm.exclusive"
            v-if="loopInstanceForm.asyncAfter || loopInstanceForm.asyncBefore"
            label="排除"
            value="排除"
            @change="updateLoopAsync('exclusive')"
          />
        </el-form-item>
        <el-form-item
          label="重试周期"
          prop="timeCycle"
          v-if="loopInstanceForm.asyncAfter || loopInstanceForm.asyncBefore"
          key="timeCycle"
        >
          <el-input v-model="loopInstanceForm.timeCycle" clearable @change="updateLoopTimeCycle" />
        </el-form-item>
      </template>
    </el-form>
  </div>
</template>

<script lang="ts" setup>
import { ApproveMethodType, APPROVE_METHODS } from '@/components/SimpleProcessDesignerV2/src/consts'

defineOptions({ name: 'ElementMultiInstance' })

const props = defineProps({
  businessObject: Object,
  type: String,
  id: String
})
const prefix = inject('prefix')
const loopCharacteristics = ref('')
//默认配置，用来覆盖原始不存在的选项，避免报错
const defaultLoopInstanceForm = ref({
  completionCondition: '',
  loopCardinality: '',
  extensionElements: [],
  asyncAfter: false,
  asyncBefore: false,
  exclusive: false
})
const loopInstanceForm = ref<any>({})
const bpmnElement = ref(null)
const multiLoopInstance = ref(null)
const bpmnInstances = () => (window as any)?.bpmnInstances

const getElementLoop = (businessObject) => {
  if (!businessObject.loopCharacteristics) {
    loopCharacteristics.value = 'Null'
    loopInstanceForm.value = {}
    return
  }
  if (businessObject.loopCharacteristics.$type === 'bpmn:StandardLoopCharacteristics') {
    loopCharacteristics.value = 'StandardLoop'
    loopInstanceForm.value = {}
    return
  }
  if (businessObject.loopCharacteristics.isSequential) {
    loopCharacteristics.value = 'SequentialMultiInstance'
  } else {
    loopCharacteristics.value = 'ParallelMultiInstance'
  }
  // 合并配置
  loopInstanceForm.value = {
    ...defaultLoopInstanceForm.value,
    ...businessObject.loopCharacteristics,
    completionCondition: businessObject.loopCharacteristics?.completionCondition?.body ?? '',
    loopCardinality: businessObject.loopCharacteristics?.loopCardinality?.body ?? ''
  }
  // 保留当前元素 businessObject 上的 loopCharacteristics 实例
  multiLoopInstance.value = bpmnInstances().bpmnElement.businessObject.loopCharacteristics
  // 更新表单
  if (
    businessObject.loopCharacteristics.extensionElements &&
    businessObject.loopCharacteristics.extensionElements.values &&
    businessObject.loopCharacteristics.extensionElements.values.length
  ) {
    loopInstanceForm.value['timeCycle'] =
      businessObject.loopCharacteristics.extensionElements.values[0].body
  }
}

const changeLoopCharacteristicsType = (type) => {
  // this.loopInstanceForm = { ...this.defaultLoopInstanceForm }; // 切换类型取消原表单配置
  // 取消多实例配置
  if (type === 'Null') {
    bpmnInstances().modeling.updateProperties(toRaw(bpmnElement.value), {
      loopCharacteristics: null
    })
    return
  }
  // 配置循环
  if (type === 'StandardLoop') {
    const loopCharacteristicsObject = bpmnInstances().moddle.create(
      'bpmn:StandardLoopCharacteristics'
    )
    bpmnInstances().modeling.updateProperties(toRaw(bpmnElement.value), {
      loopCharacteristics: loopCharacteristicsObject
    })
    multiLoopInstance.value = null
    return
  }
  // 时序
  if (type === 'SequentialMultiInstance') {
    multiLoopInstance.value = bpmnInstances().moddle.create(
      'bpmn:MultiInstanceLoopCharacteristics',
      { isSequential: true }
    )
  } else {
    multiLoopInstance.value = bpmnInstances().moddle.create(
      'bpmn:MultiInstanceLoopCharacteristics',
      { collection: '${coll_userList}' }
    )
  }
  bpmnInstances().modeling.updateProperties(toRaw(bpmnElement.value), {
    loopCharacteristics: toRaw(multiLoopInstance.value)
  })
}

// 循环基数
const updateLoopCardinality = (cardinality) => {
  let loopCardinality = null
  if (cardinality && cardinality.length) {
    loopCardinality = bpmnInstances().moddle.create('bpmn:FormalExpression', {
      body: cardinality
    })
  }
  bpmnInstances().modeling.updateModdleProperties(
    toRaw(bpmnElement.value),
    multiLoopInstance.value,
    {
      loopCardinality
    }
  )
}

// 完成条件
const updateLoopCondition = (condition) => {
  let completionCondition = null
  if (condition && condition.length) {
    completionCondition = bpmnInstances().moddle.create('bpmn:FormalExpression', {
      body: condition
    })
  }
  bpmnInstances().modeling.updateModdleProperties(
    toRaw(bpmnElement.value),
    multiLoopInstance.value,
    {
      completionCondition
    }
  )
}

// 重试周期
const updateLoopTimeCycle = (timeCycle) => {
  const extensionElements = bpmnInstances().moddle.create('bpmn:ExtensionElements', {
    values: [
      bpmnInstances().moddle.create(`${prefix}:FailedJobRetryTimeCycle`, {
        body: timeCycle
      })
    ]
  })
  bpmnInstances().modeling.updateModdleProperties(
    toRaw(bpmnElement.value),
    multiLoopInstance.value,
    {
      extensionElements
    }
  )
}

// 直接更新的基础信息
const updateLoopBase = () => {
  bpmnInstances().modeling.updateModdleProperties(
    toRaw(bpmnElement.value),
    multiLoopInstance.value,
    {
      collection: loopInstanceForm.value.collection || null,
      elementVariable: loopInstanceForm.value.elementVariable || null
    }
  )
}

// 各异步状态
const updateLoopAsync = (key) => {
  const { asyncBefore, asyncAfter } = loopInstanceForm.value
  let asyncAttr = Object.create(null)
  if (!asyncBefore && !asyncAfter) {
    // this.$set(this.loopInstanceForm, "exclusive", false);
    loopInstanceForm.value['exclusive'] = false
    asyncAttr = { asyncBefore: false, asyncAfter: false, exclusive: false, extensionElements: null }
  } else {
    asyncAttr[key] = loopInstanceForm.value[key]
  }
  bpmnInstances().modeling.updateModdleProperties(
    toRaw(bpmnElement.value),
    multiLoopInstance.value,
    asyncAttr
  )
}

const changeConfig = (config) => {
  if (config === '依次审批') {
    changeLoopCharacteristicsType('SequentialMultiInstance')
    updateLoopCardinality('1')
    updateLoopCondition('${ nrOfCompletedInstances >= nrOfInstances }')
  } else if (config === '会签') {
    changeLoopCharacteristicsType('ParallelMultiInstance')
    updateLoopCondition('${ nrOfCompletedInstances >= nrOfInstances }')
  } else if (config === '或签') {
    changeLoopCharacteristicsType('ParallelMultiInstance')
    updateLoopCondition('${ nrOfCompletedInstances > 0 }')
  }
}

/**
 * -----新版本多实例-----
 */
const approveMethod = ref()
const approveRatio = ref(100)
const otherExtensions = ref()
const getElementLoopNew = () => {
  if (props.type === 'UserTask') {
    const extensionElements =
      bpmnElement.value.businessObject?.extensionElements ??
      bpmnInstances().moddle.create('bpmn:ExtensionElements', { values: [] })
    approveMethod.value = extensionElements.values.filter(
      (ex) => ex.$type === `${prefix}:ApproveMethod`
    )?.[0]?.value

    otherExtensions.value =
      extensionElements.values.filter((ex) => ex.$type !== `${prefix}:ApproveMethod`) ?? []

    if (!approveMethod.value) {
      approveMethod.value = ApproveMethodType.SEQUENTIAL_APPROVE
      updateLoopCharacteristics()
    }
  }
}
const onApproveMethodChange = () => {
  approveRatio.value = 100
  updateLoopCharacteristics()
}
const onApproveRatioChange = () => {
  updateLoopCharacteristics()
}
const updateLoopCharacteristics = () => {
  // 根据ApproveMethod生成multiInstanceLoopCharacteristics节点
  if (approveMethod.value === ApproveMethodType.RANDOM_SELECT_ONE_APPROVE) {
    bpmnInstances().modeling.updateProperties(toRaw(bpmnElement.value), {
      loopCharacteristics: null
    })
  } else {
    if (approveMethod.value === ApproveMethodType.APPROVE_BY_RATIO) {
      multiLoopInstance.value = bpmnInstances().moddle.create(
        'bpmn:MultiInstanceLoopCharacteristics',
        { isSequential: false, collection: '${coll_userList}' }
      )
      multiLoopInstance.value.completionCondition = bpmnInstances().moddle.create(
        'bpmn:FormalExpression',
        {
          body: '${ nrOfCompletedInstances/nrOfInstances >= ' + approveRatio.value / 100 + '}'
        }
      )
    }
    if (approveMethod.value === ApproveMethodType.ANY_APPROVE) {
      multiLoopInstance.value = bpmnInstances().moddle.create(
        'bpmn:MultiInstanceLoopCharacteristics',
        { isSequential: false, collection: '${coll_userList}' }
      )
      multiLoopInstance.value.completionCondition = bpmnInstances().moddle.create(
        'bpmn:FormalExpression',
        {
          body: '${ nrOfCompletedInstances > 0 }'
        }
      )
    }
    if (approveMethod.value === ApproveMethodType.SEQUENTIAL_APPROVE) {
      multiLoopInstance.value = bpmnInstances().moddle.create(
        'bpmn:MultiInstanceLoopCharacteristics',
        { isSequential: true, collection: '${coll_userList}' }
      )
      multiLoopInstance.value.loopCardinality = bpmnInstances().moddle.create(
        'bpmn:FormalExpression',
        {
          body: '1'
        }
      )
      multiLoopInstance.value.completionCondition = bpmnInstances().moddle.create(
        'bpmn:FormalExpression',
        {
          body: '${ nrOfCompletedInstances >= nrOfInstances }'
        }
      )
    }
    bpmnInstances().modeling.updateProperties(toRaw(bpmnElement.value), {
      loopCharacteristics: toRaw(multiLoopInstance.value)
    })
  }

  // 添加ApproveMethod到ExtensionElements
  const extensions = bpmnInstances().moddle.create('bpmn:ExtensionElements', {
    values: [
      ...otherExtensions.value,
      bpmnInstances().moddle.create(`${prefix}:ApproveMethod`, {
        value: approveMethod.value
      })
    ]
  })
  bpmnInstances().modeling.updateProperties(toRaw(bpmnElement.value), {
    extensionElements: extensions
  })
}

onBeforeUnmount(() => {
  multiLoopInstance.value = null
  bpmnElement.value = null
})

watch(
  () => props.id,
  (val) => {
    if (val) {
      nextTick(() => {
        bpmnElement.value = bpmnInstances().bpmnElement
        // getElementLoop(val)
        getElementLoopNew()
      })
    }
  },
  { immediate: true }
)
</script>
