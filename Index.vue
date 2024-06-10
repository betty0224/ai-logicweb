<template>
  <div class="flex h-full flex-col">
    <HeaderLayout :backBtn="true" @back="back">
      <template v-slot:title >
        <h5 class="header-title-wrap">任务配置</h5>
      </template>
    </HeaderLayout>
    <div class="flex-1 m-4 rounded-lg bg-general-100 px-6 pt-6" v-loading="loading">
      <SectionHead title="训练配置" />
      <a-form
        :form="fromTraining"
        :label-col="{ span: 3 }" :wrapper-col="{ span: 21 }"
        class="mt-6"
      >
        <a-form-item :label="trainingData.model_id.label">
          <a-select class="form-item"
          :placeholder="trainingData.model_id.placeholder"
          v-decorator="[trainingData.model_id.key,{
            rules: trainingData.model_id.rules,
            initialValue: trainingData.model_id.value
          }]"
          @change="onChange($event, 'model_id')"
          >
            <a-select-option :value="item.model_id" v-for="item in trainingData.model_id.optList" :key="item.model_id">{{item.name}}</a-select-option>
          </a-select>
        </a-form-item>
        <a-form-item :label="trainingData.llm_type.label">
          <a-select class="form-item"
          :placeholder="trainingData.llm_type.placeholder"
          v-decorator="[trainingData.llm_type.key,{
            rules: trainingData.llm_type.rules,
            initialValue: trainingData.llm_type.value
          }]"
          >
            <a-select-option :value="item" v-for="item in trainingData.llm_type.optList" :key="item">{{item}}</a-select-option>
          </a-select>
        </a-form-item>
        <a-form-item :label="trainingData.train_method.label">
          <a-radio-group
          v-decorator="[trainingData.train_method.key,{
            rules: trainingData.train_method.rules,
            initialValue: trainingData.train_method.value
          }]"
          >
            <a-radio v-for="item in trainingData.train_method.optList" :key="item.value" :value="item.value">{{item.label}}</a-radio>
          </a-radio-group>
        </a-form-item>
        <a-form-item :label="trainingData.params.label">
          <Hparams ref="hParams" layout="card" :data="paramsValue" :paramsData="hParamsData" />
          <Hparams v-show="moreParams" ref="additionalParams" layout="card" :data="paramsValue" :paramsData="additionalParamsData" />
          <a v-if="!loading" class="text-primary-900 hover:underline" @click="changeMoreParams">{{moreParams ? '收起设置' : '高级设置'}}<hz-icon class="transform" :class="{'rotate-180': moreParams}" hzName="triangle"/></a>
        </a-form-item>
        <a-form-item>
          <Hparams ref="resourcesParams" layout="list" :data="paramsValue" :paramsData="resourcesParamsData" />
        </a-form-item>
      </a-form>
      <SectionHead title="数据配置" />
      <a-form
        :form="fromTraining"
        :label-col="{ span: 3 }" :wrapper-col="{ span: 21 }"
        class="mt-6"
      >
        <a-form-item :label="trainingData.ds_id.label" >
          <a-select class="form-item"
          :placeholder="trainingData.ds_id.placeholder"
          v-decorator="[trainingData.ds_id.key,{
            rules: trainingData.ds_id.rules,
            initialValue: trainingData.ds_id.value
          }]"
          >
            <a-select-option :value="item.ds_id" v-for="item in trainingData.ds_id.optList" :key="item.ds_id">{{item.name}}</a-select-option>
          </a-select>
        </a-form-item>
        <a-form-item :label="trainingData.split_ratio.label">
          <a-input-number
            v-decorator="[trainingData.split_ratio.key,{
              rules: trainingData.split_ratio.rules,
              initialValue: trainingData.split_ratio.value
            }]"
            :disabled="trainingData.split_ratio.disabled"
            :min="trainingData.split_ratio.min" :max="trainingData.split_ratio.max"
          class="form-item" :placeholder="trainingData.split_ratio.placeholder"/>
          <a-checkbox class="ml-2" v-model="trainingData.split_ratio.isFull" @change="onChange($event, 'split_ratio_full')">全数据训练测试</a-checkbox>
          <p class="text-type-700 mt-2">{{trainingData.split_ratio.desc}}</p>
        </a-form-item>
        <a-form-item :wrapper-col="{span: 21, offset:3}" class="mt-12">
          <a-button type="primary" @click="save">确定</a-button>
          <a-button class="ml-2" @click="back">取消</a-button>
        </a-form-item>
      </a-form>
    </div>
  </div>
</template>
<script>
import { HeaderLayout, SectionHead } from '@hertz/layout'
import { getJobDetail, getModelList, getDatasetList, configModify, getModelTemplate, getClusterMonitor, getHparams } from '../api.js'
import { traningMethod } from '../config.js'
import { mixins, regExp } from '@hertz/utils'
import { isEmpty } from 'lodash'
import Hparams from '../../h-params/Index.vue'
export default {
  name: 'SuperviseDebugConfig',
  components: {
    HeaderLayout,
    SectionHead,
    Hparams
  },
  mixins: [
    mixins.back
  ],
  data () {
    return {
      moreParams: false,
      jobId: '',
      fromTraining: this.$form.createForm(this, { name: 'define' }),
      // fromDataConfig: this.$form.createForm(this, { name: 'data' }),
      trainingData: {
        model_id: {
          label: '选择基础模型',
          require: true,
          key: 'model_id',
          name: 'model_name',
          value: undefined,
          placeholder: '请选择',
          optList: [],
          rules: [
            {
              required: true,
              message: '请选择基础模型'
            }
          ]
        },
        llm_type: {
          label: '选择模型框架',
          require: true,
          key: 'llm_type',
          value: undefined,
          placeholder: '请选择',
          optList: [],
          rules: [
            {
              required: true,
              message: '请选择模型框架'
            }
          ]
        },
        train_method: {
          key: 'train_method',
          label: '训练方法',
          value: 'lora',
          optList: traningMethod,
          rules: [
            {
              required: true,
              message: '请选择训练方法'
            }
          ]
        },
        params: {
          label: '参数配置',
          key: 'params'
        },
        ds_id: {
          key: 'ds_id',
          label: '选择数据集',
          value: undefined,
          placeholder: '请选择',
          optList: [],
          rules: [
            {
              required: true,
              message: '请选择数据集'
            }
          ]
        },
        split_ratio: {
          key: 'split_ratio',
          label: '数据拆分比例',
          require: true,
          value: 20,
          min: 0,
          max: 50,
          disabled: false,
          isFull: false,
          placeholder: '请输入1-50的正整数',
          desc: '即模型训练集和验证集的比例，例如：设置20，则表示选定数据集版本总数的80%作为训练集，20%作为验证集',
          rules: [
            {
              required: true,
              message: '请输入数据拆分比例'
            },
            {
              pattern: regExp.integer.regExp, message: regExp.integer.msg
            }
          ]
        }
      },
      loading: false,
      hParamsData: null,
      additionalParamsData: null,
      paramsValue: null,
      resourcesParamsData: null
    }
  },
  created () {
    this.init()
  },
  methods: {
    async init () {
      this.jobId = this.$route.params.id
      await this.getHparams()
      // await this.getClusterMonitor()
      this.getJobDetail()
      this.getModelList()
      this.getDatasetList()
      this.getModelTemplate()
    },
    async getClusterMonitor () {
      this.loading = true
      const { status, result } = await getClusterMonitor()
      this.loading = false
      if (status !== '0') return
      const trainingClusterData = result.data.filter((item) => { return item.host.tag === 'training' })
      const clusterMonitorList = []
      trainingClusterData.forEach((item) => {
        const gups = item.gpu.map((it) => {
          const o = { ...it }
          o.label = `[${it.index}] GRAM:${it.memory_used}/${it.memory_total}MB`
          return o
        })
        clusterMonitorList.push({
          ...item.host,
          gpus: [
            ...gups
          ]
        })
      })
      this.trainingData.host.optList = [...clusterMonitorList]
    },
    // 模型列表
    async getModelList () {
      const res = await getModelList({
        page: 1,
        page_size: 10000
      })
      if (res.status !== '0') return
      this.trainingData.model_id.optList = [...res.result.data]
    },
    // 模型框架列举表
    async getModelTemplate () {
      const { status, result } = await getModelTemplate()
      if (status !== '0') return
      this.trainingData.llm_type.optList = [...result.model_list]
    },
    async getDatasetList () {
      const res = await getDatasetList({
        page: 1,
        page_size: 10000
      })
      if (res.status !== '0') return
      this.trainingData.ds_id.optList = [...res.result.data]
    },
    async getJobDetail () {
      this.loading = true
      const { status, result } = await getJobDetail({
        job_id: this.jobId
      })
      this.loading = false
      if (status !== '0') return
      Object.keys(this.trainingData).forEach((item) => {
        if (item !== 'params' && result[item] !== undefined) {
          this.trainingData[item].value = result[item]
          if (item === 'split_ratio' && result[item] === 0) {
            this.trainingData[item].disabled = true
            this.trainingData[item].isFull = true
          }
        }
      })
      if (!isEmpty(result.params)) {
        this.paramsValue = { ...result.params }
        const hParams = {}
        const additionalParams = {}
        this.hParamsData.forEach((item) => {
          hParams[item.name] = result.params[item.name]
        })
        this.additionalParamsData.forEach((item) => {
          additionalParams[item.name] = result.params[item.name]
        })
        this.$nextTick(() => {
          this.$refs.hParams.setValues(this.paramsValue)
          this.$refs.additionalParams.setValues(this.paramsValue)
          this.$refs.resourcesParams.setValues(this.paramsValue)
        })
      }
    },
    async getHparams () {
      this.loading = true
      const { status, result } = await getHparams()
      this.loading = false
      if (status !== '0') return
      this.hParamsData = [...result.hparams]
      this.additionalParamsData = [...result.additional]
      this.resourcesParamsData = [...result.resource]
    },
    async save () {
      this.fromTraining.validateFields(async (err, values) => {
        if (!err) {
          const params = {
            ...values,
            job_id: this.jobId
          }
          const hParams = await this.$refs.hParams.getValues()
          const resourceParams = await this.$refs.resourcesParams.getValues()
          const additionalParams = await this.$refs.additionalParams.getValues()
          params.params = {
            ...hParams,
            ...resourceParams,
            ...additionalParams
          }
          this.loading = true
          const { status } = await configModify(params)
          this.loading = false
          if (status !== '0') return
          this.$hzRouter.back()
          this.$message.success('修改成功')
        }
      })
    },
    changeHost (event) {
      this.setGpusOptList(event)
      this.trainingData.gpus.value = undefined
    },
    setGpusOptList (value) {
      const host = this.trainingData.host.optList.find((item) => item.ip === value)
      this.trainingData.gpus.optList = [...host.gpus]
    },
    onChange (event, trigger) {
      if (trigger === 'model_id') {
        const model = this.trainingData.model_id.optList.find((item) => item.model_id === event)
        const llmType = this.trainingData.llm_type.optList.find((item) => item === model.adapter)
        if (llmType) {
          this.fromTraining.setFieldsValue({
            llm_type: model.adapter
          })
        }
      }
      if (trigger === 'split_ratio_full') {
        const split_ratio = this.trainingData.split_ratio
        split_ratio.isFull = event.target.checked
        split_ratio.disabled = event.target.checked
        const value = split_ratio.isFull ? 0 : ''
        this.fromTraining.setFieldsValue({
          split_ratio: value
        })
      }
    },
    changeMoreParams () {
      this.moreParams = !this.moreParams
    }
  }
}
</script>
<style lang="scss" scoped>
  ::v-deep {
    form .form-item, form .form-item {
      @apply w-116
    }
    // .ant-slider {
    //   @apply ml-0 mt-4.5;
    // }
    // .ant-form-item-children .ant-form-item-control {
    //   @apply border-1 border-solid border-mono-a-300 px-3 py-4 rounded-lg relative h-30
    // }
    // .ant-form-item-children .ant-form-item-control-wrapper {
    //   @apply pl-0
    // }
  }
</style>
