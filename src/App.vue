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
import source from "./data5.json";
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
const getDragGroupKey = ({ dragGroup }) => dragGroup;
const getDataIndex = (obj) =>
  obj.columnDataType === "INT" ? getDragGroupKey(obj) : obj.key;
const getUnikey = (keys, item) => keys.map((k) => item[k]).join("-");
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
      return mapByKey(this.series, "key");
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
      return this.series.map((e) => e.key);
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
      return [...this.proxyXAxis].pop();
    },
    /**
     * table - 列
     */
    columns() {
      let list;
      const rows = this.proxyList;
      const { rowKey, seriesMap, lastColumn, noXAxis, series } = this;
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
            const obj = lastColumn;
            res.title = getDragName(obj);
            res.dataIndex = getDataIndex(obj);
          }
          return [res];
        } else {
          // 过滤列表中含有的则为下级菜单的数据
          return getListByKey(filterData, key).map((title) => {
            const res = { title, key: getUid() };
            if (islaster) {
              const obj = filterData.find((e) => e[key] === title);
              res.dataIndex = obj.uuid;
            }
            return res;
          });
        }
      };
      // @methods 初始化一级表头
      const initBase = () => {
        if (!list) {
          const key = rowKey[0];
          const [firstColumn] = getChlidren(
            true,
            series.length <= 1,
            rows,
            key
          );
          const keyList = getChlidren(false, series.length <= 1, rows, key);
          list = noXAxis ? keyList : [firstColumn, ...keyList];
        }
      };
      // @methods 根据一级表头设置children
      const setChildren = (arr, keys, source, i = 0) => {
        const keyList = [...keys];
        const lastKey = keyList.shift();
        const key = keyList[0];
        if (!key) return;
        arr.forEach((e, index) => {
          const isFirstColumn = i + index;
          const islaster = keyList.length === 1;
          const filterData = source.filter((ele) => e.title === ele[lastKey]);
          e.children = getChlidren(
            isFirstColumn === 0,
            islaster,
            filterData,
            key
          );
          setChildren(e.children, keyList, filterData, isFirstColumn);
        });
      };
      initBase();
      setChildren(list, rowKey, rows);
      return [...this.headColumns, ...list];
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
      const { proxyList, xAxisList, unikey } = this;
      return xAxisList.map((e) => {
        return proxyList.reduce(
          (target, item) => {
            const obj = Object.assign(
              target,
              this.getUnikey2(e) === this.getUnikey2(item)
                ? { [item.uuid]: item[e.key] }
                : {}
            );
            return obj;
          },
          unikey.reduce(
            (target, key) => Object.assign(target, { [key]: e[key] }),
            {}
          )
        );
      });
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
  },
  methods: {
    getUnikey(item) {
      return getUnikey(this.unikey, item);
    },
    /**
     * 数据上同源  去除度量
     */
    getUnikey2(item) {
      return getUnikey(this.unikey2, item);
    },
    /**
     * 根据配置的行获取唯一标识
     */
    getuuid(item) {
      return this.series.map((e) => item[e.key]).join("-");
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
