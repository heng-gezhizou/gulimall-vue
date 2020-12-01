<!--  -->
<template>
  <div>
    <el-switch
      v-model="isDraggable"
      active-text="开启拖拽"
      inactive-text="关闭拖拽"
    >
    </el-switch>
    <el-button type="success" plain @click="saveBatch">批量保存</el-button>
    <el-button type="danger" plain @click="deleteBatch">批量删除</el-button>
    <el-tree
      :data="menu"
      :props="defaultProps"
      :expand-on-click-node="false"
      :default-expanded-keys="expanded"
      :draggable="isDraggable"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
      node-key="catId"
      show-checkbox
      ref="menuTree"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            type="text"
            size="mini"
            v-if="node.level <= 2"
            @click="() => append(data)"
          >
            Append
          </el-button>
          <el-button type="text" size="mini" @click="() => edit(data)">
            Edit
          </el-button>
          <el-button
            type="text"
            size="mini"
            v-if="node.childNodes.length == 0"
            @click="() => remove(node, data)"
          >
            Delete
          </el-button>
        </span>
      </span>
    </el-tree>

    <el-dialog
      title="分类操作"
      :visible.sync="dialogFormVisible"
      :close-on-click-modal="false"
    >
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="分类图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input
            v-model="category.productUnit"
            autocomplete="off"
          ></el-input>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="() => cancel()">取 消</el-button>
        <el-button
          v-if="this.category.catId == ''"
          type="primary"
          @click="() => addCategory()"
          >确 定</el-button
        >
        <el-button
          v-if="this.category.catId != ''"
          type="primary"
          @click="() => editCategory()"
          >修 改</el-button
        >
      </div>
    </el-dialog>
  </div>
</template>

<script>
//这里可以导入其他文件（比如：组件，工具js，第三方插件js，json文件，图片文件等等）
//例如：import 《组件名称》 from '《组件路径》';

export default {
  //import引入的组件需要注入到对象中才能使用
  components: {},
  data() {
    return {
      pCid: [],
      isDraggable: false,
      updateNodes: [],
      maxLevel: 0,
      category: {
        catId: "",
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        icon: "",
        productUnit: "",
      },
      dialogFormVisible: false,
      menu: [],
      expanded: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },

  methods: {
    deleteBatch() {
      let nodes = this.$refs.menuTree.getCheckedNodes();
      console.log("nodes:", nodes);
      let delNodes = [];
      let delNodesName = [];
      nodes.forEach((node) => {
        delNodes.push(node.catId);
        delNodesName.push(node.name);
      });
      this.$confirm(`确定批量删除【${delNodesName}】等菜单吗?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(delNodes, false),
          }).then(() => {
            this.$message({
              message: "批量删除菜单成功",
              type: "success",
            });
            this.getMenu();
          });
        })
        .catch(() => {});
    },
    saveBatch() {
      this.$http({
        url: this.$http.adornUrl("/product/category/save/batch"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单顺序等修改成功",
          type: "success",
        });
        this.updateNodes = [];
        this.expanded = this.pCid;
        this.maxLevel = 0;
        this.getMenu();
      });
    },
    handleDrop(draggingNode, dropNode, dropType, ev) {
      console.log("tree drop: ", draggingNode, dropNode, dropType);
      let pCid = 0;
      let siblings = null;
      //获取拖拽结束后当前node的父节点
      if (dropType == "before" || dropType == "after") {
        pCid =
          dropNode.data.parentCid == undefined ? 0 : dropNode.data.parentCid;
        siblings = dropNode.parent.childNodes;
      } else {
        pCid = dropNode.data.catId;
        siblings = dropNode.childNodes;
      }
      //获取拖拽结束后需要修改的排序(sort)信息
      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId == draggingNode.data.catId) {
          let catLevel = draggingNode.level;
          if (siblings[i].level != draggingNode.level) {
            catLevel = siblings[i].level;
            this.updateChildLevel(siblings[i]);
          }
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: pCid,
            catLevel: catLevel,
          });
          this.pCid.push(pCid);
        } else {
          this.updateNodes.push({ catId: siblings[i].data.catId, sort: i });
        }
      }
      console.log("updateNodes", this.updateNodes);
    },
    //获取拖拽结束后要修改的层级(level)数据
    updateChildLevel(node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data;
          this.updateNodes.push({
            catId: cNode.catId,
            catLevel: node.childNodes[i].level,
          });
          this.updateChildLevel(node.childNodes[i]);
        }
      }
    },
    countNodeLevel(node) {
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level;
          }
          this.countNodeLevel(node.childNodes[i]);
        }
      } else {
        this.maxLevel = node.level;
      }
    },
    allowDrop(draggingNode, dropNode, type) {
      console.log("draggingNode", draggingNode, dropNode.data);
      this.countNodeLevel(draggingNode);
      console.log(this.maxLevel, draggingNode.data.catLevel, type);
      let deep = Math.abs(this.maxLevel - draggingNode.level) + 1;
      console.log("深度", this.maxLevel, draggingNode.data.catLevel, deep);
      if (type == "inner") {
        return deep + dropNode.level <= 3;
        this.maxLevel = 0;
      } else {
        return deep + dropNode.parent.level <= 3;
        this.maxLevel = 0;
      }
    },
    cancel() {
      this.dialogFormVisible = false;
      this.category.name = "";
      this.category.catId = "";
      this.category.icon = "";
      this.category.productUnit = "";
    },
    getMenu() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        console.log("成功获取分类数据", data.data);
        this.menu = data.data;
      });
    },
    edit(data) {
      this.dialogFormVisible = true;
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
        params: this.$http.adornParams({}),
      }).then(({ data }) => {
        console.log(data);
        this.category.name = data.data.name;
        this.category.catId = data.data.catId;
        this.category.icon = data.data.icon;
        this.category.productUnit = data.data.productUnit;
        this.category.parentCid = data.data.parentCid;
      });
    },
    editCategory() {
      var { catId, name, icon, productUnit } = this.category;
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData({ catId, name, icon, productUnit }, false),
      }).then(({ data }) => {
        this.$message({
          message: "修改分类成功",
          type: "success",
        });
        this.category.name = "";
        this.category.catId = "";
        this.category.icon = "";
        this.category.productUnit = "";
        // this.category.parentCid = "";
        this.dialogFormVisible = false;
        this.expanded = [this.category.parentCid];
        this.getMenu();
      });
    },
    append(data) {
      console.log("append", data, this.dialogFormVisible);
      this.dialogFormVisible = true;
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel * 1 + 1;
    },
    addCategory() {
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        this.$message({
          message: "新增分类成功",
          type: "success",
        });
        this.dialogFormVisible = false;
        this.expanded = [this.category.parentCid];
        this.getMenu();
      });
    },

    remove(node, data) {
      console.log("remove", data, node);
      var ids = [data.catId];
      this.$confirm(`确定删除【${data.name}】菜单吗?`, "提示", {
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
              message: "删除成功",
              type: "success",
            });
            this.expanded = [node.data.parentCid];
            this.getMenu();
          });
        })
        .catch(() => {});
    },
  },
  //监听属性 类似于data概念
  computed: {},
  //监控data中的数据变化
  watch: {},
  //生命周期 - 创建完成（可以访问当前this实例）
  created() {
    this.getMenu();
  },
  //生命周期 - 挂载完成（可以访问DOM元素）
  mounted() {},
  beforeCreate() {}, //生命周期 - 创建之前
  beforeMount() {}, //生命周期 - 挂载之前
  beforeUpdate() {}, //生命周期 - 更新之前
  updated() {}, //生命周期 - 更新之后
  beforeDestroy() {}, //生命周期 - 销毁之前
  destroyed() {}, //生命周期 - 销毁完成
  activated() {}, //如果页面有keep-alive缓存功能，这个函数会触发
};
</script>
<style scoped>
</style>