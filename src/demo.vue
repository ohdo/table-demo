<template>
  <div id="app">
    <a-table
      bordered
      style="width: 60vw"
      rowKey="uid"
      :columns="columns"
      :dataSource="dataSource"
      :pagination="{ pageSize: 10 }"
    />
  </div>
</template>

<script>
import source from "./data6.json";
var uid = 0;
const getUid = () => ++uid;
const setUid = (list = []) => {
  list.forEach((e) => {
    !e.uid && (e.uid = `key-${getUid()}`);
  });
  return list;
};
const mapByKey = (list, key) =>
  list.reduce(
    (target, item) => Object.assign(target, { [item[key]]: item }),
    {}
  );
const dragGroupMap = {
  measure: "度量",
};
const getDragName = (xAxis) => {
  const { displayName, dragGroup } = xAxis;
  return dragGroupMap[dragGroup] || displayName;
};
const getDataIndex = (obj) => obj.columnDataType === "INT" ? obj.dragGroup : obj.key;
const getUnikey = (keys, item) => keys.map((k) => item[k]).join("-");
const isInt = obj => obj.columnDataType === "INT";
export default {
  name: "Demo",
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
    intSeries() {
      const obj = this.proxySeries.find(e => e.columnDataType === "INT");
      return obj.children;
    },
    /**
     * 列 度量合并
     */
    proxySeries() {
      const { series } = this;
      const list = [];
      const map = new Map();
      const children = [];
      series.forEach((e) => {
        if (e.columnDataType === "INT") {
          children.push(e);
          if (!map.has(e.dragGroup)) {
            map.set(e.dragGroup);
            list.push({
              ...e,
              key: e.dragGroup,
              children
            });
          }
        } else {
          list.push(e);
        }
      });
      return list;
    },
    /**
     * 列 - key映射表
     */
    seriesMap() {
      return mapByKey(this.proxySeries, "key");
    },
    xAxisNoInt() {
      return this.xAxis.every(e => e.columnDataType !== "INT");
    },
    /**
     * 行
     */
    xAxis() {
      const { xAxis = [] } = this.source.option;
      return xAxis;
    },
    /**
     * 度量合并
     */
    proxyXAxis() {
      const { xAxis } = this;
      const list = [];
      const map = new Map();
      xAxis.forEach((e) => {
        if (e.columnDataType === "INT") {
          if (!map.has(e.dragGroup)) {
            map.set(e.dragGroup);
            list.push(e);
          }
        } else {
          list.push(e);
        }
      });
      return list;
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
      return this.proxySeries.map((e) => e.key);
    },
    headColumns() {
      const list = [...this.proxyXAxis];
      list.pop();
      return list.map((e) => ({
        key: getUid(),
        title: getDragName(e),
        dataIndex: getDataIndex(e),
      }));
    },
    lastColumn() {
      const column = [...this.proxyXAxis].pop();
      let res, children = [];
      this.proxySeries.forEach((e, index, arr) => {
        const isLast = index + 1 === arr.length;
        const obj = {
          key: getUid(),
          title: getDragName(e),
          dataIndex: getDataIndex(e),
        }
        if (isLast) {
          Object.assign(obj, {
            title: column.title,
            dataIndex: column.key
          })
        }
        if (!res) {
          res = obj;
          res.children = children;
        } else {
          children.push(obj);
          if (!isLast) {
            children = obj.children = [];
          }
        }
      });
      return [res];
    },
    afterColumn() {
      const { seriesMap, seriesKey, getUnikey } = this;
      const lastSeriesKey = [...seriesKey].pop();
      let filterData = [...this.rows];
      // @method 获取数组 属性为key 的去重列表
      const getListByKey = (arr, key) => {
        return [...new Set(arr.map((e) => e[key]))];
      };
      const getChlidren = (e, list, intKey) => {
        const IS_INT = isInt(e);
        const islast = e.key === lastSeriesKey;
        const getCurstomRender = (key, val = undefined) => {
          const tlist = val === undefined ? list : list.filter(e => e[lastSeriesKey] === val);
          return islast ? (text, record, index) => {
            const obj = tlist.find(e => getUnikey(record) === getUnikey(e));
            return <span>{obj ? obj[key] : ''}</span>
          } : undefined;
        };
        if (IS_INT) {
          return e.children.map(e => ({
            key: getUid(),
            intKey: e.key,
            title: e.title,
            customRender: getCurstomRender(e.key)
          }))
        } else {
          const resList = getListByKey(list, e.key);
          return resList.map(title => ({
            title,
            key: getUid(),
            dataIndex: e.key,
            customRender: getCurstomRender(intKey, title)
          }));
        }
      };
      const setChildren = (arr, list, keys, resList = null, intKey) => {
        if (!keys.length) return;
        const curkeys = [...keys];
        const lastKey = curkeys.shift();
        const key = curkeys[0];
        if (!resList) {
          resList = arr = getChlidren(seriesMap[lastKey], list);
        }
        if (!key) return;
        const IS_INT = isInt(seriesMap[lastKey]);
        arr.forEach(e => {
          const filterData = IS_INT ? [...list] : list.filter(ele => e.title === ele[lastKey]);
          if (IS_INT) {
            intKey = e.intKey;
          }
          e.children = getChlidren(seriesMap[key], filterData, intKey);
          setChildren(e.children, filterData, curkeys, resList, intKey);
        });
        return resList;
      };
      return setChildren([], filterData, this.seriesKey);
    },
    /**
     * table - 列
     */
    columns() {
      return [...this.headColumns, ...this.lastColumn, ...this.afterColumn];
    },
    /**
     * 数据来源 - 代理 增加uid供映射转换使用
     */
    proxyList() {
      const { rows = [] } = this.source;
      rows.forEach((e) => {
        e.uuid = this.getuuid(e);
      });
      return setUid(rows);
    },
    xAxisKey() {
      return this.xAxis.map((e) => e.key);
    },
    /**
     * STRING类的行 转化为数据列选项
     */
    xAxisStringList() {
      const { xAxisKey, rows } = this;
      const list = [];
      const map = new Map();
      rows.forEach((e) => {
        const unikey = this.getUnikey(e);
        if (!map.has(unikey)) {
          map.set(unikey);
          list.push(
            xAxisKey.reduce(
              (target, k) => Object.assign(target, { [k]: e[k] }),
              {}
            )
          );
        }
      });
      return list;
    },
    /**
     * INT类的行 转化为数据列选项
     */
    xAxisIntList() {
      const { xAxis } = this;
      const list = [];
      xAxis.forEach((e) => {
        if (e.columnDataType === "INT") {
          list.push({ [e.dragGroup]: e.title, key: e.key });
        }
      });
      return list;
    },
    noXAxisInt() {
      return this.xAxisIntList.length === 0;
    },
    /**
     * 整合INT和STRING列的数据项
     */
    xAxisList() {
      const { xAxisStringList, xAxisIntList, noXAxisInt } = this;
      if (noXAxisInt) return xAxisStringList;
      const list = [];
      xAxisStringList.forEach((item) => {
        list.push(...xAxisIntList.map((e) => ({ ...e, ...item })));
      });
      return list;
    },
    /**
     * table 列表基础数据组装
     */
    list() {
      return this.xAxisList;
      // const { proxyList, xAxisList, unikey } = this;
      // return xAxisList.map((e) => {
      //   return proxyList.reduce(
      //     (target, item) => {
      //       const obj = Object.assign(
      //         target,
      //         this.getUnikey2(e) === this.getUnikey2(item)
      //           ? { [item.uuid]: item[e.key] }
      //           : {}
      //       );
      //       return obj;
      //     },
      //     unikey.reduce(
      //       (target, key) => Object.assign(target, { [key]: e[key] }),
      //       {}
      //     )
      //   );
      // });
    },
    /**
     * table 列表基础数据 增加uid
     */
    dataSource() {
      return setUid(this.list);
    },
    unikey() {
      const keys = this.xAxis.reduce(
        (target, item) => [
          ...target,
          item.columnDataType === "INT" ? item.dragGroup : item.key,
        ],
        []
      );
      return [...new Set(keys)];
    },
    unikey2() {
      return this.xAxis
        .filter((e) => e.columnDataType !== "INT")
        .map((e) => e.key);
    },
    seriesKey() {
      return this.proxySeries.map(e => e.key);
    }
  },
  methods: {
    getUnikey(item) {
      return getUnikey(this.unikey, item);
    },
    /**
     * 数据上同源  去除度量
     */
    getUnikey2(item) {
      return getUnikey(this.unikey, item);
    },
    /**
     * 根据配置的行获取唯一标识
     */
    getuuid(item) {
      return this.proxySeries.filter(e => e.columnDataType !== "INT").map((e) => item[e.key]).join("-");
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
