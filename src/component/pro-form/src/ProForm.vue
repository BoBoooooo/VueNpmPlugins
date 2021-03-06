<!--
@file 表单生成
@author BoBo
@createDate 2018年11月15日21:40:56
@copyright GavinZhuLei
部分修改重构
原作者地址 https://github.com/GavinZhuLei/vue-form-making
感谢大佬!

-->
<template>
  <div class="table-form-wrapper">
    <el-form
      ref="proForm"
      :class="{
        pad: deviceMode === 'pad',
        mobile: deviceMode === 'mobile',
      }"
      class="form"
      :model="models"
      :label-position="data && data.config && data.config.labelPosition"
      :label-width="data && data.config && data.config.labelWidth ? data.config.labelWidth + 'px' : '140px'"
      :size="data && data.config && data.config.size"
    >
      <!-- 遍历从父组件传入的data，data下有list和config两个属性，list下的每个对象是表示一行组件的集合 -->
      <template v-for="(item, index) in (data && data.list) || []">
        <ProLayout :models="models" :key="index" :data="data" v-on="$listeners" :item="item" :readOnly="readOnly" :rules="rules" :remote="remote" :formTableConfig="formTableConfig">
          <template :slot="slotName" slot-scope="scope" v-for="slotName in Object.keys($scopedSlots)">
            <slot :name="slotName" :model="scope.model" :widget="scope.widget"></slot>
          </template>
        </ProLayout>
      </template>
    </el-form>
  </div>
</template>

<script lang="ts">
import { Component, Vue, Prop, Watch } from 'vue-property-decorator';
import { AnyObject } from '@/types/common';
import { formElement } from '@/component/form-designer/src/componentsConfig';
import dayjs from 'dayjs';
import ProLayout from './ProLayout.vue';

@Component({
  components: {
    ProLayout,
  },
  name: 'ProForm',
})
export default class ProForm extends Vue {
  $refs!: {
    proForm: HTMLFormElement;
    FormDesigner: HTMLFormElement;
  };

  @Prop({
    type: Object,
    default: () => ({
      config: {},
      list: [],
    }),
  })
  data: any;

  // 表单初始值
  @Prop({
    type: Object,
    default: () => ({}),
  })
  value: any;

  // 表单当前实时对象
  @Prop({
    type: Object,
    default: () => ({}),
  })
  entity: any;

  // 是否只读
  @Prop({
    type: Boolean,
    default: false,
  })
  readOnly!: boolean;

  // 组件联动规则
  @Prop({
    type: Array,
    default: () => [],
  })
  rules!: any[];

  // 远端数据
  @Prop({
    type: Object,
    default: () => ({}),
  })
  remote: any;

  // 表格组件 tableParams以及prefill
  @Prop({
    type: Object,
    default: () => ({}),
  })
  formTableConfig: any;

  // deviceMode
  @Prop({
    type: String,
    default: 'pc',
  })
  deviceMode!: string;

  // 当前表单实体对象
  models: any = {};

  // 内部属性记录字段对应组件,用于联动时快速修改
  fieldMap: any = {};

  // 内部属性记录字段已联动的组件,用于字段值切换时重置原先组件状态
  linkEffect: any = {};

  created() {
    if (this.data && this.data.list) {
      // 根据数据结构生成给子组件的数据源
      this.generateModel(this.data.list);
    }
  }

  // 组件联动handler
  controlFieldHandler(model) {
    this.rules.forEach((r) => {
      const value = this.models[r.field];
      // 若该字段已经联动其他组件,先恢复被联动组件为原先状态
      if (this.linkEffect[r.field]) {
        this.resetComponent(this.linkEffect[r.field]);
      }
      if (value) {
        const controlRule = r.control.filter((_) => value.includes(_.value));
        if (controlRule.length > 0) {
          const insideRule: any = [];
          for (const rules of controlRule) {
            insideRule.push(...rules.rule);
          }
          if (insideRule.length > 0) {
            this.linkEffect[r.field] = [];
            insideRule.forEach((_: any) => {
              const field = this.fieldMap[_.field];
              if (field) {
                switch (_.operator) {
                  case 'show':
                    this.$set(field, 'hidden', false);
                    // 如果被联动组件值为空,默认显示规则中默认值
                    if ((!this.models[_.field] || this.models[_.field].length === 0) && _.value) {
                      if (Array.isArray(this.models[_.field])) {
                        this.models[_.field].push(_.value);
                      } else {
                        this.models[_.field] = _.value;
                      }
                    }
                    break;
                  case 'hidden':
                    this.$set(field, 'hidden', true);
                    break;
                  case 'required':
                    this.setRequired(field);
                    break;
                  case 'unrequired':
                    this.setUnRequired(field);
                    break;
                  default:
                    break;
                }
                // linkEffect 缓存当前字段跟其他组件联动的关系
                this.linkEffect[r.field].push(_);
              }
            });
          }
        }
      }
    });
  }

  setRequired(field: any) {
    if (!field.options.required) {
      field.options.required = true;
      field.rules.push({
        message: `${field.name}必须填写`,
        required: true,
      });
    }
  }

  resetComponent(effect) {
    effect.forEach((_) => {
      const field = this.fieldMap[_.field];
      if (field) {
        switch (_.operator) {
          case 'show':
            this.$set(field, 'hidden', true);
            break;
          case 'hidden':
            this.$set(field, 'hidden', false);
            this.models[_.field] = _.value ? _.value : null;
            break;
          case 'required':
            this.setUnRequired(field);
            break;
          case 'unrequired':
            this.setRequired(field);
            break;
          default:
            break;
        }
      }
    });
  }

  setUnRequired(field: any) {
    field.options.required = false;
    field.rules = field.rules.filter((_) => !_.required);
  }

  generateModel(genList, initValue = true) {
    // 遍历设计的结构
    for (let i = 0; i < genList.length; i += 1) {
      if (genList[i].type === 'grid') {
        genList[i].columns.forEach((item) => {
          this.generateModel(item.list);
        });
      } else if (genList[i].type === 'grid-table') {
        genList[i].rows.forEach((item) => {
          this.generateModel(item.columns);
        });
      } else if (genList[i].type === 'td') {
        this.generateModel(genList[i].list);
      } else if (genList[i].type === 'form') {
        this.generateModel(genList[i].tableColumns, false);
      } else {
        // 获取当前组件
        const row = genList[i];
        // 如果是自定义组件,model值为slotName,不在model中赋属性值
        if (Object.keys(this.value).indexOf(row.model) >= 0 && formElement.includes(row.type)) {
          this.initFormValue(row);
        } else if (formElement.includes(row.type) && initValue) {
          this.setDefaultValue(row);
        }
        // 组件option跟字段映射
        this.$set(this.fieldMap, row.model, row);
      }
    }
    this.models = { ...this.value, ...this.models };
  }

  // 多选情况下数组转字符串
  filterFormData() {
    const model = { ...this.models };
    Object.keys(model).forEach((k) => {
      if (Array.isArray(model[k])) {
        model[k] = model[k].toString();
      }
    });
    return model;
  }

  /**
   * 如果select,radio,checkbox等组件为多选情况  后台返回逗号分隔字符串 => 数组
   * 如果 this.value为null 则会按默认值赋值相应字段
   * @param {String} 当前表单json.list
   */
  initFormValue(row) {
    if (row.options.multiple || 'cascader,checkbox'.includes(row.type) || (row.options.type && row.options.type.includes('range'))) {
      if (this.value[row.model] != null && this.value[row.model] !== '') {
        this.models[row.model] = this.value[row.model].split(',');
      } else {
        this.models[row.model] = [];
      }
    } else if (this.value[row.model] == null || this.value[row.model] === '') {
      this.setDefaultValue(row);
    } else {
      this.models[row.model] = this.value[row.model];
    }
  }

  // 表单默认值回填单独拉出来封装
  setDefaultValue(config) {
    // 如果时间选择器需要默认值,默认回填当前日期
    if (config.type === 'date' || config.type === 'time') {
      if (config.options.defaultValue) {
        const { format } = config.options;
        if (format) {
          console.log(format);
          this.models[config.model] = dayjs().format(config.type === 'time' ? format : format.toUpperCase());
        }
      } else {
        this.models[config.model] = null;
      }
    } else {
      let { defaultValue } = config.options;
      // 如果默认值设置为$开头,则表示要读取vuex中的全局变量
      // 如设置为 $deptname 则读取 this.$store.getters.deptname
      if (typeof defaultValue === 'string' && defaultValue.includes('$')) {
        defaultValue = this.$store.getters[defaultValue.replace('$', '')];
      } else if (defaultValue === '') {
        defaultValue = null;
      }
      this.models[config.model] = defaultValue;
    }
  }

  // ----全局api----
  // 先验证再获取表单内容
  getData(original: boolean) {
    return new Promise((resolve, reject) => {
      this.$refs.proForm.validate((valid, obj) => {
        if (valid) {
          resolve(original ? JSON.parse(JSON.stringify(this.models)) : this.filterFormData());
        } else {
          // 校验失败时focus到文本框
          setTimeout(() => {
            const isError: any = document.getElementsByClassName('is-error');
            if (isError[0].querySelector('input')) {
              isError[0].querySelector('input').focus();
            } else if (isError[0].querySelector('textarea')) {
              isError[0].querySelector('textarea').focus();
            }
          }, 100);
          reject(obj);
        }
      });
    });
  }

  // 重置表单
  resetForm() {
    this.$refs.proForm.resetFields();
  }

  // 校验表单
  validate() {
    return new Promise((resolve, reject) => {
      this.$refs.proForm.validate((valid, obj) => {
        if (valid) {
          resolve();
        } else {
          reject(obj);
        }
      });
    });
  }

  // 不经过验证直接获取表单内容
  getDataWithoutValidate(original: boolean) {
    return new Promise((resolve) => resolve(original ? JSON.parse(JSON.stringify(this.models)) : this.filterFormData()));
  }

  /**
   * 更新某个字段值
   */
  setFieldValue(field: string | AnyObject, value?) {
    if (typeof field === 'object') {
      Object.keys(field).forEach((key) => {
        this.$set(this.models, key, field[key]);
      });
    } else {
      this.$set(this.models, field, value);
    }
  }

  /**
   * 获取某个字段值
   */
  getFieldValue(field: string) {
    return this.models[field];
  }

  /**
   * 异步更新整个表单值
   */
  setFormValue(value) {
    if (this.$refs.proForm) {
      this.$refs.proForm.clearValidate();
    }
    this.models = JSON.parse(JSON.stringify(value));
  }

  // 统一封装字段状态改动
  _changeFieldStatus(status: boolean, field: string | string[] | undefined, type) {
    let _target: string[];
    if (field) {
      _target = typeof field === 'string' ? [field] : field;
    } else {
      _target = Object.keys(this.fieldMap);
    }

    _target.forEach((key) => {
      switch (type) {
        case 'hidden':
          this.$set(this.fieldMap[key], 'hidden', status);
          break;
        case 'disabled':
          this.$set(this.fieldMap[key].options, 'disabled', status);
          break;
        default:
          console.error('设置失败');
      }
    });
  }

  disabled(status: boolean, field?: string | string[]) {
    this._changeFieldStatus(status, field, 'disabled');
  }

  hidden(status: boolean, field?: string | string[]) {
    this._changeFieldStatus(status, field, 'hidden');
  }

  @Watch('value', {
    deep: true,
  })
  valueOnChange(val) {
    if (this.$refs.proForm) {
      this.$refs.proForm.clearValidate();
    }
    this.models = { ...this.models, ...val };
  }

  @Watch('models', {
    deep: true,
    // immediate: true,
  })
  modelsOnChange(val) {
    // 组件值改变时检查是否需要联动其他组件
    if (this.rules.length > 0) {
      this.controlFieldHandler(val);
      if (this.$refs.proForm) {
        this.$refs.proForm.clearValidate();
      }
    }
    this.$emit('update:entity', val);
  }
}
</script>
