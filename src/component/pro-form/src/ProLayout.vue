<!--
 * @file: 布局组件
 * @author: BoBo
 * @copyright: BoBo
 * @Date: 2021-03-28 19:21:19
-->
<template>
  <!-- 如果这一行时网格grid布局 -->
  <div v-if="item.type == 'grid'">
    <!-- 每一行元素与基于el-row和el-col生成 -->
    <el-row :key="item.key" type="flex" :gutter="item.options.gutter ? item.options.gutter : 0" :justify="item.options.justify" :align="item.options.align">
      <!-- 生成每一行中的每一列元素 -->
      <el-col v-for="(col, colIndex) in item.columns" :key="colIndex" :span="col.span">
        <!-- 遍历生成该列所有组件 -->
        <template v-for="citem in col.list">
          <ProLayout
            v-if="citem.type.includes('grid') || citem.type.includes('tabs')"
            :item="citem"
            :readOnly="readOnly"
            :models="models"
            :rules="rules"
            :remote="remote"
            :formTableConfig="formTableConfig"
            :key="citem.key"
            v-on="$listeners"
          >
            <template :slot="slotName" slot-scope="scope" v-for="slotName in Object.keys($scopedSlots)"> <slot :name="slotName" :model="scope.model" :widget="scope.widget"></slot> </template
          ></ProLayout>
          <!-- 正常组件通过ProFormItem生成 -->
          <ProFormItem
            v-else-if="!citem.hidden"
            @selection-change="getTableSelection($event, citem)"
            :key="citem.key"
            :models="models"
            :remote="remote"
            :widget="citem"
            :readOnly="readOnly"
            @btnOnClick="btnOnClick"
            @chartOnClick="chartOnClick"
            v-on="$listeners"
            :formTableConfig="formTableConfig"
          >
            <template :slot="slotName" slot-scope="scope" v-for="slotName in Object.keys($scopedSlots)">
              <slot :name="slotName" :model="scope.model" :widget="scope.widget"></slot>
            </template>
          </ProFormItem>
        </template>
      </el-col>
    </el-row>
  </div>
  <!-- 表格布局 -->
  <div v-else-if="item.type === 'grid-table'">
    <table
      class="grid-table"
      :key="item.key"
      style="width: 100%; border: solid"
      :style="{
        'border-width': item.options.borderWidth.toString() + 'px',
        borderColor: item.options.borderColor,
      }"
    >
      <tr v-for="(row, rowIndex) in item.rows" :key="rowIndex">
        <td
          v-for="(col, colIndex) in row.columns"
          :key="colIndex"
          :colspan="col.options.colspan || 1"
          :rowspan="col.options.rowspan || 1"
          @click.stop="clickTdAutoFocus($event, col)"
          valign="middle"
          align="left"
          class="grid-table-td"
          :style="{
            'border-width': item.options.borderWidth.toString() + 'px',
            borderColor: item.options.borderColor,
            width: col.options.width,
            height: col.options.height,
            display: col.list.every((_) => _.hidden) ? 'none' : '',
          }"
        >
          <template v-for="citem in col.list">
            <ProLayout
              v-if="citem.type.includes('grid') || col.type.includes('tabs')"
              :item="citem"
              :readOnly="readOnly"
              :models="models"
              :rules="rules"
              :remote="remote"
              :formTableConfig="formTableConfig"
              :key="citem.key"
            >
              <template :slot="slotName" slot-scope="scope" v-for="slotName in Object.keys($scopedSlots)"> <slot :name="slotName" :model="scope.model" :widget="scope.widget"></slot> </template
            ></ProLayout>
            <ProFormItem
              v-else-if="!citem.hidden"
              @selection-change="getTableSelection($event, citem)"
              :key="citem.key"
              :models="models"
              :remote="remote"
              :widget="citem"
              :readOnly="readOnly"
              @btnOnClick="btnOnClick"
              v-on="$listeners"
              @chartOnClick="chartOnClick"
              :formTableConfig="formTableConfig"
            >
              <template :slot="slotName" slot-scope="scope" v-for="slotName in Object.keys($scopedSlots)">
                <slot :name="slotName" :model="scope.model" :widget="scope.widget"></slot>
              </template>
            </ProFormItem>
          </template>
        </td>
      </tr>
    </table>
  </div>
  <!-- tab布局 -->
  <div v-else-if="item.type === 'tabs'">
    <Tabs
      :widget="item"
      :models="models"
      :remote="remote"
      :readOnly="readOnly"
      @selection-change="getTableSelection"
      @chartOnClick="chartOnClick"
      @btnOnClick="btnOnClick"
      v-on="$listeners"
      :formTableConfig="formTableConfig"
    >
      <template :slot="slotName" slot-scope="scope" v-for="slotName in Object.keys($scopedSlots)"> <slot :name="slotName" :model="scope.model" :widget="scope.widget"></slot> </template
    ></Tabs>
  </div>
  <!-- 普通行布局方式 -->
  <div v-else>
    <ProFormItem
      :key="item.key"
      :models="models"
      :remote="remote"
      @selection-change="getTableSelection($event, item)"
      :widget="item"
      :readOnly="readOnly"
      @chartOnClick="chartOnClick"
      @btnOnClick="btnOnClick"
      v-if="!item.hidden"
      v-on="$listeners"
      :formTableConfig="formTableConfig"
    >
      <template :slot="slotName" slot-scope="scope" v-for="slotName in Object.keys($scopedSlots)">
        <slot :name="slotName" :model="scope.model" :widget="scope.widget"></slot>
      </template>
    </ProFormItem>
  </div>
</template>

<script>
import VueCompositionApi, { defineComponent, ref, toRef, toRefs, watch, nextTick, watchEffect } from '@vue/composition-api';
import Vue from 'vue';
import Tabs from './components/Tabs/index.vue';

Vue.use(VueCompositionApi);

export default defineComponent({
  name: 'ProLayout',
  emits: ['table-selections', 'btn-on-click', 'chart-on-click'],
  components: {
    Tabs,
  },
  props: {
    item: {
      type: Object,
      default: () => ({}),
    },
    data: {
      type: Object,
      default: () => ({}),
    },
    models: {
      type: Object,
      default: () => ({}),
    },
    readOnly: {
      type: Boolean,
      default: false,
    },
    rules: {
      type: Array,
      default: () => [],
    },
    remote: {
      type: Object,
      default: () => ({}),
    },
    formTableConfig: {
      type: Object,
      default: () => ({}),
    },
  },
  setup(props, { root, emit }) {
    // const { $message } = root;

    const getTableSelection = ($event, item) => {
      const tableSelections = {};
      tableSelections[item.model] = $event;
      emit('table-selections', tableSelections);
    };

    // 生成的按钮点击
    const btnOnClick = (event) => {
      emit('btn-on-click', {
        event,
        model: props.models,
      });
    };

    // 图表点击
    const chartOnClick = (chart) => {
      emit('chart-on-click', {
        chart,
        model: models.value,
      });
    };

    // 单元格中为input,select,textarea时会默认聚焦
    const clickTdAutoFocus = (event, td) => {
      // 判断单元格中是否有组件
      if (td.list.length > 0) {
        const dom = event.target;
        const [target] = td.list;
        // 当点击单元格时,聚焦组件
        if (dom.tagName === 'TD') {
          switch (target.type) {
            case 'input':
              dom.getElementsByTagName('INPUT')[0].focus();
              break;
            case 'select':
              dom.getElementsByTagName('INPUT')[0].focus();
              break;
            case 'textarea':
              dom.getElementsByTagName('TEXTAREA')[0].focus();
              break;
            default:
              return false;
          }
        }
        return false;
      }
      return false;
    };

    return {
      getTableSelection,
      btnOnClick,
      chartOnClick,
      clickTdAutoFocus,
    };
  },
});
</script>
<style lang="scss" scoped>
@import './styles/grid-table-form.scss';
</style>
