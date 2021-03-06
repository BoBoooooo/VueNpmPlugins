<!--
 * @file: 子表单
 * @author: BoBo
 * @copyright: BoBo
 * @Date: 2020-12-03 20:47:28
-->
<template>
  <div
    class="widget-table"
    :class="{
      active: selectWidget.key == element.key,
    }"
  >
    <div class="widget-fake-table">
      <div class="header">#</div>
      <div class="content">1</div>
    </div>
    <Draggable
      v-model="element.tableColumns"
      v-bind="{
        group: 'people',
        ghostClass: 'ghost',
        animation: 200,
        handle: '.drag-widget',
      }"
      style="flex: 1"
      @add="handleWidgetAdd"
    >
      <transition-group name="fade" tag="div" class="widget-sub-form-list">
        <div
          class="table-view"
          :class="{
            active: selectWidget.key == element.key,
          }"
          :style="{
            width: `${element.options.width}px`,
          }"
          v-for="(element, index) in tableColumns"
          :key="element.key"
        >
          <div class="table-header">{{ element.name }}</div>
          <WidgetFormItem
            v-if="element && element.key"
            :element="element"
            :customListField="tableColumns"
            :select.sync="selectWidget"
            :index="index"
            :data="data"
            @click.native.stop="handleSelectWidget(index)"
          ></WidgetFormItem>
        </div>
      </transition-group>
    </Draggable>
  </div>
</template>

<script lang="ts">
import { Component, Vue, Prop, Watch } from 'vue-property-decorator';
import Draggable from 'vuedraggable';
import { formElement } from '../../componentsConfig';

@Component({
  name: 'WidgetSubForm',
  components: {
    Draggable,
  },
})
export default class WidgetSubForm extends Vue {
  $refs!: {
    table: HTMLFormElement;
  };

  @Prop({
    type: Number,
    default: 0,
  })
  index!: number;

  @Prop({
    type: Object,
    default: () => ({}),
  })
  select!: Object;

  @Prop({
    type: Object as () => {
      tableColumns: any;
    },
    default: () => ({}),
  })
  element!: {
    tableColumns: any[];
  };

  @Prop({
    type: Object,
    default: () => ({}),
  })
  data!: any;

  tableData = [];

  selectWidget = this.select;

  formElement = formElement;

  handleSelectWidget(index) {
    this.selectWidget = this.element.tableColumns[index];
  }

  handleWidgetAdd(evt) {
    const { newIndex } = evt;
    const { model, type } = this.element.tableColumns[newIndex];

    // 防止布局元素的嵌套拖拽
    if (!formElement.includes(type)) {
      this.element.tableColumns.splice(newIndex, 1);
      this.$message.warning('仅支持拖拽表单组件');
      return false;
    }

    // 为拖拽到容器的元素添加唯一 key
    const key = model || `${type}_${Math.ceil(Math.random() * 99999)}`;
    this.$set(this.element.tableColumns, newIndex, {
      ...this.element.tableColumns[newIndex],
      options: {
        ...this.element.tableColumns[newIndex].options,
        remoteFunc: `func_${key}`,
        hiddenLabel: true, // 子表单默认隐藏标签
        width: 200, // 默认列宽200px
      },
      key,
      // 绑定键值
      // model: this.element.tableColumns[newIndex].type + '_' + key,
      model: key,
      rules: [],
    });

    if (this.element.tableColumns[newIndex].type === 'radio' || this.element.tableColumns[newIndex].type === 'checkbox') {
      this.$set(this.element.tableColumns, newIndex, {
        ...this.element.tableColumns[newIndex],
        options: {
          ...this.element.tableColumns[newIndex].options,
          options: this.element.tableColumns[newIndex].options.options.map((item) => ({
            ...item,
          })),
          hiddenLabel: true, // 子表单默认隐藏标签
          width: 200, // 默认列宽200px
        },
      });
    }
    console.log(this.element.tableColumns[newIndex]);
    this.selectWidget = this.element.tableColumns[newIndex];
    return null;
  }

  get tableColumns() {
    return this.element.tableColumns;
  }

  @Watch('selectWidget', {
    deep: true,
  })
  selectWidgetOnChange(val) {
    this.$emit('update:select', val);
  }

  @Watch('select')
  selectOnChange(val) {
    this.selectWidget = val;
  }
}
</script>
