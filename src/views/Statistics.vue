<template>
  <Layout>
    <Tabs class-prefix="type" :data-source="recordTypeList" :value.sync="type"/>
    <div class="chart-wrapper" ref="chartWrapper">
      <Chart class="chart" :options="chartOptions"/>
    </div>
    <ol v-if="groupedList.length>0">
      <li v-for="(group, index) in groupedList" :key="index">
        <h3 class="title">{{ beautify(group.title) }} <span>￥{{ group.total }}</span></h3>
        <ol>
          <li v-for="item in group.items" :key="item.id"
              class="record">
            <span>{{ tagString(item.tags) }}</span>
            <span class="notes">{{ item.notes }}</span>
            <span>￥{{ item.amount }} </span>
          </li>
        </ol>
      </li>
    </ol>
    <div v-else class="noResult">
      目前没有相关记录
    </div>
  </Layout>
</template>

<script lang="ts">
import Vue from 'vue';
import {Component} from 'vue-property-decorator';
import Tabs from '@/components/Tabs.vue';
import recordTypeList from '@/constants/recordTypeList';
import dayjs from 'dayjs';
import clone from '@/lib/clone';
import Chart from '@/components/Chart.vue';
import _ from 'lodash';

@Component({
  components: {Tabs, Chart},
})
export default class Statistics extends Vue {
  // eslint-disable-next-line no-undef
  tagString(tags: Tag[]) {
    return tags.length === 0 ? '无' : tags.map(t => t.name).join('，');
  }

  mounted() {
    if (this.$refs.chartWrapper) {
      const div = this.$refs.chartWrapper as HTMLDivElement;
      div.scrollLeft = div.scrollWidth;
    }
  }
  get keyValueList(){
    // console.log(this.groupedList);
    const today = new Date();
    const array = [];
    for (let i = 0; i <= 29; i++) {
      // this.recordList=[{createdAt:'2022-09-01',amount:1.00,...},{},]
      // this.groupedList=[{title:'2022-09-01',items:[{},{},],total:1.00,...},{},]
      // array=[{key:'2022-09-01',value:1.00},{},]

      const dateString = dayjs(today).subtract(i, 'day').format('YYYY-MM-DD');

      const found = _.find(this.groupedList, {title: dateString});

      array.push({key: dateString, value: found ? found.total : 0});
    }
    array.sort((a, b) => {
      if (a.key > b.key) {
        return 1;
      } else if (a.key === b.key) {
        return 0;
      } else {
        return -1;
      }
    });
    // console.log(array);
    // console.log(this.recordList.map(r => _.pick(r, 'createdAt', 'amount')));
    return array
  }

  get chartOptions() {
    const dates = this.keyValueList.map(item => item.key);
    const values = this.keyValueList.map(item => item.value);
    // console.log(values);
    return {
      //消除echarts的四周padding(来自谷歌：echarts padding解答，事实上官方文档里echarts的绘图区域是grid)
      grid: {
        left: 0,
        right: 0,
        top: 0,
        bottom: 30
      },
      xAxis: {
        axisTick: {
          show: true,
          alignWithLabel: true
        },
        axisLabel:{
          // 使用函数模板，函数参数分别为刻度数值（类目），刻度的索引
          formatter: function (value:string, index:number) {
            // 格式化成月-日
            return value.substring(5);
          }
        },
        type: 'category',
        data: dates
      },
      yAxis: {
        type: 'value',
        show: false
      },
      series: [{
        symbolSize: 12,
        data: values,
        type: 'line'
      }],
      tooltip: {show: true}
    };
  }

  beautify(string: string) {
    const day = dayjs(string);
    const now = dayjs();
    if (day.isSame(now, 'day')) {
      return '今天';
    } else if (day.isSame(now.subtract(1, 'day'), 'day')) {
      return '昨天';
    } else if (day.isSame(now.subtract(2, 'day'), 'day')) {
      return '前天';
    } else if (day.isSame(now, 'year')) {
      return day.format('M月D日');
    } else {
      return day.format('YYYY年M月D日');
    }
  }

  get recordList() {
    // eslint-disable-next-line no-undef
    return (this.$store.state as RootState).recordList;
  }

  get groupedList() {
    const {recordList} = this;
    if (recordList.length === 0) {return [];}
    const newList = clone(recordList)
        .filter(r => r.type === this.type)
        .sort((a, b) => dayjs(b.createdAt).valueOf() - dayjs(a.createdAt).valueOf());
    if (newList.length === 0) {return [];}

    type Result = { title: string, total?: number, items: RecordItem[] }[]
    const result: Result = [{title: dayjs(newList[0].createdAt).format('YYYY-MM-DD'), items: [newList[0]]}];

    for (let i = 1; i < newList.length; i++) {
      const current = newList[i];
      const last = result[result.length - 1];
      if (dayjs(last.title).isSame(dayjs(current.createdAt), 'day')) {
        last.items.push(current);
      } else {
        result.push({title: dayjs(current.createdAt).format('YYYY-MM-DD'), items: [current]});
      }
    }
    result.map(group => {
      group.total = group.items.reduce((sum, item) => {
        return sum + item.amount;
      }, 0);
    });
    return result;
  }

  beforeCreate() {
    this.$store.commit('fetchRecords');
  }

  type = '-';
  recordTypeList = recordTypeList;
}
</script>

<style scoped lang="scss">
.noResult {
  padding: 16px;
  text-align: center;
}

::v-deep {
  .type-tabs-item {
    background: #C4C4C4;

    &.selected {
      background: white;
      border-bottom: 3px solid #46ad86;

      &::after {
        display: none;
      }
    }
  }

  .interval-tabs-item {
    height: 48px;
  }
}

%item {
  padding: 8px 16px;
  line-height: 24px;
  display: flex;
  justify-content: space-between;
  align-content: center;
}

.title {
  @extend %item;
}

.record {
  background: white;
  @extend %item;
}

.notes {
  margin-right: auto;
  margin-left: 16px;
  color: #999;
}

.chart {
  //希望一个屏幕展示7天数据，那30天需要 4+（2/7）个屏幕，约430%
  width: 430%;
  //横向滚动
  &-wrapper {
    overflow: auto;
    //google: css hide scrollbar
    //隐藏滚动条，PC端需要方向键滚动
    &::-webkit-scrollbar {
      display: none;
    }
  }
}
</style>