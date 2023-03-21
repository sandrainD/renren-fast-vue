<template>
  <!-- expand-on-click-node设置false点击append或delete不会自动展开 
  
  -->
  <div>
    <el-switch v-model="draggable" active-text="开启拖拽" inactive-text="关闭拖拽">
    </el-switch>
    <el-tree
      :data="menus"
      :props="defaultProps"
      show-checkbox
      node-key="catId"
      :expand-on-click-node="false"
      :default-expanded-keys="expandedKey"
      :draggable="draggable"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
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
    <el-dialog
      :title="title"
      :visible.sync="dialogVisible"
      width="30%"
      :close-on-click-modal="false"
    >
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
          <el-input
            v-model="catagory.productUnit"
            autocomplete="off"
          ></el-input>
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
      draggable: false,
      updateNodes: [], //拖拽节点后要修改的节点
      dialogType: "", //对话框区别，区别是点击append还是edit
      maxLevel: 0,
      title: "",
      catagory: {
        catId: null,
        name: "",
        parentCid: 0,
        catLevel: null,
        showStatus: 1,
        sort: 0,
        productUnit: "",
        icon: "",
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

    handleNodeClick(data) {
      console.log(data);
    },
    updateChildNodeLevel(node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data;
          this.updateNodes.push({
            catId: cNode.catId,
            catLevel: node.childNodes[i].level,
          });
          this.updateChildNodeLevel(node.childNodes[i]);
        }
      }
    },
    handleDrop(draggingNode, dropNode, dropType, ev) {
      console.log("handleDrop: ", draggingNode, dropNode, dropType);
      //1、当前节点最新的父节点id
      let pCid = 0;
      let siblings = null;
      if (dropType == "before" || dropType == "after") {
        pCid =
          dropNode.parent.data.catId == undefined
            ? 0
            : dropNode.parent.data.catId;
        siblings = dropNode.parent.childNodes;
      } else {
        pCid = dropNode.data.catId;
        siblings = dropNode.childNodes;
      }

      //2、当前拖拽节点的最新顺序，
      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId == draggingNode.data.catId) {
          //如果遍历的是当前正在拖拽的节点
          let catLevel = draggingNode.level;
          if (siblings[i].level != draggingNode.level) {
            //当前节点的层级发生变化
            catLevel = siblings[i].level;
            //修改他子节点的层级
            this.updateChildNodeLevel(siblings[i]);
          }
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: pCid,
            catLevel: catLevel,
          });
        } else {
          this.updateNodes.push({ catId: siblings[i].data.catId, sort: i });
        }
      }

      //3、当前拖拽节点的最新层级
      console.log("updateNodes", this.updateNodes);
      this.$http({
        url: this.$http.adornUrl("/product/category/update/sort"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单顺序等修该成功",
          type: "success",
        });
        //刷新菜单
        this.getMenus();
        //设置默认展开菜单
        this.expandedKey = [this.catagory.parentCid];
      });
    },
    //draggingNode当前正在拖动的节点,dropNode目标节点，type表示节点被拖拽到了目标的哪个位置
    allowDrop(draggingNode, dropNode, type) {
      //1被拖动的当前节点以及所在的父节点总层数不能大于3
      //1)、被拖动的当前节点总层数
      console.log(draggingNode, dropNode, type);
      this.countNodeLevel(draggingNode.data); //统计当前节点的总层数
      let deep = this.maxLevel - draggingNode.data.catLevel + 1;
      console.log("深度：", deep);
      if (type == "inner") {
        return deep + dropNode.level <= 3;
      } else {
        return deep + dropNode.parent.level <= 3;
      }
    },
    countNodeLevel(node) {
      if (node.children != null && node.children.length > 0) {
        for (let i = 0; i < node.children.length; i++) {
          if (node.children[i].catLevel > this.maxLevel) {
            this.maxLevel = node.children[i].catLevel;
          }
          this.countNodeLevel(node.children[i]);
        }
      }
    },
    sbumitData() {
      if (this.dialogType == "add") {
        this.addCatagory();
      }
      if (this.dialogType == "edit") {
        this.editCatagoty();
      }
    },
    editCatagoty() {
      var { catId, name, icon, productUnit } = this.catagory;
      var data = this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData(
          { catId: catId, name: name, icon: icon, productUnit: productUnit },
          // this.catagory,
          false
        ),
      }).then(({ data }) => {
        this.$message({
          message: "菜单修该成功",
          type: "success",
        });
        //保存成功关闭对话框
        this.dialogVisible = false;
        //刷新菜单
        this.getMenus();
        //设置默认展开菜单
        this.expandedKey = [this.catagory.parentCid];
      });
    },
    edit(data) {
      console.log("要修改的数据：", data);
      this.dialogVisible = true;
      this.dialogType = "edit";
      this.title = "修改分类";
      //发送请求获取节点罪行的数据
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.data.catId}`),
        method: "get",
      }).then(({ data }) => {
        console.log("要回显的数据:", data);
        this.catagory.name = data.data.name;
        this.catagory.catId = data.data.catId;
        this.catagory.icon = data.data.icon;
        this.catagory.productUnit = data.data.productUnit;
        this.catagory.parentCid = data.data.parentCid;
        this.catagory.catLevel = data.data.catLevel;
        this.catagory.showStatus = data.data.showStatus;
        this.catagory.sort = data.data.sort;
      });
    },
    //添加
    append(data) {
      console.log("append", data);
      this.dialogType = "add";
      this.title = "添加分类";
      this.dialogVisible = true;
      this.catagory.parentCid = data.catId;
      this.catagory.catLevel = data.catLevel * 1 + 1;
      this.catagory.name = "";
      this.catagory.catId = null;
      this.catagory.icon = "";
      this.catagory.productUnit = "";
      this.catagory.showStatus = 1;
      this.catagory.sort = 0;
    },
    //添加三级分类
    addCatagory() {
      console.log("append提交的三级分类的数据：", this.catagory);
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.catagory, false),
      }).then(({ data }) => {
        this.$message({
          message: "保存成功",
          type: "success",
        });
        //保存成功关闭对话框
        this.dialogVisible = false;
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