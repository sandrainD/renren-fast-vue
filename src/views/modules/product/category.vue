<template>
<el-tree :data="data" :props="defaultProps" @node-click="handleNodeClick"></el-tree>
</template>

<script>
export default {
  data() {
    return {
      data: [],
      defaultProps: {
        children: "children",
        label: "label",
      },
    };
  },
  methods: {
    handleNodeClick(data) {
      console.log(data);
    },
    getMenus(){
      this.$http({
          url: this.$http.adornUrl('/product/category/list/tree'),
          method: 'get',
        }).then(({data}) => {
          // 大括号解构之后拿到data中的data列表
          console.log("success获取菜单数据",data.data)
        })
    }
  },
  created(){
    this.getMenus();
  }
};

</script>


<style >
</style>