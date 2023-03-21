<template>
  <!-- expand-on-click-node设置false点击append或delete不会自动展开 
  
  -->
  <div>
    <el-tree
      :data="menus"
      :props="defaultProps"
      show-checkbox
      node-key="catId"
      :expand-on-click-node="false"
      :default-expanded-keys="expandedKey"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => append(data)"
          >
            Append
          </el-button>
          <el-button type="text" size="mini" @click="() => edit(node)">
            edit
          </el-button>
          <el-button
            v-if="node.childNodes.length == 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >
            Delete
          </el-button>
        </span>
      </span>
    </el-tree>

    <el-dialog :title="title" :visible.sync="dialogVisible" width="30%">
      <el-form :model="catagory">
        <el-form-item label="分类名称">
          <el-input v-model="catagory.name" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <el-form :model="catagory">
        <el-form-item label="图标">
          <el-input v-model="catagory.icon" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <el-form :model="catagory">
        <el-form-item label="计量单位">
          <el-input v-model="catagory.productUnit" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="sbumitData">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  data() {
    return {
      dialogType:"",//对话框区别，区别是点击append还是edit
      title:"",
      catagory:{
        name:"",
        parentCid:0,
        catLevel:0,
        showStatus:1,
        sort:0,
        productUnit:"",
        icon:"",
        catId:null,
      },
      dialogVisible: false,
      menus: [],
      expandedKey: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },
  methods: {
    handleNodeClick(data) {
      console.log(data);
    },
    getMenus() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        // 大括号解构之后拿到data中的data列表
        console.log("成功获取到data菜单数据:", data);
        this.menus = data.page;
      });
    },
    sbumitData(){
      if(this.dialogType=="add"){
          this.addCatagory();
      }
      if(this.dialogType=="edit"){
        this.editCatagoty();
      }
    },
    editCatagoty(){

    },
    edit(data){
      console.log("要修改的数据：",data);
      this.dialogVisible=true;
      this.dialogType="edit";
      this.title="修改分类";


      //发送请求获取节点罪行的数据
      this.$http({
      url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
      method: 'get',
      }).then(({data}) => {
        console.log("要回显的数据:",data);
        // this.catagory.name=data.name;
        // this.catagory.catId=data.catId;
      })
      



    },
    //添加
    append(data) {
      console.log("append", data);
      this.dialogType="add";
      this.title="添加分类";
      this.dialogVisible = true;
      this.catagory.parentCid=data.catId;
      this.catagory.catLevel=data.catLevel*1+1;
    },
    //添加三级分类
    addCatagory(){
      console.log("append提交的三级分类的数据：",this.catagory);
      this.$http({
      url: this.$http.adornUrl('/product/category/save'),
      method: 'post',
      data: this.$http.adornData(this.catagory, false)
      }).then(({ data }) => {
        this.$message({
              message: "保存成功",
              type: "success",
            });
        //保存成功关闭对话框
        this.dialogVisible=false
        //刷新菜单
        this.getMenus();
        //设置默认展开菜单
        this.expandedKey = [this.catagory.parentCid];
      });
    },

    remove(node, data) {
      var ids = [data.catId];
      this.$confirm(`是否删除当前菜单【${data.name}】, 是否继续?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          }).then(({ data }) => {
            this.$message({
              message: "成功删除",
              type: "success",
            });
            //刷新菜单
            this.getMenus();
            //设置默认展开菜单
            this.expandedKey = [node.parent.data.catId];
          });
        })
        .catch(() => {});

      console.log("remove", node, data);
    },
  },
  created() {
    this.getMenus();
  },
};
</script>


<style >
</style>