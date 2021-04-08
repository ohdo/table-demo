<template>
  <div id="app">
    <a-table
      style="width: 60vw"
      rowKey="uid"
      :columns="columns"
      :dataSource="dataSource"
      :pagination="false"
    />
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
const mapByKey = (list, key) => list.reduce((target, item) => Object.assign(target, { [item[key]]: item }), {});
const dragGroupMap = {
  measure: "度量",
};
export default {
  name: "App",
  data() {
    return {
      source,
    };
  },
  computed: {
    /**
     * 列
     */
    series() {
      const { series = [] } = this.source.option;
      return series;
    },
    /**
     * 列 - key映射表
     */
    seriesMap() {
      return mapByKey(this.series, 'key');
    },
    /**
     * 行
     */
    xAxis() {
      const { xAxis = [] } = this.source.option;
      return xAxis;
    },
    /**
     * 无 行
     */
    noXAxis() {
      return this.xAxis.length === 0;
    },
    /**
     * 数据来源
     */
    rows() {
      return this.source.rows || [];
    },
    /**
     * 列的key提取
     */
    rowKey() {
      return this.series.map((e) => e.key);
    },
    /**
     * table - 列
     */
    columns() {
      let list;
      const rows = this.proxyList;
      const { rowKey, seriesMap, xAxis, noXAxis, series } = this;
      // @method 获取数组 属性为key 的去重列表
      const getListByKey = (arr, key) => {
        return [...new Set(arr.map((e) => e[key]))];
      };
      /**
       * @method 获取children
       * isFirstColumn：是否为第一列
       * islaster：是否为最后一条数据（PS：最后一条数据需要添加dataIndex）
       * filterData：创建children需要使用的列表数据
       * key：当前children对应的key
       */
      const getChlidren = (isFirstColumn, islaster, filterData, key) => {
        if (isFirstColumn && !noXAxis) {
          const res = { title: seriesMap[key].title, key: getUid() };
          if (islaster) {
            const { dragGroup } = xAxis[0];
            res.title = dragGroupMap[dragGroup];
            res.dataIndex = dragGroup;
          }
          return [res];
        } else {
          // 过滤列表中含有的则为下级菜单的数据
          return getListByKey(filterData, key).map((title) => {
            const res = { title, key: getUid() };
            if (islaster) {
              const obj = filterData.find((e) => e[key] === title);
              res.dataIndex = obj.uid;
            }
            return res;
          });
        }
      }
      // @methods 初始化一级表头
      const initBase = () => {
        if (!list) {
          const key = rowKey[0];
          const [ firstColumn ] = getChlidren(true, series.length <= 1, rows, key);
          const keyList = getChlidren(false, series.length <= 1, rows, key);
          list = noXAxis ? keyList : [ firstColumn, ...keyList ];
        }
      };
      // @methods 根据一级表头设置children
      const setChildren = (arr, keys, source) => {
        const keyList = [...keys];
        const lastKey = keyList.shift();
        const key = keyList[0];
        if (!key) return;
        arr.forEach((e, index) => {
          const isFirstColumn = index === 0;
          const islaster = keyList.length === 1;
          const filterData = source.filter((ele) => e.title === ele[lastKey]);
          e.children = getChlidren(isFirstColumn, islaster, filterData, key);
          setChildren(e.children, keyList, filterData);
        });
      };
      initBase();
      setChildren(list, rowKey, rows);
      return list;
    },
    /**
     * 数据来源 - 代理 增加uid供映射转换使用
     */
    proxyList() {
      return setUid(this.source.rows);
    },
    /**
     * table 列表基础数据组装
     */
    list() {
      const { proxyList } = this;
      return this.xAxis.map(({dragGroup, ...e}) =>
        proxyList.reduce(
          (target, item) => Object.assign(target, { [item.uid]: item[e.key] }),
          { [dragGroup]: e.title }
        )
      );
    },
    /**
     * table 列表基础数据 增加uid
     */
    dataSource() {
      return setUid(this.list);
    }
  }
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
