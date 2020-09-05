<template>
  <div>
    <el-switch v-model="draggable" active-text="开启拖拽" inactive-text="关闭拖拽"></el-switch>
    <el-button v-if="draggable" @click="batchSaveNode">批量保存</el-button>
    <el-button type="danger" @click="batchDelete">批量删除</el-button>
    <el-tree :data="menuData"
             :props="defaultProps"
             :expand-on-click-node="false"
             @node-drop="handleDrop"
             :allow-drop="allowDrop"
             :default-expanded-keys="expandedKeys"
             :draggable="draggable"
             show-checkbox
             ref="treeManu"
             node-key="catId">
      <!--操作按钮  node 当前节点  data 数据库中的数据  node.data==data-->
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level<=2"
            type="text"
            size="mini"
            @click="() => append(data)">
            add
          </el-button>
          <el-button
            type="text"
            size="mini"
            @click="editCategory(data)">
            edit
          </el-button>
          <el-button
            type="text"
            size="mini"
            v-if="node.childNodes.length==0"
            @click="() => remove(node, data)">
            delete
          </el-button>
        </span>
      </span>

    </el-tree>
    <!--新增-->

    <el-dialog :title="dialogTitle" :visible.sync="dialogFormVisible">
      <el-form :model="category">
        <el-form-item label="菜单名称">
          <el-input v-model="category.name"></el-input>
        </el-form-item>
        <el-form-item label="排序">
          <el-input v-model="category.sort"></el-input>
        </el-form-item>

        <el-form-item label="图标">
          <el-input v-model="category.icon"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input v-model="category.productUnit"></el-input>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="dialogFormVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitData">确 定</el-button>
      </div>
    </el-dialog>
  </div>
</template>
<script>
  export default {
    componts: {},
    props: {},
    data: function () {
      return {
        parentId: [],
        draggable: false,
        //拖拽后要更新的节点
        updateNodes: [],
        maxLevle: 1,
        dialogTitle: "",
        dialogType: "add",
        dialogFormVisible: false,
        category: {
          name: null,
          parentCid: 0,
          catLevel: 0,
          sort: 0,
          showStatus: 1,
          catId: null
        },
        menuData: [],
        defaultProps: {
          children: 'childrens',
          label: 'name'
        },
        expandedKeys: []
      };
    },
    methods: {

      /*批量删除*/
      batchDelete: function () {
        let deleteIds = [];
        let checkedNodes = this.$refs.treeManu.getCheckedNodes();
        for (let i = 0; i < checkedNodes.length; i++) {
          deleteIds.push(checkedNodes[i].catId);
        }
        this.$confirm(`确定要删除吗?`, '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(deleteIds, false)
          }).then(({data}) => {
            this.$message({
              message: `删除成功！`,
              type: 'success'
            });
            //执行刷新 TODO
            this.getMenuData();
          });
        }).catch(() => {

        });


      },
      batchSaveNode: function () {
        //更新数据
        this.$http({
          url: this.$http.adornUrl("/product/category/updateByIds"),

          method: "post",
          data: this.$http.adornData(this.updateNodes, false)
        }).then(({data}) => {
          this.$message({
            message: "修改层级成功",
            type: 'success'
          });
          this.getMenuData();
          this.expandedKeys = this.parentId;
          this.updateNodes = [];
          this.maxLevle = 0;
        });
      },
      /*拖拽成功后的 函数*/
      handleDrop: function (draggingNode, targetNode, dragType, event) {
        //需要获取当前节点的 最新 父节点ID 最新顺序  最新层级
        //父节点Id
        let parentId = 0;
        //拖拽后  被拖拽节点的兄弟节点 用于排序
        let brotherNodes = null;
        if (dragType === "inner") {
          //1.获取当前节点的 最新 父节点ID
          parentId = targetNode.data.catId;
          //2.targetNode
          brotherNodes = targetNode.childNodes;
        } else {
          //1.获取当前节点的 最新 父节点ID (当处于顶级节点时父Id可能是undefined)
          parentId = targetNode.parent.data.catId == undefined ? 0 : targetNode.parent.data.catId;
          brotherNodes = targetNode.parent.childNodes;

        }
        this.parentId.push(parentId);
        //获取当前节点的最新顺序
        for (let i = 0; i < brotherNodes.length; i++) {
          if (brotherNodes[i].data.catId == draggingNode.data.catId) {
            //遍历的是当前被拖拽的节点
            //被拖拽节点的层级
            let catLevel = draggingNode.level;
            if (brotherNodes[i].level != draggingNode.level) {
              //节点层级发生变化
              catLevel = brotherNodes[i].level;
              //修改字节点的层级

              this.updateChildrenNodes(brotherNodes[i]);
            }
            //不仅要改变顺序还要改变 父Id
            this.updateNodes.push({catId: brotherNodes[i].data.catId, sort: i, parentCid: parentId, catLevel: catLevel})
          } else {

            this.updateNodes.push({catId: brotherNodes[i].data.catId, sort: i})
          }
        }


      },
      updateChildrenNodes: function (node) {

        if (node.childNodes.length > 0) {
          for (let i = 0; i < node.childNodes.length; i++) {
            let currentNode = node.childNodes[i].data;
            this.updateNodes.push({cId: currentNode.cId, catLevel: node.childNodes[i].level})
            this.updateChildrenNodes(node.childNodes[i]);

          }
        }
      },
      /*判断是否可以被拖拽*/
      allowDrop: function (draggingNode, dropNode, type) {
        console.log("正在拖拽的节点", draggingNode, dropNode, type);
        this.countDraggingNodeMaxLevel(draggingNode.data);
        console.log("计算完成后的", this.maxLevle)
        //当前节点的深度
        let deep = this.maxLevle - draggingNode.data.catLevel + 1;
        console.log("deep", deep);
        if (type == "inner") {
          // 往里拖动 当前拖拽的节点深度 + 要拖动的节点的层级 《=3
          console.log("deep + dropNode.levle", deep, dropNode.level)
          return deep + dropNode.level <= 3
        } else {
          // 往前后拖动 当前拖拽的节点深度 + 要拖动的节点父节点的层级 《=3

          return deep + dropNode.parent.level <= 3
        }
      },
      /*计算拖拽节点的最大深度*/
      countDraggingNodeMaxLevel(data) {
        if (data.childrens != null && data.childrens.length > 0) {
          for (let i = 0; i < data.childrens.length; i++) {
            if (data.childrens[i].catLevel > this.maxLevle) {
              console.log("this.maxLevle 修改了", this.maxLevle, "====>", data.childrens[i].catLevel)
              this.maxLevle = data.childrens[i].catLevel;
            }
            this.countDraggingNodeMaxLevel(data.childrens[i]);
          }
        }
      },
      getMenuData() {
        this.$http({
          url: this.$http.adornUrl('/product/category/list/tree'),
          method: 'get',
        }).then(({data}) => {
          this.menuData = data.data;
          //console.log("返回数据", this.menuData)
        });
      },
      handleNodeClick(data) {
        console.log(data);
      },
      //点击修改
      editCategory(data) {
        this.dialogType = "edit";
        this.dialogTitle = "修改分类"
        //获取最新的数据
        this.$http({
          url: this.$http.adornUrl('/product/category/info/' + data.catId),
          method: 'get',
        }).then(({data}) => {
          this.category = data.category;
        });

        this.dialogFormVisible = true;
      },
      //点击添加
      append(data) {
        //修改对话框属性
        this.dialogType = "add";
        this.dialogTitle = "添加分类"
        //打开对话框
        this.dialogFormVisible = true;
        this.category.parentCid = data.catId;
        this.category.catLevel = data.catLevel + 1;
        console.log("append------")
        console.log(" this.category:", this.category);
      },
      submitData() {
        if (this.dialogType == "add") {
          this.add();
        }
        if (this.dialogType == "edit") {
          this.edit();
        }

        this.dialogFormVisible = false;
        //刷新
        this.getMenuData();
        this.expandedKeys = [this.category.parentCid];
        this.category.name = "";
        this.category.sort = null;
        this.category.productUnit = null;
        this.category.icon = null;

      },
      //实际的修改
      edit() {
        console.log("要发送的修改内容", this.category)
        this.$http({
          url: this.$http.adornUrl("/product/category/update"),
          method: "post",
          data: this.$http.adornData(this.category, false)
        }).then(({data}) => {
          this.$message({
            message: `修改【${this.category.name}】成功！`,
            type: 'success'
          });

        });

      },
      //实际的增加
      add() {
        this.$http({
          url: this.$http.adornUrl("/product/category/save"),
          method: "post",
          data: this.$http.adornData(this.category, false)
        }).then(({data}) => {

          this.$message({
            message: `新增成功！`,
            type: 'success'
          });

          /* console.log("新增成功",category)
           console.log(`新增【${this.category.name}】成功！`)*/

        });
      },
      remove(node, data) {
        console.log("remove------")
        let catIds = [data.catId]
        this.$confirm(`确定要删除【${node.label}】吗?`, '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(catIds, false)
          }).then(({data}) => {
            this.$message({
              message: `删除【${node.label}】成功！`,
              type: 'success'
            });
            //刷新
            this.getMenuData();
            this.expandedKeys = [node.parent.data.catId];
          });


        }).catch(() => {

        });
      }

    },
    created() {
      this.getMenuData()
    },

    computed: {},
    watch: {},


  }
</script>
