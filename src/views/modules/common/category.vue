<template>
  <el-tree :data="menuData"
           :props="defaultProps"
           node-key="catId"
           ref="treeManu"
           @node-click="nodeClick"
  >

  </el-tree>
</template>
<script>
  export default {
    componts: {},
    data() {
      return {

        menuData: [],
        defaultProps: {
          children: 'childrens',
          label: 'name'
        },
      };
    },
    methods: {
      /*
      *
      * */
      nodeClick(data, node, commonData) {
        console.log("子组件被点击", data, node, commonData);
        //向父组件发送事件 事件名自定义
        this.$emit("tree-node-click", data, node, commonData);



      },
      /**
       * 获取树形数据
       */
      getMenuData() {
        this.$http({
          url: this.$http.adornUrl('/product/category/list/tree'),
          method: 'get',
        }).then(({data}) => {
          this.menuData = data.data;
          //console.log("返回数据", this.menuData)
        });
      },
    },
    created() {
      this.getMenuData();
    }
  }
</script>
