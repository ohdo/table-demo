<template>
  <div id="app">
    <!-- <a-table style="width: 60vw" :columns="[...columns, tableColums]" :dataSource="dataSource" :pagination="false">
      <template #hospital="val, record, index">
        <span @click="zuanqu(record)">{{ val }}</span>
      </template>
      <template #table="val, record, index">
        <a-table :columns="columns" :dataSource="dataSource" :pagination="false"/>
      </template>
    </a-table> -->
    <a-button @click="isRow = !isRow"> change </a-button>
    <a-table
      style="width: 60vw"
      rowKey="uid"
      :columns="columns"
      :dataSource="dataSource"
      :pagination="false"
    >
      <template #hospital="val, record, index">
        <span @click="zuanqu(record)">{{ val }}</span>
      </template>
    </a-table>
  </div>
</template>

<script>
import source from "./data2.json";
var uid = 0;
const getUid = () => ++uid;
const setUid = (list = []) => {
  list.forEach((e) => {
    !e.uid && (e.uid = `key-${getUid()}`);
  });
  return list;
};

const dragGroupMap = {
  measure: "度量",
};
export default {
  name: "App",
  data() {
    return {
      source,
      tableColums: { title: "table", scopedSlots: { customRender: "table" } },
      isRow: false,
    };
  },
  computed: {
    noSeries() {
      const { series = [] } = this.source.option;
      return !series.length;
    },
    columnsRow() {
      const { series, xAxis } = this.source.option;
      const columns1 = xAxis.map((e) => {
        const { title, key } = e;
        return {
          title,
          key,
          dataIndex: key,
          scopedSlots: { customRender: key },
        };
      });
      const columns2 = series.map((e) => {
        const { key, title } = e;
        return {
          title,
          key,
          dataIndex: key,
          scopedSlots: { customRender: key },
        };
      });
      const columns = [...columns1, ...columns2];
      return columns;
    },
    series() {
      const { series = [] } = this.source.option;
      return series;
    },
    rows() {
      return this.source.rows || [];
    },
    rowKey() {
      return this.series.map((e) => e.key);
    },
    columnsCol() {
      let list;
      const rows = this.proxyList;
      const { rowKey } = this;
      const getListByKey = (arr, key) => {
        return [...new Set(arr.map((e) => e[key]))];
      };
      const initBase = () => {
        if (!list) {
          const key = rowKey[0];
          list = getListByKey(rows, key).map((title) => ({
            title,
            key: getUid(),
            dataIndex: key,
          }));
        }
      };
      const setChildren = (arr, keys, source) => {
        const keyList = [...keys];
        const lastKey = keyList.shift();
        const key = keyList[0];
        if (!key) return;
        arr.forEach((e) => {
          const islaster = keyList.length === 1;
          const filterData = source.filter((ele) => e.title === ele[lastKey]);
          // 过滤列表中含有的则为下级菜单的数据
          e.children = getListByKey(filterData, key).map((title) => {
            const res = { title, key: getUid() };
            if (islaster) {
              const obj = filterData.find((e) => e[key] === title);
              res.dataIndex = obj.uid;
            }
            return res;
          });
          setChildren(e.children, keyList, filterData);
        });
      };
      initBase();
      setChildren(list, rowKey, rows);
      return list;
      // const columns = [];
      // const { option: { xAxis } } = this.source;
      // const firstColumn = xAxis[0];
      // if (!this.noSeries) {
      //   const key = firstColumn.key;
      //   columns.push({
      //     title: firstColumn.title,
      //     key,
      //     dataIndex: key,
      //     scopedSlots: { customRender: key },
      //   });
      // }
      // this.proxyList.forEach(e => {
      //   const key = e.uid;
      //   columns.push({
      //     title: e[firstColumn.key],
      //     key,
      //     dataIndex: key,
      //     scopedSlots: { customRender: key },
      //   });
      // });
      // return columns;
    },
    proxyList() {
      return setUid(this.source.rows);
    },
    colList() {
      const { xAxis = [] } = this.source.option;
      const { proxyList } = this;
      return xAxis.map((e) =>
        proxyList.reduce(
          (target, item) => Object.assign(target, { [item.uid]: item[e.key] }),
          {}
        )
      );
      // const { option: { xAxis } } = this.source;
      // const firstColumn = xAxis[0];
      // const { key, title } = firstColumn;
      // const list = [...this.columnsRow];
      // list.shift();
      // const { proxyList } = this;
      // return list.map(e => proxyList.reduce((target, ele) => Object.assign(target, { [ele.uid]: ele[e.key] }), { [key]: e.title }));
    },
    columns() {
      return this.isRow ? this.columnsRow : this.columnsCol;
    },
    dataSource() {
      const list = this.isRow ? this.proxyList : this.colList;
      return setUid(list);
    },
    zuanquMap() {
      if (!this.isRow) return {};
      const { drills = [] } = this.source.option;
      return drills.reduce(
        (target, item) => Object.assign(target, { [item.key]: item }),
        {}
      );
    },
  },
  methods: {
    /**
     * 钻取 TODO:
     * 1. 获取钻取下一级的数据
     */
    zuanqu(record) {
      // TODO: 获取钻取下一级数据
      console.log("钻取, 调用接口");
    },
    getKey(item) {
      return this.rowKey
        .reduce((e, key) => {
          e.push(item[key]);
          return e;
        }, [])
        .join("*");
    },
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
  display: flex;
  align-items: center;
  justify-content: center;
  min-height: 80vh;
}
</style>
